# Security baseline, abuse prevention, privacy, and test plan

## Contents
1. Django and DRF settings baseline
2. Shop-specific threats and controls
3. Payments and PCI scope
4. Audit log immutability (MySQL)
5. Files, GCS, and PDFs
6. Privacy and retention
7. Logging, monitoring, and supply chain
8. Test matrix
9. CI gate and deployment checks
10. Security red flags for reviews

## 1. Django and DRF settings baseline

```python
import os
from django.utils.csp import CSP

DEBUG = False
SECRET_KEY = os.environ["DJANGO_SECRET_KEY"]
ALLOWED_HOSTS = ["api.example.com"]                       # never ["*"]
CSRF_TRUSTED_ORIGINS = ["https://admin.example.com"]

SECURE_SSL_REDIRECT = True
SECURE_HSTS_SECONDS = 31536000                            # ramp up from a small value first
SECURE_HSTS_INCLUDE_SUBDOMAINS = True
SECURE_PROXY_SSL_HEADER = ("HTTP_X_FORWARDED_PROTO", "https")   # only if the proxy sets and strips it
SESSION_COOKIE_SECURE = CSRF_COOKIE_SECURE = True
SESSION_COOKIE_HTTPONLY = True
SESSION_COOKIE_SAMESITE = "Lax"
SECURE_REFERRER_POLICY = "same-origin"
X_FRAME_OPTIONS = "DENY"
DATA_UPLOAD_MAX_MEMORY_SIZE = 5 * 1024 * 1024
DATA_UPLOAD_MAX_NUMBER_FIELDS = 1000

PASSWORD_HASHERS = ["django.contrib.auth.hashers.Argon2PasswordHasher",       # pip install "django[argon2]"
                    "django.contrib.auth.hashers.PBKDF2PasswordHasher"]
# Keep all AUTH_PASSWORD_VALIDATORS enabled.

MIDDLEWARE = ["django.middleware.security.SecurityMiddleware", "...",
              "django.middleware.csp.ContentSecurityPolicyMiddleware"]       # for Django Admin and any HTML pages
SECURE_CSP = {"default-src": [CSP.SELF], "script-src": [CSP.SELF, CSP.NONCE],
              "object-src": [CSP.NONE], "base-uri": [CSP.NONE], "frame-ancestors": [CSP.NONE]}

REST_FRAMEWORK = {
    "DEFAULT_AUTHENTICATION_CLASSES": ["rest_framework_simplejwt.authentication.JWTAuthentication"],
    "DEFAULT_PERMISSION_CLASSES": ["rest_framework.permissions.IsAuthenticated"],   # DRF default is AllowAny; opt out explicitly
    "DEFAULT_RENDERER_CLASSES": ["rest_framework.renderers.JSONRenderer"],
    "DEFAULT_PARSER_CLASSES": ["rest_framework.parsers.JSONParser"],               # add multipart only on upload routes
    "DEFAULT_THROTTLE_CLASSES": ["rest_framework.throttling.AnonRateThrottle",
                                 "rest_framework.throttling.UserRateThrottle",
                                 "rest_framework.throttling.ScopedRateThrottle"],
    "DEFAULT_PAGINATION_CLASS": "config.api.pagination.CappedPagination",
    "EXCEPTION_HANDLER": "config.api.exceptions.handler",
    "NUM_PROXIES": 1,
}
SIMPLE_JWT = {"ACCESS_TOKEN_LIFETIME": timedelta(minutes=10), "REFRESH_TOKEN_LIFETIME": timedelta(days=7),
              "ROTATE_REFRESH_TOKENS": True, "BLACKLIST_AFTER_ROTATION": True, "ALGORITHM": "HS256",
              "SIGNING_KEY": os.environ["JWT_SIGNING_KEY"]}                        # separate from SECRET_KEY; INSTALLED_APPS needs token_blacklist
```

Run `manage.py check --deploy` in CI. Use Redis for cache, throttling, and Channels (never local memory with multiple workers). Public routes opt out of the default permission explicitly with `AllowAny` and a comment.

Serializer rules: explicit `fields` lists only (never `__all__` or `exclude`), `read_only_fields` for server-controlled values (`status`, `owner`, `role`, `is_staff`, prices on order items), `write_only` for secrets, separate public and staff serializers, and related-field querysets scoped to the caller.

## 2. Shop-specific threats and controls

| Threat | Control |
|---|---|
| Price or discount tampering | Server recomputes everything; order item prices are read-only; coupon and promotion rules evaluated server-side. |
| IDOR on orders, receipts, cases, addresses, warranties | Queryset scoping by owner or token; 404 for foreign objects; tests per resource. |
| Privilege escalation through role, `is_staff`, or grant endpoints | Role and flag fields read-only for non-admins; `grant_permission()` service with allowlist; superuser-only role promotion; audit. |
| Mass assignment on registration and profile updates | Explicit serializer fields; `role`, `is_superuser`, `is_phone_verified` never writable by the user. |
| Stock hoarding (cart or checkout abuse to block inventory) | Reservations only at checkout with short expiry; per-IP and per-user checkout throttle; bot protection; quantity limits. |
| Payment bypass or replay | Order paid only after verified capture with matching amount and currency; idempotent capture; signature-verified webhooks; unique `event_id`. |
| Double refund or over-refund | Row lock on payment, pending-refund accounting, persisted idempotency key, thresholds, dual approval. |
| Coupon brute force and abuse | Throttle validation, uniform errors, high-entropy codes for targeted campaigns, per-user limits, atomic redemption under lock. |
| OTP brute force and SMS pumping | Hashed OTP, attempt limits, resend cooldown, per-target/IP/day caps, country caps, bot protection, spend alerts. |
| Guest link guessing or leakage | 256-bit hashed tokens, purpose and order binding, expiry, revocation, uniform 404, no-referrer, log scrubbing. |
| Account enumeration | Uniform responses for login, reset, register, verification, link requests; timing-safe comparisons. |
| Card testing through checkout | PayPal handles card entry; still throttle order creation, require bot protection for guests, watch failed-capture rates. |
| XSS through reviews, chat, notes, product text | Store plain text; sanitize any rich text with `nh3` allowlist; escape in templates and PDFs; CSP on Admin. |
| Spreadsheet formula injection in exports | Prefix cells beginning with `=`, `+`, `-`, `@` (and tab or CR) with a single quote or escape them in CSV and Excel exports. |
| SSRF via courier or webhook URLs, image fetches | Allowlists, no user-supplied URLs fetched server-side, block private ranges, timeouts. |
| Staff data abuse (bulk customer lookup) | Grants, search throttles, audit of PII views and searches, masked fields by default. |
| Webhook spoofing (PayPal, courier, SMS provider) | Verify signatures, store raw event, reject stale timestamps, allow only expected event types. |
| Inventory or serial tampering | Append-only ledger, serial state machine, no direct field edits in Admin, nightly invariants check. |
| Notification template injection | Substitution of allowlisted placeholders only; no full template engine for admin-edited text. |

## 3. Payments and PCI scope

- Card data never touches the server: PayPal's approval flow keeps the integration in the lightest PCI scope (self-assessment for hosted or redirect flows). Do not add card fields to any model, form, log, or analytics event.
- Store only provider IDs, status, amounts, currency, and a masked payer email if needed. Keep raw provider payloads in a restricted table, redacted, with a retention limit.
- Credentials: PayPal client ID and secret, webhook ID, and signing keys in a secret manager; rotate on exposure; separate sandbox and live; fail startup if live credentials are used with `DEBUG`.
- Use TLS 1.2+ everywhere, pin to official PayPal hosts in the client (no user-controlled base URLs), set timeouts, and retry only idempotent calls.
- Reconcile daily: sum of captures and refunds in PayPal reports vs local payments; flag differences.

## 4. Audit log immutability (MySQL)

Application level: `AuditLog.save()` raises on update, `delete()` raises, the manager's queryset blocks `update()` and `delete()`, and Django Admin is read-only (no add, change, or delete).

Database level, in a migration with `RunSQL`:

```sql
CREATE TRIGGER audit_block_update BEFORE UPDATE ON audit_auditlog
FOR EACH ROW SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'audit rows are immutable';
CREATE TRIGGER audit_block_delete BEFORE DELETE ON audit_auditlog
FOR EACH ROW SIGNAL SQLSTATE '45000' SET MESSAGE_TEXT = 'audit rows are immutable';
```

Also grant the application database user only `INSERT` and `SELECT` on the audit table, and use a separate, rarely used role for approved retention purges. Optional tamper evidence: store `prev_hash` and `row_hash` (SHA-256 over the row plus the previous hash) and verify the chain on a schedule.

Write audit rows inside the same transaction as the change (via the service layer) so a committed change always has its record and a rolled-back one leaves none. Required fields: action ID, action, actor, timestamp, related order when applicable; add customer, product, payment, refund, before/after (PII redacted), IP, request ID.

## 5. Files, GCS, and PDFs

- Buckets: private (receipts, profile photos, case attachments, exports) with uniform bucket-level access and public access prevention; separate public bucket or CDN path for product images. Object versioning and retention policy on receipts per business requirement.
- Service account with least privilege (object creator and viewer on specific buckets), workload identity instead of key files. Signed URLs (V4) expire in minutes, are generated per request after an authorization check, and are never stored.
- Uploads: validate size, extension, MIME by content (Pillow for images), re-encode images to strip metadata, random object names, ignore client filenames, scan attachments where risk warrants it. Treat SVG as active content (reject or sanitize). Serve user-uploaded files with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`.
- Direct-to-GCS uploads use signed upload URLs with content-type and size limits, followed by server-side verification before the object is linked to a record.
- PDF rendering never fetches remote or local resources from user-controlled input.

## 6. Privacy and retention

- Minimize: guests provide only what delivery, payment, and notification need. Keep addresses and phone numbers in snapshots on orders for legal and support use.
- Consent: `ConsentRecord` rows for each marketing channel (who, when, source, text shown, IP), with withdrawal events. Honor withdrawal immediately.
- Retention schedule (configurable, documented): chat logs, OTP records (days), `InteractionEvent` (90-180 days), search logs, guest carts, expired tokens, notifications bodies, receipts (legal period). Implement as Celery beat jobs with audit entries.
- Erasure: anonymize profile, addresses, contacts, and chat content while retaining orders, payments, refunds, and audit rows with PII replaced by stable pseudonyms where the law requires financial records to be kept.
- Staff views of PII are permissioned, minimized, and logged.

## 7. Logging, monitoring, and supply chain

- Structured JSON logs with request IDs. Never log passwords, tokens (JWT, guest, OTP), full payloads from PayPal, full addresses, or chat bodies. Use `@sensitive_variables` and `@sensitive_post_parameters`, and error reporting (for example Sentry) with `send_default_pii=False` and scrubbers for token paths.
- Alerts: payment mismatches, webhook signature failures, failed captures spike, refund volume anomalies, permission grants and role changes, OTP spend, repeated guest-token misses, stock invariant violations.
- Pin dependencies with hashes (`uv lock` or `pip-compile --generate-hashes`), run `pip-audit` in CI, use Dependabot or Renovate, apply Django and DRF security releases promptly, and wait a short period before adopting brand-new third-party releases. Review new packages for maintenance and necessity.
- Containers: multi-stage builds, slim image, non-root user, no secrets in layers. MySQL runs with TLS, a least-privilege runtime role (no `DROP`, no `GRANT`) separate from the migration role, and tested backups with point-in-time recovery.

## 8. Test matrix

Write tests with each feature slice (pytest-django, factory_boy, DRF `APIClient`).

- **Permissions**: anonymous, each role, and each ownership case for every route (see `roles-permissions.md` section 8). Assert 401, 403, and 404 deliberately.
- **Money and ETA**: price recomputation ignores client input; discount allocation sums exactly; rounding; ETA with items at 10, 30, and 15 days equals the 30-day date; empty cart rejected.
- **Stock concurrency**: two threads reserving the last unit (use `TransactionTestCase`); expiry releases; invariant job passes; deadlock retry works.
- **Payments**: mocked PayPal client for create, capture completed, pending, denied, amount mismatch, duplicate capture, webhook replay, forged webhook rejected, reconciliation of stuck payments.
- **Refunds**: full, partial, line-item; cannot exceed refundable; concurrent refund requests; idempotent retry after timeout; threshold approval; customer-initiated flow; audit rows created.
- **Guest tokens**: wrong purpose, expired, revoked, another order, brute-force throttling, uniform 404, masked response fields.
- **OTP**: expiry, attempts lock, cooldown, single use, resend limits, no enumeration.
- **Reviews**: all three eligibility conditions, one per product, rating bounds (DB constraint too), guest cannot rate.
- **Packing and serials**: duplicate, invalid, wrong variant, already sold, missing serial blocks packing, state transitions, history rows.
- **Notifications**: consent suppression, dedupe key, rollback sends nothing, template placeholder allowlist.
- **Audit**: every state-changing service writes a row; update and delete attempts fail at app and DB level.
- **Query counts**: `django_assert_num_queries` on list endpoints (products, orders, cases) and `FETCH_RAISE` in hot-path tests.
- **Security regressions**: mass-assignment attempts (`role`, `is_staff`, `status`, `price`), CSV formula injection, oversize uploads, SVG upload, open redirect parameters.

## 9. CI gate and deployment checks

```bash
ruff check . && ruff format --check .
python -Wd manage.py makemigrations --check --dry-run
pytest -q --cov
python manage.py check --deploy --fail-level WARNING --settings=config.settings.production
pip-audit
```

Then: container build and scan, migration gating with a rollback plan, health and readiness probes, Celery worker and beat, Channels workers, and a smoke test of checkout in the PayPal sandbox before release.

## 10. Security red flags for reviews

`fields = "__all__"`; `queryset = Model.objects.all()` on non-public viewsets; missing `permission_classes` or `AllowAny` on sensitive routes; trusting client `price`, `total`, `status`, or `role`; order status or payment status set outside services; PayPal or SMTP calls inside `transaction.atomic()`; `select_for_update` without ordering; capture without amount and currency verification; webhooks without signature checks; guest tokens stored raw or compared by `==` on raw values; OTP stored in plaintext; unthrottled OTP, coupon, or login endpoints; `mark_safe` or `|safe` on user text; `csrf_exempt` outside webhooks; secrets in the repo; JWTs in `localStorage`; public GCS buckets for receipts; `DB_CASCADE` on financial tables; audit rows editable in Admin; templates rendered with the full Django engine from admin-edited text.
