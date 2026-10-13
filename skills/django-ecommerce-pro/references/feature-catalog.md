# Feature catalog and coverage matrix seed

Use this file to build `docs/feature-coverage.md` in the project. Track each ID as `done`, `partial`, `deferred (reason)`, or `out of scope (user decision)`.

**Source tags**: `DC` = DevCommerce functional specification (section number), `FT` = FEATURES.md, `UR` = USER_ROLES.md, `STD` = standard e-commerce capability added by this skill (implied by the documents or expected from a modern shop).
**Priority**: `M` must ship in the baseline, `S` should ship unless the user opts out, `O` optional module (offer, do not build unprompted).

## Contents
1. Accounts and identity
2. Permissions and roles
3. Catalog
4. Discovery
5. Reviews
6. Cart and wishlist
7. Promotions
8. Checkout and payments
9. Orders and tracking
10. Receipts and documents
11. Notifications
12. Warranty
13. Inventory and serial numbers
14. Packing and delivery
15. Customer care
16. Refunds and returns
17. Audit
18. Reports
19. Administration and content
20. Platform and compliance

## 1. Accounts and identity
- ACC-01 [DC3.1] Customer registration with password, optional email verification. M
- ACC-02 [DC3.1, FT] Login and logout using JWT, with refresh rotation and blacklist on logout. M
- ACC-03 [DC3.1, FT] Forgot, reset, and change password; all refresh tokens invalidated on change. M
- ACC-04 [FT, DC3.3] Contact verification by 6-digit OTP via email or SMS; `is_email_verified` and `is_phone_verified` tracked. M
- ACC-05 [DC3.2] Profile: name, email, phone, profile photo (private GCS), default delivery address, account settings. M
- ACC-06 [FT, DC3.2] Address book: billing and shipping addresses, default flags, E.164 phones. M
- ACC-07 [DC2.1] Saved contact information. S
- ACC-08 [DC21] Notification and communication preferences (see NOT-05). M
- ACC-09 [DC2.2] Guest-to-customer conversion: a guest who registers and verifies the same email or phone can claim earlier guest orders and become eligible for ratings. S
- ACC-10 [FT] Administrative user management: create, deactivate, assign role. M
- ACC-11 [UR] Superuser-only endpoint `/api/users/superusers/` to promote and demote roles and manage core user data. M
- ACC-12 [STD] Staff MFA (TOTP) for ADMIN, ASSISTANT, STOCK, PACKING; Django Admin requires it. S
- ACC-13 [STD] Account data export and deletion/anonymization request (retaining legally required order and payment records). S
- ACC-14 [STD] Active sessions or devices list and revoke. O

## 2. Permissions and roles
- PRM-01 [UR] Roles: CUSTOMER, ASSISTANT, STOCK, PACKING, ADMIN, plus Django superuser. M
- PRM-02 [UR, FT] `GranularPermission` records with view/add/change/delete/delegate flags, per model (global) or per object, via generic foreign key. M
- PRM-03 [UR] Customers have hardcoded access to their own data only; no grants needed. M
- PRM-04 [UR] Staff roles ASSISTANT, STOCK, PACKING are default-deny until granted. M
- PRM-05 [UR] ADMIN has global API access, can assign any grant to any user (within the allowlist), cannot create or delete superusers unless a superuser. M
- PRM-06 [UR] Peer delegation through `/api/users/permissions/`, with the three security guarantees (same role, cannot grant what you lack, cannot delegate without delegation right). M
- PRM-07 [UR] Admin-only models (Product, Category, Brand, Coupon, RecommendationTag) reject granular grants. M
- PRM-08 [DC2.6] Different Django Admin users see only the areas assigned to them (groups and model permissions). M
- PRM-09 [STD] Grant lineage, cascade revocation, optional expiry, optional constraints JSON (for example refund amount cap). S
- PRM-10 [STD] Permission changes are audit logged and visible to ADMIN. M

## 3. Catalog
- CAT-01 [DC4.1, FT] Product: name, slug, description, short description, brand, SKU, status (active/inactive), search metadata, weight and dimensions, unit of measure, tags. M
- CAT-02 [DC4.1, FT] Pricing: price, sale price (with start/end), currency, cost price and margin (staff-only). M
- CAT-03 [FT] Expiration date support (product-level date, lot-level where stock is perishable), with expiring-soon report. S
- CAT-04 [DC4.2] Hierarchical categories (main, sub, child); a product belongs to many categories. M
- CAT-05 [DC4.3] Brands: create, edit, activate/deactivate, brand listing pages, brand filter. M
- CAT-06 [DC4.4] Flexible specification key/value pairs; filterable keys are flagged. M
- CAT-07 [DC6] Variants (color, size, model, capacity, custom) with own SKU, price, sale price, stock, images, availability, serial tracking. M
- CAT-08 [DC4.1, DC46] Images and media in GCS with alt text, ordering, gallery, zoom-ready sizes (thumbnails generated asynchronously). M
- CAT-09 [DC4.1, DC22] Warranty configuration per product or variant (duration, eligibility, start and end rule). M
- CAT-10 [DC12, DC4.1] Estimated arrival days on every product (variant override). M
- CAT-11 [DC11] Availability mode: immediate, pre-order, temporarily unavailable, out of stock. M
- CAT-12 [DC4.1, DC23] Serial-number tracking flag per product or variant. M
- CAT-13 [DC4.1, FT, UR] Recommendation tags (admin-only) and general product tags. M
- CAT-14 [DC4.1] Related products and "frequently bought together" links. M
- CAT-15 [DC48] Catalog management without code changes (Django Admin plus admin API). M
- CAT-16 [STD] Bulk import and export of products (CSV) with validation report. S
- CAT-17 [STD] Product Q&A. O

## 4. Discovery
- DIS-01 [DC5.1] Search by name, SKU, brand, category, keywords, tags, specs, ranked by relevance. M
- DIS-02 [DC5.2] Filters: brand, category (multi), price range, availability, on sale, pre-order, rating, color, size, attributes, configured filters. M
- DIS-03 [DC5.3] Sorting: relevance, popularity, trending, newest, price asc/desc, highest rating, most reviewed, on sale. M
- DIS-04 [DC35] Search suggestions and autocomplete. S
- DIS-05 [STD] Facet counts per filter. S
- DIS-06 [DC8, FT] Homepage sections: most popular, trending, new arrivals, recommended for you, on sale, featured products, featured brands, categories, promotional content. M
- DIS-07 [DC8.1, DC31] Personalized recommendations from purchases, viewed categories, tags, wishlist, search, brand interest; empty result when data is insufficient. M
- DIS-08 [DC8.2] Sale recommendations (recommended products currently on sale). M
- DIS-09 [DC32] Trending products from recent sales, views, wishlist, search, ratings, configurable weights. M
- DIS-10 [DC34] Product comparison (price, brand, ratings, specs, availability), maximum 4 products. M
- DIS-11 [DC35] Recently viewed products. S
- DIS-12 [DC35] Breadcrumb data (category path) in product responses. S
- DIS-13 [STD] SEO: slugs, meta fields, sitemap data, canonical URLs. S
- DIS-14 [STD] Search analytics (zero-result queries, top queries) with retention limits. S

## 5. Reviews
- REV-01 [DC7.1, UR] Ratings 1-5 only from logged-in, phone-verified customers who purchased the product. M
- REV-02 [DC7.2, FT] Written reviews with edit rules (window configurable) and "my reviews". M
- REV-03 [DC7.2] Moderation: hide/remove, review reports, rating activity. M
- REV-04 [DC7.2] Summary: average, total, per-star breakdown. M
- REV-05 [STD] Verified-purchase badge and helpful votes. O

## 6. Cart and wishlist
- CRT-01 [DC10] Cart: add products and variants, change quantity, remove, subtotal, discounts, charges, per-item and order ETA, proceed to checkout. M
- CRT-02 [DC10] Availability validation (at add, at view, at checkout). M
- CRT-03 [DC2.2] Guest cart via opaque cart token; merge into account cart on login. M
- CRT-04 [DC9, FT] Wishlist: add, remove, view, move to cart. M
- CRT-05 [DC9] Price-change and sale notifications for wishlist items, per consent. M
- CRT-06 [STD] Back-in-stock notification subscriptions. S
- CRT-07 [STD] Abandoned-cart reminders (consent required). O

## 7. Promotions
- PRO-01 [DC33] Sale and discount pricing, time-limited offers. M
- PRO-02 [DC33, FT, UR] Coupons (admin-only): percent, fixed, free shipping; minimum order; caps; validity window; total and per-user usage limits. M
- PRO-03 [DC33] Campaigns scoped to product, category, brand, or cart. M
- PRO-04 [DC33] Stacking and priority rules; final discount displayed in checkout. M
- PRO-05 [STD] Discount allocation across lines (needed for partial refunds). M
- PRO-06 [STD] Gift cards or store credit. O

## 8. Checkout and payments
- CHK-01 [DC13.1] Guest checkout: name, email, phone, delivery address, billing details where required. M
- CHK-02 [DC13.2] Registered checkout: saved or new address, promotions, ETA review. M
- CHK-03 [STD] Delivery methods and rates (zones, flat, weight-based, free over threshold). S
- CHK-04 [DC15, STD] Taxes and charges (inclusive or exclusive, per region or category). S
- CHK-05 [DC14.1] PayPal Checkout: initiate, approve, capture, success, failure, cancel, status tracking, transaction reference. M
- CHK-06 [DC14.1, DC43] Webhooks with signature verification, idempotent processing, reconciliation job for stuck payments. M
- CHK-07 [DC10, DC13] Final price and stock validation, stock reservation window during payment. M
- CHK-08 [STD] Idempotency key on order placement. M
- CHK-09 [STD] Terms acceptance record, order notes. S
- CHK-10 [STD] Bot protection on guest checkout and registration. S
- CHK-11 [STD] Multi-currency. O

## 9. Orders and tracking
- ORD-01 [DC15] Order data: number, customer or guest, items, quantities, prices, discounts, taxes, delivery info, payment status, order status, ETA, shipment info, warranty info. M
- ORD-02 [DC15.1] Statuses: Pending, Payment Processing, Paid, Confirmed, Awaiting Stock, Pre-order, Processing, Picking, Packing, Packed, Ready for Dispatch, Handed to Delivery Company, In Transit, Out for Delivery, Delivered, Delivery Failed, Cancelled, Refund Requested, Partially Refunded, Refunded, Returned, Completed. Transitions admin-configurable. M
- ORD-03 [DC12, DC50] Order ETA equals the latest item ETA; item-level ETAs shown. M
- ORD-04 [DC16, DC38] Order history and per-order timeline for customers (readable) and staff (audit-level). M
- ORD-05 [DC17] Tracking for registered customers (account) and guests (secure link). M
- ORD-06 [DC18] Guest links: tracking, receipt download, warranty tracking, plus phone-call request link. M
- ORD-07 [DC11] Pre-orders flagged in product, cart, checkout, confirmation, details, notifications. M
- ORD-08 [STD] Customer cancellation before packing, with automatic stock release and refund. S
- ORD-09 [STD] Reorder from history. S
- ORD-10 [STD] Tax invoice numbering and format where the jurisdiction requires it. O

## 10. Receipts and documents
- RCP-01 [DC19] PDF receipt with the contents listed in DC19.1, including ETA and warranty information. M
- RCP-02 [DC19.3, DC19.4] Email and SMS receipts with secure download links. M
- RCP-03 [DC19.5, FT] Registered customers can download, re-download, and resend by email or SMS. M
- RCP-04 [DC47, DC46] Receipts stored in private GCS, linked to order, retained per policy, retrieved without exposing other files. M

## 11. Notifications
- NOT-01 [DC20] Email and SMS, event-driven and configurable. M
- NOT-02 [DC20.1] Order events: placed, payment success or failure, confirmed, processing, packing, dispatched, handed over, tracking updates, delivery attempt, delivered, cancelled, refund, partial refund, return status. M
- NOT-03 [DC42] Admin-managed templates with placeholders (email and SMS). M
- NOT-04 [DC41] Communication history per customer and order. M
- NOT-05 [DC21] Preferences: order updates email/SMS, receipt email/SMS, promotional email/SMS, recommended-sale notifications, other categories. System-critical transactional notices follow separate business rules. M
- NOT-06 [DC20.3] Sale notifications only with explicit consent. M
- NOT-07 [STD] Delivery status tracking, retries with backoff, provider webhooks, consent log, unsubscribe and STOP handling. M
- NOT-08 [DC27.2] WhatsApp: click-to-chat link with order context, or business API integration. M

## 12. Warranty
- WAR-01 [DC22] Warranty configuration: duration, eligibility, start rule, end rule. M
- WAR-02 [FT] Warranty registration by serial number. M
- WAR-03 [DC22.1] Registered view: covered product, serial, start, end, remaining period, status; active and historical. M
- WAR-04 [DC22.2] Guest warranty tracking via secure link. M
- WAR-05 [UR] Warranty claims initiated by staff. M
- WAR-06 [STD] Warranty-expiry reminders (consent). O

## 13. Inventory and serial numbers
- INV-01 [DC24] Product and variant stock, reserved, available, damaged, lost. M
- INV-02 [FT, DC24] Stock movement ledger: additions, removals, damages, losses, adjustments, with actor and reason. M
- INV-03 [DC2.5] Low-stock and out-of-stock monitoring, alerts, and views of reserved and allocated stock (orders and pre-orders). M
- INV-04 [DC23, FT] Serial numbers with states: in stock, reserved, packed, sold, delivered, returned, damaged, warranty-related; duplicates and invalid serials prevented or flagged; full history. M
- INV-05 [DC11] Pre-order stock allocation (first in, first out) when stock arrives. M
- INV-06 [FT, STD] Lot or batch tracking with expiry (first expired, first out). S
- INV-07 [DC39.2] Stock valuation, stock reports. S
- INV-08 [STD] Stocktake and cycle counts, suppliers and purchase orders. O

## 14. Packing and delivery
- PCK-01 [DC2.4, DC25] Packing queue, picking confirmation, quantity checks. M
- PCK-02 [DC23, UR] Serial entry and validation per unit; `mark_as_packed` action. M
- PCK-03 [DC2.4] Flag missing, invalid, duplicate, or damaged serials and other packing issues. M
- PCK-04 [DC25] Package creation, handover to courier, dispatch confirmation, tracking number entry. M
- PCK-05 [DC26] Couriers, shipments, events, delivery attempts, failure, return to sender. M
- PCK-06 [DC26] Courier integration adapter with webhook or polling sync. S
- PCK-07 [STD] Shipping labels and packing slips (PDF). O

## 15. Customer care
- CAR-01 [DC27.1] Live chat for customers and guests, handled by assistants in the operational app. M
- CAR-02 [FT] Care cases with in-case messaging and communications; internal notes hidden from customers. M
- CAR-03 [DC27.3, FT] Phone-call requests (logged in or via secure order link), managed by staff. M
- CAR-04 [DC28] Escalation: normal vs urgent attention, visible to administrators, with reason, customer, order, creator, time, status, resolution. M
- CAR-05 [DC2.3] Customer, order, payment-summary, and communication history lookup. M
- CAR-06 [DC2.3] Resolve cases; assign cases; support notes. M
- CAR-07 [STD] Satisfaction rating after case closure (feeds assistant reports). S

## 16. Refunds and returns
- REF-01 [DC29] Full, partial, and line-item refunds with reason, amount, status, timestamp, payment link, initiator. M
- REF-02 [DC29] Hard cap at the refundable amount; audit logged. M
- REF-03 [FT] Customer-initiated refund requests approved by staff. M
- REF-04 [DC14.1] Refunds executed through PayPal with idempotency and webhook confirmation. M
- RET-01 [DC30] Returns: request, reason, items, approval or rejection, tracking, refund linkage, history. M
- RET-02 [DC30] Admin-configurable return rules (window, eligible categories, condition). M
- RET-03 [STD] Return inspection outcome (restock, damaged, scrap) feeding stock and serial state. S

## 17. Audit
- AUD-01 [DC37] Audit record: action ID, action, user, date/time, related order; optional customer, product, payment, refund, before/after, context. M
- AUD-02 [DC37] Covers login/security events, order, payment, refund, product and price, stock and serial, packing and handover, tracking, profile, recommendation admin, warranty, notification resend, permission change, escalation. M
- AUD-03 [DC37] Protected from modification (application and database level). M
- AUD-04 [DC38] Per-order audit view for admins and permitted staff. M

## 18. Reports
- RPT-01 [DC39.1] Financial: sales, revenue, discounts, refunds, partial refunds, net sales, payment totals, sales over time. M
- RPT-02 [DC39.2] Stock reports. M
- RPT-03 [DC39.3] Product: sales, performance, views, ratings, distribution, review counts, damaged stock, refunds and returns, popularity, by category and brand. M
- RPT-04 [DC39.4] Orders: by date, status, customer, product, pre-orders, completed, cancelled, returned, refunded, delivery performance. M
- RPT-05 [DC39.5] Care: cases, status, urgent, refunds processed, ratings, performance. M
- RPT-06 [DC39.6] Trending products and categories. M
- RPT-07 [DC39.7] Customers: registrations, active, spend, frequency, segmentation, guest vs registered. M
- RPT-08 [FT] Dashboard metrics: gross revenue, order volume, catalog volume, low-stock alerts. M
- RPT-09 [DC40] Filters (date range, product, brand, category, customer, order status, payment status, staff user) and CSV, Excel, PDF export, with spreadsheet formula-injection protection. M

## 19. Administration and content
- ADM-01 [DC45] Django Admin for customers, staff, permissions, products, categories, brands, specs, variants, promotions, orders, payments, refunds, inventory master data, warranties, templates, escalations, audit logs (read-only), reports, site configuration. M
- ADM-02 [DC2.6] Delivery and shipping settings, pre-order settings, recommendation configuration, trending weights. M
- ADM-03 [DC2.6] Website content and settings (banners, promotional content, static pages). S
- ADM-04 [STD] Feature flags and system configuration with change audit. S

## 20. Platform and compliance
- PLT-01 [DC51] Responsive-friendly, cacheable catalog API; fast discovery. M
- PLT-02 [STD] OpenAPI schema (drf-spectacular), consistent error format, request IDs. M
- PLT-03 [DC43] Rate limiting, bot protection, secure headers, secrets management. M
- PLT-04 [STD] Health, readiness, and metrics endpoints; structured logging; error monitoring. M
- PLT-05 [STD] Privacy: consent records, retention schedules, PII minimization, export and erasure. S
- PLT-06 [STD] Backups and restore drills for MySQL, versioning and retention policy on GCS. M
- PLT-07 [STD] Outbound webhooks for integrations (ERP, accounting). O
- PLT-08 [STD] Multi-language and localization. O
