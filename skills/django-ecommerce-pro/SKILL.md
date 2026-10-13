---
name: "django-ecommerce-pro"
description: Build, extend, review, and secure complete e-commerce backends with Django 6.x and Django REST Framework (DRF 3.18+), including catalog, variants, search, cart, guest and registered checkout, PayPal Checkout, orders and tracking, stock and serial numbers, packing and dispatch, receipts (PDF, email, SMS), notifications, warranty, customer care (chat, WhatsApp, call requests, escalations), refunds and returns, promotions and coupons, recommendations, audit logging, reports, and a granular role-based permission engine. Defaults to MySQL and Google Cloud Storage. Use this skill whenever the user mentions an online store, shop, e-commerce or marketplace API, checkout, cart, orders, PayPal, inventory, warehouse packing, receipts, refunds, DevCommerce, or FLXCORE, even if they only ask for one module such as stock or refunds, so the full feature baseline, permission rules, and payment and security invariants stay consistent across the system.
metadata:
  model: opus
---

# Django E-Commerce Pro: complete, secure shop APIs on Django 6.1 and DRF 3.18

Use this skill to design and build e-commerce APIs that cover the full feature baseline below, not just the module the user asked about. A shop is a set of tightly coupled invariants (money, stock, payment state, access control, audit trail), and a feature built in isolation usually breaks one of them. Always build within the shared architecture, permission engine, and security rules described here and in `references/`.

The baseline merges three requirement documents the user supplied (the DevCommerce functional specification, FEATURES, and USER_ROLES) with standard e-commerce capabilities that those documents imply or omit. Every feature has an ID, a source tag, and a priority in `references/feature-catalog.md`.

## Stack baseline (verified October 2026; re-verify before pinning)

| Layer | Default | Notes |
|---|---|---|
| Django | 6.1.x (6.1.2 on Oct 6, 2026) | Python 3.12-3.14 (default 3.13). Use the newest patch: patches carry security fixes. Choose 5.2 LTS only if Python or MySQL constraints force it. |
| DRF | 3.18.x | Supports Django 5.2, 6.0, 6.1. List-serializer (`many=True`) errors are dict-shaped since 3.18. `@throttle_scope` exists for function views. |
| Database | MySQL 8.4+ (InnoDB, utf8mb4) | Django 6.1 dropped MySQL below 8.4. Use `mysqlclient`. MariaDB 10.11+ also works. See `references/domain-model.md` for MySQL-specific traps. |
| File storage | Google Cloud Storage via `django-storages` | Private bucket for receipts and personal files, separate public-read bucket or CDN path for product images. |
| Payments | PayPal Checkout, Orders API v2 | Server-side create and capture, webhook verification, `PayPal-Request-Id` idempotency. |
| Async | Celery + Redis (workers and beat) | Notifications, PDF receipts, webhook processing, reconciliation, trending scores, reservation expiry. Django's built-in Tasks framework has no scheduler, so keep Celery for periodic and retried work. |
| Realtime | Django Channels + Redis | Live chat for customers, guests, and care assistants. |
| Auth | `djangorestframework-simplejwt` (JWT) for the API, Django sessions for Django Admin | Short-lived access tokens, rotating and blacklisted refresh tokens. |
| Other | `django-filter`, `drf-spectacular`, `phonenumbers`, ReportLab or WeasyPrint, `nh3` | Verify current versions on PyPI at build time. |

If the user states a different stack (PostgreSQL, S3, Stripe), keep every rule in this skill and swap only the adapters.

## Build workflow (follow in order)

1. **Ground the request.** Identify what exists (empty repo, partial FLXCORE-style codebase, review only), the stack in use, and which catalog items are in scope. If the user asks for one module, still read the catalog entries that touch it (for example refunds touch payments, audit, stock, serial numbers, notifications, receipts).
2. **Create the coverage matrix.** Copy the IDs from `references/feature-catalog.md` into the project (for example `docs/feature-coverage.md`) with a status per feature: `done`, `partial`, `deferred (reason)`, or `out of scope (user decision)`. MUST items are never dropped silently. Deferring one needs a stated reason.
3. **Design before coding.** Read `references/domain-model.md` (apps, models, state machines, MySQL constraints) and `references/roles-permissions.md`. Resolve the requirement conflicts listed below, or ask the user when a conflict changes behavior they would notice.
4. **Implement in vertical slices, in this order**, because each layer depends on the previous one: core (settings, audit, idempotency) and accounts, then permissions, catalog, inventory, pricing and promotions, cart, checkout and payments, orders, fulfilment (packing, delivery), notifications and receipts, care/warranty/returns/refunds, discovery (search, recommendations, trending), reports, hardening.
5. **Write tests with each slice**: happy path, permission matrix, concurrency (stock, refunds, coupons), and idempotency (see `references/security-and-testing.md`).
6. **Verify** and report: run the commands below, then give the user the coverage matrix summary, the conflicts you resolved, and anything deferred.

Verification commands:
```bash
python -Wd manage.py test            # or: pytest -q
python manage.py makemigrations --check --dry-run
python manage.py check --deploy --fail-level WARNING --settings=config.settings.production
pip-audit && ruff check --select S
```

## Architecture rules

- **Apps** (one cohesive domain each): `core`, `accounts`, `access` (permission engine), `audit`, `catalog`, `inventory`, `promotions`, `cart`, `orders`, `payments`, `fulfilment`, `documents`, `notifications`, `care`, `warranty`, `returns`, `reviews`, `recommendations`, `reports`. Details in `references/domain-model.md`.
- **Layering**: views and serializers handle transport and validation only. Business rules live in `services.py` functions that own `transaction.atomic()` blocks, state transitions, stock changes, and audit writes. This keeps Django Admin, DRF, and Celery tasks on the same code path so none of them can bypass a rule.
- **Two serializer families per resource**: public/customer and staff. Cost price, margin, supplier data, internal notes, and stock thresholds exist only in staff serializers.
- **Snapshots**: order items store product name, SKU, unit price, discount allocation, tax, and ETA at purchase time. Historical orders, receipts, and refunds never read live catalog prices.
- **State machines**: orders, payments, refunds, returns, serial numbers, shipments, cases, and call requests each have an explicit transition table enforced in one service function.
- **Idempotency**: checkout, payment capture, refunds, receipt resend, and webhook handlers accept or derive an idempotency key and are safe to retry.
- **Outbox for side effects**: write `Notification` rows inside the business transaction, enqueue delivery with `transaction.on_commit()`, and let the worker retry. A rolled-back order must never send a message.
- **Time**: `USE_TZ = True`, UTC in the database, store-timezone dates for arrival estimates.

## Non-negotiable invariants

Each rule exists because the failure it prevents is expensive (lost money, oversold stock, leaked orders). Details and code are in the references.

1. **Server-side money.** The server recomputes prices, discounts, shipping, tax, and totals at checkout from the database. Never trust client-sent prices. Use `Decimal` with explicit rounding, never floats.
2. **Order state follows the actual payment result.** An order becomes `PAID` only after the server verifies PayPal capture status `COMPLETED` with matching amount and currency, confirmed idempotently (capture response and webhook may both arrive).
3. **No oversold stock.** Reserve with a conditional `UPDATE` (or `select_for_update` in id order) inside a transaction, and release reservations on expiry, cancellation, or payment failure.
4. **Refunds never exceed the refundable amount.** Lock the payment row, count pending refunds, create the refund row first, and call PayPal after the commit with a persisted idempotency key.
5. **Guest links are capabilities.** Random 256-bit tokens, only the hash stored, purpose-scoped (track, receipt, warranty, call request), order-bound, expiring, revocable, and rate-limited. Invalid tokens always return the same 404.
6. **Authorization is default-deny and queryset-scoped.** Customers see only their own rows (hardcoded, no granular grants). Staff roles need explicit grants. Object-level grants filter list queries, not just detail views.
7. **The permission engine cannot be used to escalate.** Allowlist delegable models, forbid self-grants, bound grants by what the granter holds, and cascade revocation (see `references/roles-permissions.md`).
8. **Audit everything important, immutably.** Audit rows are written in the same transaction as the change, are append-only at the database level, and carry action ID, action, actor, timestamp, and related order (plus optional customer, product, payment, refund, before/after).
9. **Reviews need purchase plus verified phone plus login**, one per customer per product, rating 1-5 enforced by validators and a `CheckConstraint`.
10. **Estimated arrival**: the order's estimated arrival date is the latest item arrival date (the slowest product wins). Compute server-side, snapshot on the order, and show it in cart, checkout, confirmation, order details, receipt, email, SMS, and tracking.
11. **Consent gates marketing.** Promotional email/SMS and recommended-sale notifications send only with an explicit opt-in on file. Transactional messages follow separate rules.
12. **Secrets and payment data**: never store card data (PayPal handles it), keep PayPal credentials in a secret manager, verify every webhook signature, and keep receipts and personal files in private storage behind short-lived signed URLs or API streaming.
13. **Abuse controls on cost and enumeration endpoints**: OTP send/verify, coupon validation, login, password reset, guest checkout, receipt resend, search autocomplete.

## Requirement conflicts and how this skill resolves them

The three source documents disagree in a few places. Apply these defaults and tell the user which ones you applied, so they can override them.

| Conflict | Default applied |
|---|---|
| USER_ROLES gives `STOCK` "full CRUD on Products, Categories, Brands" but also lists Product, Category, Brand, Coupon, RecommendationTag as strictly admin-only and non-delegable | Admin-only wins (least privilege). `STOCK` works on stock items, movements, lots, and serial numbers, and sees a read-only product summary inside inventory endpoints. The admin-only set is one constant, `ADMIN_ONLY_MODELS`, so it is easy to change. |
| FEATURES: marking an order packed sets it to `IN_TRANSIT` automatically. The spec has Packed, Ready for Dispatch, Handed to Delivery Company, then In Transit | Follow the spec: `mark_as_packed` sets `PACKED`; a separate handover action sets `HANDED_TO_DELIVERY_COMPANY`; courier events or a manual update set `IN_TRANSIT`. A setting `ORDER_AUTO_IN_TRANSIT_ON_PACKED` (default `False`) restores the FEATURES behavior. |
| Spec: administrators use Django Admin and different admins have different areas. USER_ROLES: `ADMIN` has global API access | Implement both. Django Admin access follows Django groups and model permissions (ship a "Full Admin" group by default so the documented behavior holds, and allow narrower groups). The API `ADMIN` role has global access to the admin-only endpoints. |
| FEATURES: customers "initiate refund requests" that staff approve. Spec: care assistants refund directly | Support both: customer-initiated refunds start as `REQUESTED` and need approval. Staff-initiated refunds start as `APPROVED`. Add a configurable amount threshold above which an administrator must approve. |
| Spec reviews: login + verified phone + purchase. USER_ROLES mentions only `is_phone_verified` | Enforce all three. |

## Reference map (read what the task needs)

| File | Read when |
|---|---|
| `references/feature-catalog.md` | Always at the start: IDs, sources, priorities, coverage matrix seed. |
| `references/domain-model.md` | Designing models, migrations, state machines, MySQL constraints, stock and money fields. |
| `references/roles-permissions.md` | Anything involving roles, `GranularPermission`, delegation, staff endpoints, Django Admin access. |
| `references/commerce-flows.md` | Cart, checkout, PayPal, orders, stock reservation, packing, delivery, refunds, returns, receipts, notifications, guest links, OTP, warranty, recommendations. |
| `references/api-surface.md` | Laying out endpoints, throttle scopes, permission mapping per route, and OpenAPI. |
| `references/security-and-testing.md` | Settings baseline, DRF hardening, abuse prevention, privacy, file handling, logging, test matrix, CI. |

## Django 6.x features worth using in a shop

- **Fetch modes** (6.1): `Product.objects.fetch_mode(models.FETCH_PEERS)` collapses N+1 patterns on list endpoints, and `FETCH_RAISE` in tests catches accidental per-row queries. Still use `select_related` and `prefetch_related` for known paths.
- **`MAILERS`** (6.1) replaces `EMAIL_BACKEND`/`EMAIL_*`. Use separate mailers for transactional and marketing mail. The old settings are deprecated.
- **Built-in CSP** (6.0+) for Django Admin and any server-rendered pages, using `ContentSecurityPolicyMiddleware` and `SECURE_CSP`.
- **Avoid `DB_CASCADE`/`DB_SET_NULL`** on financial, audit, and order tables: they skip signals and make deletion silently destroy history. Use `PROTECT` there.
- **Deprecations to avoid**: `select_related()` with no arguments, `values_list(flat=True)` without a field, `EMAIL_*` settings, `salted_hmac()` without an explicit `algorithm="sha256"`.
- Run tests with deprecation warnings visible so `RemovedInDjango2028Warning` surfaces early.

## Response format

Lead with what you built or changed, then the coverage status for the touched feature IDs, then conflicts resolved and deferred items. Provide runnable code with tests. For schema or settings changes, include the migration and rollout note (especially stock, payment, and permission tables). When the request touches authentication, payments, files, or permissions, end with a short "Security notes" list. Keep small requests small: one endpoint does not need the full checklist, but it still follows the invariants above.

## Example requests

- "Scaffold the whole e-commerce API from these requirement documents" (full matrix, build order above)
- "Implement guest checkout with PayPal and secure tracking, receipt, and warranty links"
- "Add partial refunds by line item for customer care assistants" (refund service, permissions, audit, PayPal refund, notifications)
- "Build the packing workflow with serial number validation"
- "Review my FLXCORE permissions app for privilege escalation"
- "Add trending products and personalized recommendations with consent-aware sale notifications"
