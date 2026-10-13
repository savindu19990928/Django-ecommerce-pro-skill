# Commerce flows: how the pieces must behave

## Contents
1. Pricing, discounts, and estimated arrival
2. Cart
3. Checkout and PayPal
4. Stock reservation lifecycle and pre-orders
5. Packing, serial numbers, and delivery
6. Refunds
7. Returns
8. Receipts
9. Guest secure links
10. OTP verification
11. Notifications and consent
12. Warranty
13. Customer care and live chat
14. Search, recommendations, trending, reviews

## 1. Pricing, discounts, and estimated arrival

One pricing service computes everything the cart, checkout, and order creation show, so numbers never differ between screens.

```python
def effective_price(variant, now):
    on_sale = variant.sale_price is not None and (variant.sale_starts_at or now) <= now <= (variant.sale_ends_at or now)
    return variant.sale_price if on_sale else variant.price

def quote(lines, *, coupon=None, address=None, shipping_method=None, today):
    # 1) unit prices from DB  2) line discounts (campaigns)  3) coupon  4) allocate order-level discount across lines
    # 5) shipping  6) tax  7) ETA  -> returns Quote(lines, subtotal, discount_total, shipping_total, tax_total, grand_total, eta)
```

- Apply discounts in a fixed order (campaign, then coupon), cap discounts at the line total, and allocate order-level discounts to lines proportionally with the remainder assigned to the largest line so allocations sum exactly. Store the allocation on each `OrderItem` because partial refunds depend on it.
- Display the final discount and each applied promotion in the quote.

**Estimated arrival (business rule: slowest product wins).**

```python
def estimate_arrival(lines, *, start: date):
    if not lines:
        raise ValueError("empty cart")
    per_item = {l.variant_id: start + timedelta(days=l.estimated_arrival_days) for l in lines}
    return max(per_item.values()), per_item
```

`estimated_arrival_days` is the variant override if set, else the product value. Use `start = today` in cart and checkout, and the placement date when the order is created. Persist `Order.estimated_arrival_date` and each `OrderItem.estimated_arrival_date`. Test with the spec example: items at 10, 30, and 15 days produce the 30-day date. Include the date and item-level dates in the cart, checkout, confirmation, order details, receipt, email and SMS templates, and tracking responses. Recompute (and notify) if an item is removed before fulfilment or a pre-order slips.

## 2. Cart

- Registered carts belong to the user. Guest carts use an opaque random token (header `X-Cart-Token` or `HttpOnly` cookie); store only its hash. On login, merge the guest cart into the user cart (sum quantities, cap at stock or per-item maximum).
- Validate on add, on view, and again at checkout: variant active, product active, availability mode, quantity limits, stock (or pre-order capacity).
- Return live prices. `added_price_snapshot` exists only to show "price changed since you added this".
- Pre-order and in-stock items can share a cart. Label each line, and make the order ETA the latest date.

## 3. Checkout and PayPal

Flow (registered and guest share the same code path; guest adds contact and address fields and a guest contact record):

1. `POST /api/checkout/quote/`: returns totals, ETA, and any availability issues. No side effects.
2. `POST /api/checkout/` with an `Idempotency-Key` header: in one transaction re-quote from the database, validate stock, create `Order` (`PENDING`) with item snapshots, create `Reservation`s with a short expiry (for example 20 minutes), apply coupon redemption provisionally, create `Payment` (`CREATED`). After commit, create the PayPal order and store `provider_order_id`. Return the PayPal order ID for the client to approve.
3. Client approves in PayPal (JS SDK or redirect).
4. `POST /api/payments/paypal/capture/` with the order public ID: the server captures. This endpoint is the only place a customer-facing request can trigger capture.
5. PayPal webhooks confirm and repair state. Reconciliation catches anything missed.

PayPal calls (Orders API v2). Use a thin client (or PayPal's current server SDK, verified on PyPI) that caches the OAuth token, sets timeouts, and sends `PayPal-Request-Id` on every POST so retries are safe.

- Create: `POST /v2/checkout/orders` with `intent: CAPTURE`, one purchase unit, `amount` and an item breakdown that matches the local totals exactly, `custom_id` or `invoice_id` = our order public ID, and `PayPal-Request-Id` derived from the local payment ID.
- Capture: `POST /v2/checkout/orders/{id}/capture` with `PayPal-Request-Id`.
- Refund: `POST /v2/payments/captures/{capture_id}/refund` with `PayPal-Request-Id` = the persisted refund idempotency key.
- Read: `GET /v2/checkout/orders/{id}` for reconciliation.

Capture handling:

```python
def capture_payment(order_public_id, actor):
    payment = Payment.objects.select_related("order").get(order__public_id=order_public_id)
    authorize_payer(actor, payment.order)                     # owner, or guest capability bound to this order
    if payment.status == Payment.Status.CAPTURED:
        return payment                                        # idempotent
    result = paypal.capture(payment.provider_order_id, request_id=f"capture-{payment.pk}")   # network call, outside any transaction
    unit = result["purchase_units"][0]["payments"]["captures"][0]
    with transaction.atomic():
        payment = Payment.objects.select_for_update().get(pk=payment.pk)
        if payment.status == Payment.Status.CAPTURED:
            return payment
        if unit["status"] == "COMPLETED" and Decimal(unit["amount"]["value"]) == payment.amount and unit["amount"]["currency_code"] == payment.currency:
            mark_paid(payment, capture_id=unit["id"])         # sets payment CAPTURED, order PAID, commits reservations,
                                                              # writes OrderEvent + AuditLog, queues receipt + notifications
        elif unit["status"] in {"PENDING"}:
            mark_capture_pending(payment, unit)               # order stays PAYMENT_PROCESSING; webhook finalizes
        else:
            mark_failed(payment, unit)                        # release reservations, order stays PENDING/CANCELLED per policy
    return payment
```

- Treat any amount or currency mismatch as a fraud signal: do not mark paid, flag the order, alert.
- **Webhooks**: endpoint is `csrf_exempt`, verifies the signature with PayPal's verification API or SDK before touching state, stores `WebhookEvent` with a unique `event_id` (duplicate deliveries return 200 and do nothing), then processes asynchronously. Handle at least `PAYMENT.CAPTURE.COMPLETED`, `PAYMENT.CAPTURE.PENDING`, `PAYMENT.CAPTURE.DENIED`, `PAYMENT.CAPTURE.REFUNDED`, and `CHECKOUT.ORDER.COMPLETED`/`APPROVED`. Webhook processing calls the same service functions as the capture endpoint.
- **Reconciliation task** (every few minutes): find payments in `CREATED`, `APPROVED`, or `CAPTURE_PENDING` older than a threshold, `GET` the PayPal order, and apply the same transitions. Expire unpaid orders and release reservations after the window.
- Never log full PayPal payloads at info level, never store card data, and keep credentials in the secret manager with separate sandbox and live sets.
- Order completion must match the real payment result: no path may set `PAID` without a verified capture record.

## 4. Stock reservation lifecycle and pre-orders

| Moment | Effect |
|---|---|
| Checkout creates order | `reserved += qty` (guarded `UPDATE`), `Reservation(ACTIVE, expires_at)`, `StockMovement(RESERVE)` |
| Payment captured | Reservation `COMMITTED`; reserved stays until packing |
| Packing confirmed | `on_hand -= qty`, `reserved -= qty`, `StockMovement(SALE)`, serials move to `PACKED` |
| Cancel, payment failed, expiry | `reserved -= qty`, reservation `RELEASED` or `EXPIRED`, `StockMovement(RELEASE)`, serials back to `IN_STOCK` |
| Return received and inspected | Restock (`RETURN` movement, serial `IN_STOCK`) or damaged (`DAMAGE` movement, serial `DAMAGED`) |

- Lock rows in primary-key order and keep the transaction short; retry deadlocks (see `domain-model.md`).
- A Celery beat task expires stale reservations and posts the movements.
- **Pre-orders**: when `availability_mode = PREORDER`, orders skip stock reservation and go to `PREORDER` status after payment, with a pre-order allocation counter. When stock arrives (an `ADDITION` movement), allocate to the oldest pre-orders first, move them to `PROCESSING`, and notify customers. Mark pre-order status on product, cart, checkout, confirmation, order details, and notifications.
- A nightly job verifies `on_hand == sum(movements)`, `reserved == sum(active reservations)`, and that every `PACKED` or `SOLD` serial has an order item. Mismatches raise alerts, never silent fixes.
- Low-stock alerts: when `available <= low_stock_threshold`, notify permitted `STOCK` users once per crossing (dedupe key), and expose the list.

## 5. Packing, serial numbers, and delivery

Endpoints and states (role `PACKING` plus grants; see `roles-permissions.md`):

1. Queue: orders in `PAID`/`CONFIRMED`/`PROCESSING`/`PICKING`, oldest first, with items, quantities, and packing-relevant product data (no cost or margin).
2. Confirm picking: per item, quantity must match, otherwise create a `PackingIssue` (SHORT_STOCK, DAMAGED).
3. Serial entry (for `track_serials` items): one serial per unit. Validate in one atomic call:
   - Normalize (trim, uppercase, strip separators per product rule) and match the product's format regex.
   - Must exist for that variant in `IN_STOCK` or `RESERVED` (reserved for this order); duplicates in the same request are rejected.
   - Invalid, duplicate, missing, or damaged serials create a `PackingIssue` and block `mark_as_packed` until resolved; they never silently pass.
   - Transition each serial to `PACKED`, link to the order item, write `SerialEvent`.
4. `POST /api/orders/{id}/mark_as_packed/`: requires all items picked and all serials valid. Creates the `Package`, sets order `PACKED`, deducts stock (section 4), writes timeline and audit, queues notification. If `ORDER_AUTO_IN_TRANSIT_ON_PACKED` is true, it also moves to `IN_TRANSIT`.
5. Handover: courier, tracking number, `HANDED_TO_DELIVERY_COMPANY` (packing user confirms dispatch). Notify the customer.
6. Delivery updates: `ShipmentEvent` rows from manual updates, courier webhooks (verify signatures), or polling; map courier statuses to `IN_TRANSIT`, `OUT_FOR_DELIVERY`, `DELIVERED`, `DELIVERY_FAILED`, return-to-sender. Record each attempt. On `DELIVERED`, set serials to `DELIVERED`, start warranty if the rule is "delivery", and notify.

Every step writes an `OrderEvent` (customer-readable where appropriate) and an `AuditLog`. Packing history per order is a filtered view of those events.

## 6. Refunds

Rules: full, partial amount, and line-item refunds; assistants refund according to grants and thresholds; the total never exceeds what is refundable; every action is audited.

```python
def request_refund(actor, order, *, items=None, amount=None, reason):
    with transaction.atomic():
        payment = Payment.objects.select_for_update().get(order=order, status__in=REFUNDABLE_STATES)
        lines = resolve_lines(order, items)                       # quantity-limited by item.quantity - item.refunded_quantity
        computed = refund_amount_for_lines(lines)                 # unit price - allocated discount + tax, per unit
        total = amount or computed                                # a manual partial amount cannot exceed computed refundable for those lines
        pending = payment.refunds.filter(status__in=OPEN_STATES).aggregate(s=Sum("amount"))["s"] or 0
        refundable = payment.amount - payment.refunded_total - pending
        if total <= 0 or total > refundable:
            raise RefundExceedsRefundable(refundable)
        status = APPROVED if actor_can_approve(actor, total) else REQUESTED
        refund = Refund.objects.create(payment=payment, order=order, amount=total, status=status, reason=reason,
                                       requested_by=actor, idempotency_key=uuid.uuid4())
        RefundItem.objects.bulk_create(refund_items(refund, lines))
        audit.record("refund.created", actor=actor, order=order, refund=refund, after=serialize(refund))
    if refund.status == APPROVED:
        process_refund.delay(refund.pk)                           # Celery task, after commit
    return refund

def process_refund(refund_id):
    refund = mark_processing(refund_id)                           # short transaction with row lock
    result = paypal.refund(refund.payment.provider_capture_id, amount=refund.amount, currency=..., request_id=str(refund.idempotency_key),
                           invoice_id=str(refund.public_id), note=...)
    finalize_refund(refund, result)                               # COMPLETED/PENDING/FAILED; updates payment.refunded_total, order status
                                                                  # (PARTIALLY_REFUNDED or REFUNDED), item refunded quantities, stock/serial effects, notifications
```

- Customer-initiated requests are created as `REQUESTED` (staff approve or reject). Staff-initiated refunds are `APPROVED` unless the amount exceeds the actor's `constraints_json` cap or the configured admin-approval threshold; the initiator cannot approve their own refund above the threshold.
- The PayPal call happens outside the database lock, with the idempotency key persisted before the call. A retry after a timeout reuses the same key, so PayPal cannot double-refund.
- `PAYMENT.CAPTURE.REFUNDED` webhooks confirm and repair state idempotently.
- PayPal limits how long refunds are possible after capture; surface a clear error and offer an alternative workflow (store credit or manual process) when it is exceeded.
- Refund events appear in the customer timeline, audit log, refund and financial reports, and trigger notifications (refund, partial refund).

## 7. Returns

`ReturnRequest` flow: customer requests (within `ReturnRule` window, eligible items and quantities) -> staff approve or reject -> customer ships back (tracking recorded) -> received -> inspected (outcome per item: restock, damaged, scrap) -> refund created through `request_refund` -> closed. Serial numbers move to `RETURNED` and then `IN_STOCK` or `DAMAGED`. Each step is audited and visible in the timeline. Return rules (window, categories, condition) are admin-editable.

## 8. Receipts

- Generate after payment is confirmed (Celery task triggered from `mark_paid` via `on_commit`). Number format is unique and stored on `Receipt`. Content follows feature RCP-01 and uses order snapshots only.
- PDF engine: ReportLab (pure Python) or WeasyPrint (HTML to PDF). If using HTML rendering, escape all user-supplied text and disable remote and `file://` fetching in the renderer's URL fetcher.
- Store in the private GCS bucket under a random object name, record `sha256`, and never expose the bucket path.
- Retrieval: registered customer endpoint (ownership check) or guest receipt token; stream through the API or redirect to a short-lived signed URL (5-15 minutes). `Cache-Control: no-store`.
- Resend by email or SMS: registered customers use their stored contact; guests can only trigger resend to the contact on the order, never to a new address supplied in the request. Throttle per order (for example 3 per hour), log in communication history, audit.
- If the file is missing (retention, failure), regenerate from snapshots and bump `version`.
- Retention follows policy: GCS lifecycle and retention rules, and a documented anonymization path that keeps legally required financial data.

## 9. Guest secure links

Purposes: `TRACK`, `RECEIPT`, `WARRANTY`, `CALL_REQUEST`.

```python
def issue_guest_token(order, purpose, ttl):
    raw = secrets.token_urlsafe(32)                                # 256 bits
    GuestAccessToken.objects.create(order=order, purpose=purpose,
                                    token_hash=hashlib.sha256(raw.encode()).hexdigest(),
                                    expires_at=timezone.now() + ttl)
    return raw                                                     # goes only into the email/SMS link, never stored raw

def resolve_guest_token(raw, purpose):
    h = hashlib.sha256(raw.encode()).hexdigest()
    tok = GuestAccessToken.objects.select_related("order").filter(token_hash=h, purpose=purpose, revoked_at__isnull=True, expires_at__gt=timezone.now()).first()
    if tok is None:
        raise Http404                                              # identical response for unknown, expired, revoked, wrong purpose
    return tok
```

- The token authorizes one purpose for one order. A tracking token cannot download a receipt. Responses return only what that purpose needs, with address, phone, and email partially masked.
- Suggested lifetimes (configurable): tracking 180 days after delivery, receipt 1 year with the ability to request a fresh link, warranty until the warranty end plus a grace period, call request 30 days.
- Throttle by IP and by token hash prefix, send `Referrer-Policy: no-referrer` and `Cache-Control: no-store`, keep tokens out of logs (strip query and path segments in access logs and error reports), and revoke tokens on suspected leak or when a guest converts to an account (links keep working only if the owner chooses).
- "Request link again" sends a new token only to the contact on the order (rate limited, uniform response whether or not the order exists).

## 10. OTP verification

```python
code = f"{secrets.randbelow(10**6):06d}"
digest = hmac.new(settings.OTP_KEY, f"{channel}:{target}:{code}".encode(), hashlib.sha256).hexdigest()
```

- Store only the HMAC, expiry 5-10 minutes, single use, maximum 5 verify attempts then lock for a cooling-off period, resend cooldown of about 60 seconds, new code invalidates the old one.
- Rate-limit sends per target, per user, per IP, and per day; add bot protection on public send endpoints. Unthrottled SMS endpoints are a direct cost attack (SMS pumping), so also cap by country or destination prefix and alert on spikes.
- Responses never reveal whether an account or contact exists. Compare in constant time. Changing a verified phone or email resets its verified flag and re-runs verification.
- `is_phone_verified` gates reviews (REV-01) and phone-based guest conversion.

## 11. Notifications and consent

- Pipeline: event -> `notify(event, order/user, context)` -> choose channels from preferences and consent -> create `Notification` rows (with a `dedupe_key`, for example `order:123:shipped:sms`) inside the business transaction -> `on_commit` enqueue -> worker renders and sends -> provider webhook updates delivery status -> retries with exponential backoff and a cap.
- **Transactional vs marketing**: order, payment, refund, security, and receipt-on-request messages are transactional (still honoring channel preferences where the business rules allow). Promotional, recommended-sale, wishlist-price, and abandoned-cart messages require explicit consent for that channel, recorded in `ConsentRecord`, with unsubscribe links in email and STOP handling for SMS. A suppressed message is stored as `SUPPRESSED` with the reason.
- **Templates**: admin-edited bodies use allowlisted placeholders (`{{order.number}}`, `{{customer.first_name}}`, `{{order.estimated_arrival_date}}`, `{{links.tracking}}`) rendered by a small substitution function, not the full Django template language, so a template cannot reach arbitrary object attributes or call methods. Escape values for the channel (HTML for email).
- SMS and email content never includes full card or payment details, and links are the guest tokens above. Keep SMS short and avoid personal data beyond what the message needs.
- WhatsApp: the care contact method is a click-to-chat link (`https://wa.me/<number>?text=...`) with order number pre-filled (no sensitive data), or the WhatsApp Business API if the user wants two-way logging into `CaseMessage` with channel `WHATSAPP`.

## 12. Warranty

- Policy per product or variant: duration, eligibility, start rule (purchase, delivery, registration), end rule.
- Create `Warranty` rows when the start condition occurs (for delivery-based rules, on `DELIVERED`), tied to the serial number when the product is serial-tracked. Also allow registration by serial (FT): validate the serial exists, is `SOLD` or `DELIVERED`, and is not already registered by someone else.
- Registered customers see covered product, serial, start, end, remaining period, status, active and historical. Guests use the warranty token (section 9). Staff create claims (`WarrantyClaim`), which can set serial state to a warranty-related state and trigger notifications.
- Expiry job marks `EXPIRED` and (with consent) sends reminders.

## 13. Customer care and live chat

- A `CareCase` ties together customer or guest, order, messages, priority (NORMAL or URGENT), assignee, and status. Internal notes (`is_internal`) are filtered out in customer serializers and queryset, never just hidden in the UI.
- Escalation creates an `Escalation` with reason, customer, order, creator, time, status, and resolution, flags the case `URGENT`, notifies administrators, and appears in the admin list.
- Phone call requests: logged-in customers or guest `CALL_REQUEST` token; one open request per order, rate limited; staff manage status and outcome.
- **Live chat (Channels)**: authenticate in the WebSocket handshake with a short-lived one-time ticket obtained over an authenticated HTTPS call (do not put long-lived JWTs in the URL), validate `Origin`, authorize every conversation join (customer owns it, guest token matches, or staff has the grant), rate-limit messages and cap size, persist every message, support reconnect with history, and apply the retention policy. Attachments go through signed upload URLs with type and size limits and server-side validation.
- Care staff can view payment summary (status, amounts, masked payer email, capture ID) but not raw provider payloads or credentials.

## 14. Search, recommendations, trending, reviews

- **Search**: `SearchService` with MySQL `FULLTEXT` (names, descriptions, specs text) plus exact and prefix matches for SKU, brand, category; relevance score combined with a small popularity boost. Filters and sorts per DIS-02 and DIS-03 via `django-filter`; ordering and filter fields are allowlisted. Cache public catalog responses (short TTL, invalidated on product change). Autocomplete requires a minimum length, is throttled, and returns only active products.
- **Recommendations**: events (`VIEW`, `SEARCH`, `WISHLIST_ADD`, `CART_ADD`, `PURCHASE`) are recorded asynchronously with a retention limit. A periodic task aggregates `CustomerInterest` (category, tag, brand scores with time decay). Serving: if the customer has fewer than the configured minimum signals (for example 2 purchases or 5 interactions in 90 days), return an empty list so the front end hides the section. Otherwise return active, available products matching top interests via recommendation tags and categories, excluding already purchased items, ranked by interest overlap plus popularity. The "sale" variant filters to on-sale items. Sale notifications go only to customers with the explicit consent flag.
- **Trending**: periodic task computes a score per product from weighted recent sales, views, wishlist adds, searches, and rating, with decay, using the admin-editable `TrendingConfig`; results are cached and exposed to the homepage, sorting, and reports.
- **Reviews**: the service verifies login, verified phone, a completed eligible purchase (paid and delivered order item, not fully refunded), one review per customer and product, and rating 1-5. Update `rating_avg`, `rating_count`, and the star breakdown atomically. Moderation hides content without deleting rows. Guests cannot rate; they can after converting to a verified account (ACC-09).
