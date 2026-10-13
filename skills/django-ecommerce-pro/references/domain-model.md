# Domain model, state machines, and MySQL rules

## Contents
1. Conventions
2. Apps and models
3. State machines
4. MySQL-specific rules
5. Settings fragments (database, storage)

## 1. Conventions

- Every table has a `BigAutoField` primary key. Models exposed to clients also get `public_id` (UUID, unique, indexed), because MySQL stores UUID primary keys as wide strings that bloat every secondary index. Prefer `UUID7` generated in Python or by the database function (Django 6.1) for index locality.
- Human-facing order numbers are random and non-sequential (for example `DC-8F3K-29QX`, unique), so they cannot be enumerated. The order number alone never authorizes access.
- Money: `DecimalField(max_digits=12, decimal_places=2)` plus a `currency` code. Keep a single `quantize` helper and one rounding mode (`ROUND_HALF_UP`). Currencies without minor units need `decimal_places=0` handling at the PayPal boundary.
- `created_at` and `updated_at` on everything. Soft-delete (`is_active`, `archived_at`) for catalog data. Never delete orders, payments, refunds, audit rows, or stock movements.
- `on_delete=PROTECT` for any foreign key into orders, payments, refunds, stock movements, and audit. `CASCADE` only for true children (cart items, product images).
- Choices as `TextChoices` classes. State fields are changed only by service functions.

## 2. Apps and models

Fields shown are the ones that carry rules; add the usual descriptive fields.

**accounts**
- `User` (custom, `AbstractUser`): `email` (unique, normalized lowercase), `phone` (E.164), `role` (CUSTOMER, ASSISTANT, STOCK, PACKING, ADMIN), `is_email_verified`, `is_phone_verified`, `profile_photo` (private storage), `last_password_change_at`.
- `Address`: user, kind (BILLING, SHIPPING), recipient, line fields, city, region, postal code, country, phone, `is_default`.
- `ContactVerification`: user or pending contact, channel (EMAIL, SMS), `target`, `code_hmac`, `expires_at`, `attempts`, `consumed_at`, `sent_count_window`.
- `NotificationPreference`: user, one boolean per category (order email, order SMS, receipt email, receipt SMS, promo email, promo SMS, recommended-sale), `updated_at`. `ConsentRecord`: user or contact, category, granted/withdrawn, source, IP, timestamp (append-only).
- `GuestContact`: normalized email and phone captured at guest checkout (links guest orders for later conversion).

**access** (see `roles-permissions.md`): `GranularPermission`.

**audit**: `AuditLog` (append-only; see section 4 of `security-and-testing.md` for DB triggers).
- `action_id` (UUID), `action` (dotted string such as `order.status_changed`), `actor` (nullable FK), `actor_type` (USER, GUEST, SYSTEM, WEBHOOK), `created_at`, `order` (nullable FK, PROTECT), optional `customer`, `product`, `payment`, `refund`, `before` and `after` (JSON, PII redacted), `ip_address`, `request_id`, `context` (JSON).

**catalog**
- `Brand`: name, slug, `is_active`, logo, `is_featured`.
- `Category`: parent (self FK), name, slug, `path` (materialized path or use `django-treebeard`), `is_active`, sort order.
- `Product`: name, slug, descriptions, `brand`, `sku` (unique), `status`, `availability_mode` (IMMEDIATE, PREORDER, TEMPORARILY_UNAVAILABLE, OUT_OF_STOCK), `estimated_arrival_days` (required, >= 0), `price`, `sale_price`, `sale_starts_at`, `sale_ends_at`, `currency`, `cost_price` (staff-only), `unit_of_measure`, `expiry_date` (nullable), `weight`, dimensions, `track_serials`, `warranty_policy` (FK), search metadata, `rating_avg`, `rating_count` (denormalized, updated in the review service).
- `ProductCategory` (through): product, category, `is_primary`, sort. `UniqueConstraint(product, category)`.
- `ProductVariant`: product, `sku` (unique), attribute values (JSON of option ids), `price`, `sale_price`, `estimated_arrival_days` (nullable override), `track_serials` (nullable override), `is_active`.
- `SpecKey` (name, `is_filterable`) and `ProductSpecification` (product, key, value). `AttributeOption` for variant options (color, size).
- `ProductImage`: product or variant, image (GCS), alt text, sort order.
- `Tag`, `RecommendationTag` (admin-only), `RelatedProduct` (from, to, kind: RELATED, FREQUENTLY_BOUGHT_TOGETHER, UPSELL).

**inventory**
- `StockItem`: one per variant (or product without variants), `on_hand`, `reserved`, `damaged`, `lost`, `low_stock_threshold`. `CheckConstraint(on_hand >= 0, reserved >= 0, reserved <= on_hand)`. `available = on_hand - reserved` is computed, never stored.
- `StockMovement` (append-only ledger): stock item, `kind` (ADDITION, REMOVAL, DAMAGE, LOSS, ADJUSTMENT, RESERVE, RELEASE, SALE, RETURN, PREORDER_ALLOCATE), signed `quantity`, `reason`, `reference_type` and `reference_id` (order, return, purchase order), `actor`, `before_on_hand`, `after_on_hand`, `created_at`. Invariant: `on_hand` equals the sum of on-hand-affecting movements; a nightly job asserts it.
- `StockLot` (optional, perishables): stock item, lot code, `quantity`, `expiry_date`. Allocate first-expired-first-out.
- `SerialNumber`: product or variant, `serial` (stored normalized), `state`, `order_item` (nullable), `lot`. `UniqueConstraint(serial, variant)` (or global uniqueness if serials are globally unique). `SerialEvent` (append-only): serial, from state, to state, actor, order, note.
- `Reservation`: order, stock item, quantity, `expires_at`, `status` (ACTIVE, COMMITTED, RELEASED, EXPIRED).

**promotions**
- `Coupon` (admin-only): `code` (stored uppercase, unique), kind (PERCENT, FIXED, FREE_SHIPPING), value, `min_order_amount`, `max_discount`, window, `usage_limit_total`, `usage_limit_per_user`, `is_active`, scope (products, categories, brands), `stackable`, `priority`. `CouponRedemption`: coupon, order, customer or guest contact, amount. `UniqueConstraint(coupon, order)`.
- `Promotion` (campaign): scope, rule, window, priority, stacking policy.
- `TaxRate` (region, category, rate, inclusive flag). `ShippingZone`, `ShippingMethod`, `ShippingRate` (flat, weight-based, free-over threshold).

**cart**: `Cart` (user or `guest_token_hash`, `expires_at`), `CartItem` (cart, variant, quantity, `added_price_snapshot` used only to warn about price changes), `Wishlist`, `WishlistItem` (price at add time), `StockSubscription` (back in stock).

**orders**
- `Order`: `number`, `public_id`, user (nullable) or `guest_contact`, `status`, `payment_status`, currency, `subtotal`, `discount_total`, `shipping_total`, `tax_total`, `grand_total`, `estimated_arrival_date`, shipping and billing address snapshots (JSON), `contact_email`, `contact_phone`, `is_preorder`, `placed_at`, `idempotency_key` (unique), `terms_accepted_at`, `customer_note`.
- `OrderItem`: order, variant (PROTECT), name/SKU snapshots, `quantity`, `unit_price`, `discount_allocated`, `tax_amount`, `line_total`, `estimated_arrival_date`, `is_preorder`, `refunded_quantity`, `refunded_amount`.
- `OrderStatusTransition`: from status, to status, `allowed_roles`, `requires_permission`, `is_active` (admin-editable workflow; the status codes themselves are fixed enums because code depends on them).
- `OrderEvent`: order, `kind`, description (customer-readable), `visible_to_customer`, actor, `created_at`. This is the customer timeline; `AuditLog` is the staff-grade record.
- `GuestAccessToken`: order, `purpose` (TRACK, RECEIPT, WARRANTY, CALL_REQUEST), `token_hash` (unique), `expires_at`, `revoked_at`, `last_used_at`, `use_count`.

**payments**
- `Payment`: order, provider (`paypal`), `provider_order_id`, `provider_capture_id`, `status` (CREATED, APPROVED, CAPTURE_PENDING, CAPTURED, FAILED, CANCELLED, PARTIALLY_REFUNDED, REFUNDED), `amount`, `currency`, `captured_at`, `refunded_total`, raw provider payload (redacted, restricted). `UniqueConstraint(provider, provider_order_id)`.
- `WebhookEvent`: provider, `event_id` (unique), type, payload, `processed_at`, `status`.
- `Refund`: payment, order, `status` (REQUESTED, APPROVED, PROCESSING, COMPLETED, FAILED, REJECTED), `kind` (FULL, PARTIAL, LINE_ITEMS), `amount`, `reason`, `requested_by`, `approved_by`, `idempotency_key` (UUID, persisted before the PayPal call), `provider_refund_id`, `processed_at`. `RefundItem`: refund, order item, quantity, amount.

**fulfilment**: `Courier`, `Package` (order, status, courier, `tracking_number`, packed_by, packed_at, handed_over_at), `PackingIssue` (order item, kind: MISSING_SERIAL, INVALID_SERIAL, DUPLICATE_SERIAL, DAMAGED, SHORT_STOCK, OTHER, status, notes), `ShipmentEvent` (package, status, location, `occurred_at`, source: MANUAL, COURIER_WEBHOOK, POLL).

**documents**: `Receipt` (order, `number` unique, `file` in private storage, `sha256`, `generated_at`, `version`).

**notifications**: `NotificationTemplate` (event, channel, language, subject, body with allowlisted placeholders, `is_active`), `Notification` (recipient contact, channel, category, template, rendered body, order, status: QUEUED, SENT, DELIVERED, FAILED, SUPPRESSED, `provider_message_id`, attempts, `dedupe_key` unique), `CommunicationLog` (a read model combining notifications, receipt sends, chat, call requests).

**care**: `CareCase` (customer or guest contact, order, `status`, `priority` NORMAL or URGENT, `assigned_to`, `resolved_at`), `CaseMessage` (case, sender type, `channel` CHAT, WHATSAPP, EMAIL, NOTE, `is_internal`, body, attachments), `CallRequest` (order, customer or guest contact, preferred time, status: REQUESTED, SCHEDULED, COMPLETED, CANCELLED, handled_by), `Escalation` (case or order, reason, customer, created_by, status, resolution, resolved_by), `ChatConversation` and `ChatMessage` (live chat, retained per policy).

**warranty**: `WarrantyPolicy` (duration months, start rule: PURCHASE, DELIVERY, REGISTRATION; end rule; eligibility), `Warranty` (order item, serial, customer or guest contact, start, end, status: ACTIVE, EXPIRED, VOID, CLAIMED), `WarrantyClaim` (warranty, reason, status, handled_by).

**returns**: `ReturnRequest` (order, `rma_number`, status: REQUESTED, APPROVED, REJECTED, IN_TRANSIT, RECEIVED, INSPECTED, REFUNDED, CLOSED, reason, tracking), `ReturnItem` (order item, quantity, condition, outcome: RESTOCK, DAMAGED, SCRAP), `ReturnRule` (window days, eligible categories, conditions).

**reviews**: `Review` (user, product, `rating` 1-5, title, body, status: PUBLISHED, HIDDEN, PENDING, `edited_at`). `UniqueConstraint(user, product)`, `CheckConstraint(rating between 1 and 5)`. `ReviewReport`.

**recommendations**: `InteractionEvent` (user or anonymous id, kind: VIEW, SEARCH, WISHLIST_ADD, CART_ADD, PURCHASE, product, category, `created_at`, partitioned or pruned by retention), `CustomerInterest` (user, category/tag/brand, score, updated_at), `TrendingScore` (product, score, window, computed_at), `TrendingConfig` (signal weights, window, decay).

**core**: `SiteConfig` (key/value with audit), `IdempotencyRecord` (key, scope, `request_hash`, response, `expires_at`), `FeatureFlag`.

## 3. State machines

Define each as a dict of allowed transitions in one module, enforced by one `transition()` service that locks the row, validates the edge and actor, applies side effects, writes `OrderEvent` plus `AuditLog`, and queues notifications.

**Order** (statuses are those in feature ORD-02). Main path:
`PENDING -> PAYMENT_PROCESSING -> PAID -> CONFIRMED -> PROCESSING -> PICKING -> PACKING -> PACKED -> READY_FOR_DISPATCH -> HANDED_TO_DELIVERY_COMPANY -> IN_TRANSIT -> OUT_FOR_DELIVERY -> DELIVERED -> COMPLETED`.
Branches: `PAID -> AWAITING_STOCK` or `PREORDER` (when items are not in stock), `OUT_FOR_DELIVERY -> DELIVERY_FAILED -> OUT_FOR_DELIVERY` (retry) or `-> RETURNED`, any pre-`PACKED` status `-> CANCELLED`, `DELIVERED -> REFUND_REQUESTED -> PARTIALLY_REFUNDED or REFUNDED`, `DELIVERED -> RETURNED`.
Rules: only the payment service moves into `PAID`; only the packing service moves `PACKING -> PACKED` (it requires all serials valid); only refund processing moves into `PARTIALLY_REFUNDED` and `REFUNDED`; customers can trigger only cancellation (before packing), refund request, and return request.

**Payment**: `CREATED -> APPROVED -> CAPTURED`, `CREATED/APPROVED -> CAPTURE_PENDING -> CAPTURED or FAILED`, `CREATED -> CANCELLED`, `CAPTURED -> PARTIALLY_REFUNDED -> REFUNDED`.

**Serial number**: `IN_STOCK -> RESERVED -> PACKED -> SOLD -> DELIVERED -> RETURNED -> IN_STOCK or DAMAGED`, `IN_STOCK -> DAMAGED`, `DELIVERED -> WARRANTY_REPLACED` (or a dedicated warranty state). Any other edge is rejected and audited.

**Refund**: `REQUESTED -> APPROVED -> PROCESSING -> COMPLETED or FAILED`, `REQUESTED -> REJECTED`, `FAILED -> PROCESSING` (retry with the same idempotency key).

**Return**: see the status list above; `RECEIVED -> INSPECTED` decides restock or damaged and may trigger a refund.

## 4. MySQL-specific rules

- **utf8mb4 everywhere**, collation `utf8mb4_0900_ai_ci` (case- and accent-insensitive). Uniqueness on email, SKU, and coupon code is therefore case-insensitive; still normalize (lowercase email, uppercase codes) before saving so behavior is explicit.
- **No conditional unique constraints.** Django ignores `UniqueConstraint(condition=...)` on MySQL, and unique indexes allow multiple NULLs. Model "global scope" with a sentinel (`object_id = 0`) instead of NULL, or enforce in a service under a lock. Do not rely on `nulls_distinct`.
- **CHECK constraints are enforced** (MySQL 8.0.16+), so use `CheckConstraint` for non-negative stock, rating range, `reserved <= on_hand`, and refund totals.
- **Isolation and deadlocks.** InnoDB defaults to REPEATABLE READ with gap locks. Lock rows in a consistent order (sort by primary key before `select_for_update`), keep transactions short, never call PayPal, SMTP, or SMS inside a transaction, and retry on MySQL error 1213 (deadlock) or 1205 (lock wait timeout) with a small bounded retry decorator.
- **Atomic counters.** Use `F()` expressions with a guarded `UPDATE` and check the affected row count:
  ```python
  rows = StockItem.objects.filter(pk=item.pk, on_hand__gte=F("reserved") + qty).update(reserved=F("reserved") + qty)
  if rows == 0:
      raise OutOfStock(item)
  ```
- **No full-text search helpers in Django for MySQL.** Use a `FULLTEXT` index created in a migration (`RunSQL`) with `MATCH ... AGAINST` through a small custom lookup for names and descriptions, and `startswith` on indexed SKU and brand columns. Move to a search engine (OpenSearch, Meilisearch, or Typesense) when the catalog or traffic grows, behind the same `SearchService` interface.
- **Strict mode and TLS**: `sql_mode` includes `STRICT_TRANS_TABLES`; the connection requires TLS in production.
- **Large tables** (`InteractionEvent`, `AuditLog`, `StockMovement`, `Notification`): index on the common filter columns and `created_at`, paginate by keyset (cursor) rather than offset, and plan partitioning or archival from the start.
- **Migrations**: add nullable column, backfill in batches, then add the constraint. MySQL 8 supports `ALGORITHM=INSTANT` for many `ADD COLUMN` operations; test migrations against production-sized data.
- **JSONField** is fine for snapshots and attributes. Do not query across JSON for hot filters; promote those to columns.

## 5. Settings fragments

```python
DATABASES = {
    "default": {
        "ENGINE": "django.db.backends.mysql",
        "NAME": env("DB_NAME"), "USER": env("DB_USER"), "PASSWORD": env("DB_PASSWORD"),
        "HOST": env("DB_HOST"), "PORT": "3306",
        "CONN_MAX_AGE": 60,
        "OPTIONS": {
            "charset": "utf8mb4",
            "init_command": "SET sql_mode='STRICT_TRANS_TABLES'",
            "ssl": {"ca": env("DB_SSL_CA")},
        },
    }
}

# Two private/public storages (option names per the django-storages docs; verify at build time)
STORAGES = {
    "default": {"BACKEND": "storages.backends.gcloud.GoogleCloudStorage",
                "OPTIONS": {"bucket_name": env("GCS_PRIVATE_BUCKET"), "querystring_auth": True, "expiration": 600}},
    "public_media": {"BACKEND": "storages.backends.gcloud.GoogleCloudStorage",
                     "OPTIONS": {"bucket_name": env("GCS_PUBLIC_BUCKET"), "querystring_auth": False}},
    "staticfiles": {"BACKEND": "whitenoise.storage.CompressedManifestStaticFilesStorage"},
}
```

Use workload identity or an attached service account with least-privilege bucket roles instead of key files. Use a callable `storage=` on `FileField`/`ImageField` to pick `public_media` for product images and the default private storage for receipts and profile photos.
