# API surface, permissions per route, and throttle scopes

Keep the paths already documented by the user (`/api/users/permissions/`, `/api/users/superusers/`, the `mark_as_packed` action). Version the API through the `Accept` header (`AcceptHeaderVersioning`) so documented paths stay stable. Generate the schema with `drf-spectacular` and keep it behind authentication for staff-only routes.

Legend for the Access column: **P** public, **G** guest token, **C** authenticated customer (own data), **A** `ADMIN`/superuser, **S:<role>** staff role with granular permission on the named model, **W** webhook (signature verified).

## Contents
1. Accounts and users
2. Catalog and discovery
3. Cart, checkout, payments
4. Orders and tracking
5. Fulfilment and inventory
6. Care, refunds, returns, warranty
7. Notifications, reports, admin
8. Throttle scopes
9. Conventions

## 1. Accounts and users

| Route | Methods | Access | Notes |
|---|---|---|---|
| `/api/auth/register/` | POST | P | Throttle `register`; bot protection; no account-existence leak. |
| `/api/auth/login/`, `/refresh/`, `/logout/` | POST | P / C | JWT; logout blacklists refresh. Throttle `login`. |
| `/api/auth/password/forgot/`, `/reset/`, `/change/` | POST | P / P / C | Uniform responses; token expiry; invalidate refresh tokens on change. |
| `/api/auth/verify/send/`, `/verify/confirm/` | POST | C | OTP (6 digits), channels email or SMS. Throttle `otp`. |
| `/api/me/` | GET, PATCH | C | Profile, photo upload via signed URL or multipart with validation. |
| `/api/me/addresses/` | CRUD | C | Default flags; validate and normalize. |
| `/api/me/preferences/` | GET, PATCH | C | Notification and consent preferences (append `ConsentRecord`). |
| `/api/me/export/`, `/api/me/delete/` | POST | C | Data export and deletion request (async). |
| `/api/users/` | CRUD | A | Staff and customer administration; cannot create or delete superusers unless superuser. |
| `/api/users/superusers/` | GET, PATCH | superuser | Promote and demote roles, core user data. |
| `/api/users/permissions/` | GET, POST, PATCH, DELETE | A, S (delegation) | Calls `grant_permission()`; see `roles-permissions.md`. |

## 2. Catalog and discovery

| Route | Methods | Access | Notes |
|---|---|---|---|
| `/api/catalog/products/` | GET | P | Paginated active products, filters, sorts, search `q`; cached. |
| `/api/catalog/products/{slug}/` | GET | P | Variants, specs, images, ETA, rating summary, breadcrumbs, related. |
| `/api/catalog/categories/`, `/brands/` | GET | P | Tree and listings; brand pages. |
| `/api/catalog/search/suggest/` | GET | P | Autocomplete. Throttle `suggest`. |
| `/api/catalog/compare/` | GET | P | `?ids=` maximum 4. |
| `/api/home/` | GET | P / C | Sections: popular, trending, new, on sale, featured, categories, promo content; `recommended` only for C with enough data. |
| `/api/me/recommendations/` | GET | C | Empty list when data is insufficient. |
| `/api/me/recently-viewed/` | GET, POST | C | Guests keep this client-side or via cart token. |
| `/api/admin/products/`, `/variants/`, `/categories/`, `/brands/`, `/specs/`, `/recommendation-tags/`, `/coupons/`, `/promotions/` | CRUD | A | Admin-only models; staff serializers include cost and margin. |

## 3. Cart, checkout, payments

| Route | Methods | Access | Notes |
|---|---|---|---|
| `/api/cart/` and `/api/cart/items/` | GET, POST, PATCH, DELETE | C / P (cart token) | Totals, ETA, availability issues. |
| `/api/cart/merge/` | POST | C | Merge guest cart at login. |
| `/api/wishlist/` and `/items/` | CRUD, `move_to_cart` | C | |
| `/api/coupons/validate/` | POST | C / P | Throttle `coupon`; uniform failure messages. |
| `/api/checkout/quote/` | POST | C / P | No side effects. |
| `/api/checkout/` | POST | C / P | Requires `Idempotency-Key`. Creates order, reservations, PayPal order. |
| `/api/payments/paypal/capture/` | POST | C / G-confirmation | Server-side capture, idempotent. |
| `/api/payments/paypal/webhook/` | POST | W | Verify signature; store event; process async. |

## 4. Orders and tracking

| Route | Methods | Access | Notes |
|---|---|---|---|
| `/api/orders/` | GET | C | Own history. Staff: S:ASSISTANT/PACKING/ADMIN scoped. |
| `/api/orders/{id}/` | GET | C / S / A | Customer serializer vs staff serializer. |
| `/api/orders/{id}/timeline/` | GET | C / S | Customer-readable events; staff see audit-level detail. |
| `/api/orders/{id}/cancel/` | POST | C / S | Before packing only. |
| `/api/orders/{id}/reorder/` | POST | C | |
| `/api/orders/{id}/receipt/` | GET | C | Download, re-download. |
| `/api/orders/{id}/receipt/send/` | POST | C | Body `{channel: email|sms}`; throttle `receipt_send`. |
| `/api/track/{token}/` | GET | G (TRACK) | Status, package, courier, events, ETA, masked details. |
| `/api/receipts/{token}/` | GET | G (RECEIPT) | PDF stream or signed redirect. |
| `/api/warranty/{token}/` | GET | G (WARRANTY) | Guest warranty tracking. |
| `/api/call-requests/{token}/` | POST | G (CALL_REQUEST) | |
| `/api/me/warranties/` | GET | C | Active and historical. |
| `/api/me/reviews/` and `/api/catalog/products/{slug}/reviews/` | CRUD / GET, POST | C / P | Eligibility enforced in service. |
| `/api/admin/orders/{id}/audit/` | GET | A, S:audit view | Complete order history. |

## 5. Fulfilment and inventory

| Route | Methods | Access | Notes |
|---|---|---|---|
| `/api/packing/queue/` | GET | S:PACKING | Orders awaiting packing. |
| `/api/orders/{id}/confirm_picking/` | POST | S:PACKING | Quantities per item. |
| `/api/orders/{id}/serials/validate/` | POST | S:PACKING | Dry-run validation per serial. |
| `/api/orders/{id}/mark_as_packed/` | POST | S:PACKING | Requires valid serials; sets `PACKED`. |
| `/api/orders/{id}/handover/` | POST | S:PACKING | Courier, tracking number, `HANDED_TO_DELIVERY_COMPANY`. |
| `/api/packing/issues/` | CRUD | S:PACKING | Packing issues. |
| `/api/shipments/{id}/events/` | GET, POST | S:PACKING / W | Manual and courier events. |
| `/api/inventory/stock/` | GET | S:STOCK | Levels by product and variant, reserved, low and out of stock. |
| `/api/inventory/movements/` | GET, POST | S:STOCK | Append-only; no update or delete. |
| `/api/inventory/serials/` | CRUD (limited), search | S:STOCK | State changes through actions only. |
| `/api/inventory/lots/` | CRUD | S:STOCK | Expiry. |

## 6. Care, refunds, returns, warranty

| Route | Methods | Access | Notes |
|---|---|---|---|
| `/api/care/cases/` | CRUD, `resolve`, `escalate`, `assign` | C (own) / S:ASSISTANT | Internal notes filtered for C. |
| `/api/care/cases/{id}/messages/` | GET, POST | C / S | Chat and case communications. |
| `/api/care/call-requests/` | CRUD | C / S | |
| `/api/care/escalations/` | GET, PATCH | A / S (create) | Urgent attention queue for admins. |
| `/api/care/customers/search/`, `/orders/search/` | GET | S:ASSISTANT | Throttled; audit searches of PII. |
| `ws://.../ws/chat/{conversation}/` | WebSocket | C / G / S | Ticket-based auth, origin validation. |
| `/api/refunds/` | GET, POST, `approve`, `reject` | C (request) / S:ASSISTANT / A | Service enforces caps and thresholds. |
| `/api/returns/` | CRUD, `approve`, `receive`, `inspect` | C / S / A | |
| `/api/warranty/claims/` | CRUD | S:ASSISTANT / A | |
| `/api/warranty/register/` | POST | C | Register by serial. |

## 7. Notifications, reports, admin

| Route | Methods | Access | Notes |
|---|---|---|---|
| `/api/admin/notification-templates/` | CRUD | A | Allowlisted placeholders. |
| `/api/admin/communications/` | GET | A / S | Per customer or order. |
| `/api/reports/<name>/` | GET (`?export=csv|xlsx|pdf`) | A / S with report grant | Filters per RPT-09; async export for large sets; sanitize spreadsheet cells. |
| `/api/admin/audit/` | GET | A | Read-only. |
| `/api/admin/config/` | GET, PATCH | A | Site configuration with audit. |
| `/healthz`, `/readyz` | GET | internal | No sensitive data. |

Django Admin (`/admin/`, relocated is optional) covers the management list in ADM-01 and calls the same services.

## 8. Throttle scopes

Define in `DEFAULT_THROTTLE_RATES` (starting points, tune with data): `login` 5/min, `register` 5/hour per IP, `otp` 3/min and 10/day per target, `coupon` 10/min, `suggest` 60/min, `receipt_send` 3/hour per order, `guest_token` 30/min per IP, `search` 120/min, `checkout` 10/min, `chat_message` 30/min, `user` 300/min, `anon` 60/min. Use a shared Redis cache and set `NUM_PROXIES` correctly.

## 9. Conventions

- Cursor pagination for large and fast-growing lists (orders, movements, audit, notifications); capped page size elsewhere.
- Error shape: `{ "code": "...", "detail": "...", "fields": {...}, "request_id": "..." }`. Do not leak stack traces, SQL, or provider payloads.
- Return 404 for objects the caller may not see; 403 only when the resource is visible but the action is forbidden.
- All mutating endpoints that must not run twice (`checkout`, `refunds`, `receipt/send`, `capture`) accept `Idempotency-Key`.
- Public catalog serializers never include cost price, margin, supplier, internal notes, or stock thresholds.
- Document every route's access in the OpenAPI schema using tags per role.
