# Roles, granular permissions, and delegation

The model is hybrid RBAC plus ABAC: a user has one primary `role`, and staff additionally need explicit `GranularPermission` records. This file keeps the documented behavior (USER_ROLES.md) and closes the escalation paths a generic-foreign-key permission system tends to have.

## Contents
1. Role capability matrix
2. Data model
3. Evaluation and queryset scoping
4. Grant, revoke, and delegation service
5. DRF enforcement
6. Admin-only models and sensitive models
7. Django Admin access
8. Permission tests

## 1. Role capability matrix

| Role | Access model | Typical capabilities (once granted where applicable) |
|---|---|---|
| Superuser (`is_superuser`) | Bypasses grants | Everything, including `/api/users/superusers/` to promote and demote roles. |
| `ADMIN` | Global API access | All operational endpoints, grant any permission (within the allowlist) to any user, admin-only models (Product, Category, Brand, Coupon, RecommendationTag), reports. Cannot create or delete superusers. |
| `CUSTOMER` | Hardcoded ownership scope, no grants | Own orders, cart, wishlist, addresses, cases and case messages, call requests, reviews (needs verified phone and purchase), warranties, receipts. |
| `ASSISTANT` | Default deny, grants required | View orders, customer lookup, payment summary, communication history; chat and cases; call requests; refunds and warranty claims; escalate to admins; resolve cases. |
| `STOCK` | Default deny, grants required | Stock items, movements, lots, serial numbers, low-stock and reservation views, stock reports. Read-only product summary inside inventory endpoints. |
| `PACKING` | Default deny, grants required | Packing queue, picking confirmation, serial entry and validation, packing issues, `mark_as_packed`, package handover, tracking entry, packing history. |
| Guest | Capability tokens, no account | Only the action and order a token authorizes (see `commerce-flows.md`). |

Examples from the documents that must work: an admin grants a `STOCK` user view and change on stock item 4 but not delete; a `STOCK` user with global delegation on stock items grants another `STOCK` user view on one stock item only; `ASSISTANT A` with delegation on `customercarecase` grants `ASSISTANT B` view and change but not delegation.

## 2. Data model

```python
GLOBAL = 0  # sentinel; MySQL unique indexes allow many NULLs, so NULL cannot represent "global" safely

class GranularPermission(models.Model):
    user = models.ForeignKey(settings.AUTH_USER_MODEL, on_delete=models.CASCADE, related_name="granular_permissions")
    content_type = models.ForeignKey(ContentType, on_delete=models.CASCADE)
    object_id = models.PositiveBigIntegerField(default=GLOBAL)   # 0 = every object of the model
    can_view = models.BooleanField(default=False)
    can_add = models.BooleanField(default=False)
    can_change = models.BooleanField(default=False)
    can_delete = models.BooleanField(default=False)
    can_delegate = models.BooleanField(default=False)
    constraints_json = models.JSONField(default=dict, blank=True)  # e.g. {"max_refund_amount": "100.00"}
    granted_by = models.ForeignKey(settings.AUTH_USER_MODEL, null=True, on_delete=models.SET_NULL, related_name="grants_made")
    parent = models.ForeignKey("self", null=True, blank=True, on_delete=models.CASCADE, related_name="derived")
    expires_at = models.DateTimeField(null=True, blank=True)
    created_at = models.DateTimeField(auto_now_add=True)

    class Meta:
        constraints = [models.UniqueConstraint(fields=["user", "content_type", "object_id"], name="uniq_grant_scope")]
        indexes = [models.Index(fields=["user", "content_type"])]
```

API compatibility: the payload in USER_ROLES uses `"object_id": null` for global and `"model_name": "customercarecase"`. Map `null` to `0` and back in the serializer, and resolve `model_name` through an allowlist, never through arbitrary `ContentType` lookup.

`parent` records which grant authorized a delegated grant, so revoking the parent revokes everything derived from it (a delegate cannot outlive the person who delegated).

## 3. Evaluation and queryset scoping

```python
def covering_grants(user, ct):
    now = timezone.now()
    return GranularPermission.objects.filter(user=user, content_type=ct).filter(
        Q(expires_at__isnull=True) | Q(expires_at__gt=now))

def has_perm(user, model, action: str, obj_id: int | None = None) -> bool:
    if not user.is_authenticated or not user.is_active:
        return False
    if user.is_superuser:
        return True
    if model in ADMIN_ONLY_MODELS:
        return user.role == Role.ADMIN
    if user.role == Role.ADMIN:
        return True
    ct = ContentType.objects.get_for_model(model)
    flag = f"can_{action}"
    grants = covering_grants(user, ct)
    q = Q(object_id=GLOBAL) | (Q(object_id=obj_id) if obj_id else Q(pk__in=[]))
    return grants.filter(q, **{flag: True}).exists()

def scoped_queryset(user, model, qs, action="view"):
    """Lists must be filtered. has_object_permission alone never filters a list."""
    if user.is_superuser or (user.role == Role.ADMIN):
        return qs
    ct = ContentType.objects.get_for_model(model)
    grants = covering_grants(user, ct).filter(**{f"can_{action}": True})
    if grants.filter(object_id=GLOBAL).exists():
        return qs
    return qs.filter(pk__in=grants.exclude(object_id=GLOBAL).values("object_id"))
```

Memoize grants per request (store on `request._grants`) so a list view does not query per row. Cache across requests only with explicit invalidation on grant changes; correctness matters more than saving one indexed query.

Customers never go through this engine: their viewsets filter by `owner=request.user` (or by guest token), full stop.

## 4. Grant, revoke, and delegation service

All grant changes go through one service. The serializer and Django Admin both call it.

```python
@transaction.atomic
def grant_permission(actor, *, target, model_name, object_id, flags: dict, expires_at=None, constraints=None):
    model = resolve_delegable_model(model_name)           # allowlist; raises for unknown, ADMIN_ONLY, or sensitive models
    scope = object_id or GLOBAL
    if actor.pk == target.pk:
        raise PermissionDenied("Self-grants are not allowed.")
    if target.is_superuser or target.role in {Role.ADMIN, Role.CUSTOMER}:
        raise PermissionDenied("Grants apply to staff roles only.")

    parent = None
    if not (actor.is_superuser or actor.role == Role.ADMIN):
        if target.role != actor.role:
            raise PermissionDenied("Cannot delegate across roles.")          # STOCK cannot delegate to PACKING
        ct = ContentType.objects.get_for_model(model)
        held = best_covering_grant(actor, ct, scope)                          # global covers any object; an object grant covers only itself
        if held is None or not held.can_delegate:
            raise PermissionDenied("You do not hold delegation rights for this scope.")
        for flag, wanted in flags.items():                                    # includes can_delegate itself
            if wanted and not getattr(held, flag):
                raise PermissionDenied(f"You cannot grant {flag}: you do not hold it.")
        if scope == GLOBAL and held.object_id != GLOBAL:
            raise PermissionDenied("Cannot widen scope beyond your own grant.")
        constraints = narrow_constraints(held.constraints_json, constraints)  # delegate limits can only be equal or stricter
        expires_at = min_expiry(held.expires_at, expires_at)                  # a delegate cannot outlive the delegator's grant
        parent = held

    ct = ContentType.objects.get_for_model(model)
    before = GranularPermission.objects.select_for_update().filter(user=target, content_type=ct, object_id=scope).first()
    grant, _ = GranularPermission.objects.update_or_create(
        user=target, content_type=ct, object_id=scope,
        defaults={**flags, "granted_by": actor, "parent": parent, "expires_at": expires_at, "constraints_json": constraints or {}},
    )
    audit.record("permission.granted", actor=actor, before=serialize(before), after=serialize(grant), context={"target_user": target.pk})
    return grant
```

Rules enforced above (the three guarantees from USER_ROLES plus the closing of gaps):
- Same role only for non-admin delegators; a user cannot grant a permission they do not hold; `can_delegate=True` requires holding `can_delegate`.
- Self-grant, grants to customers, admins, and superusers are rejected.
- Delegation cannot widen scope: an object-level holder cannot create a global grant.
- Delegated grants are never broader than the parent (flags, scope, constraints, expiry).
- Optional `MAX_DELEGATION_DEPTH` (default 2) prevents unbounded chains; compute depth by walking `parent`.

Revocation: ADMIN and superusers can revoke any grant. A delegator can revoke grants derived from their own grants. Revoking a grant cascades to `derived` rows. Every revoke is audited.

## 5. DRF enforcement

```python
class GranularModelPermission(BasePermission):
    ACTION_FLAG = {"list": "view", "retrieve": "view", "create": "add",
                   "update": "change", "partial_update": "change", "destroy": "delete"}

    def has_permission(self, request, view):
        user = request.user
        if not (user and user.is_authenticated):
            return False
        flag = view.action_permission_map.get(view.action) or self.ACTION_FLAG.get(view.action)
        if flag is None:
            return False                              # unknown or unmapped custom action: deny
        return has_perm(user, view.permission_model, flag) or scoped_queryset(
            user, view.permission_model, view.permission_model.objects.all(), flag).exists()

    def has_object_permission(self, request, view, obj):
        flag = view.action_permission_map.get(view.action) or self.ACTION_FLAG.get(view.action)
        return has_perm(request.user, view.permission_model, flag, obj.pk)
```

Pattern for staff viewsets:

```python
class CareCaseViewSet(StaffViewSet):
    permission_classes = [IsAuthenticated, GranularModelPermission]
    permission_model = CareCase
    action_permission_map = {"resolve": "change", "escalate": "change"}

    def get_queryset(self):
        return scoped_queryset(self.request.user, CareCase, CareCase.objects.select_related("customer", "order"))
```

- Custom actions (`mark_as_packed`, `refund`, `resolve`) must appear in `action_permission_map`; unmapped means denied.
- Also check the role for role-specific actions, because a grant on `order` does not make an `ASSISTANT` a packer: `mark_as_packed` requires role `PACKING` (or `ADMIN`) plus `change` on the order.
- Sensitive actions add business-rule checks beyond flags: refund amount cap from `constraints_json`, refund approval threshold, and self-approval prevention (the initiator cannot approve their own refund when a threshold applies).
- `request.user` for staff endpoints must come from JWT authentication only. Do not enable session or basic authentication on staff routes.

## 6. Admin-only models and sensitive models

```python
ADMIN_ONLY_MODELS = {Product, ProductVariant, Category, Brand, Coupon, Promotion, RecommendationTag}   # from USER_ROLES (extended to variants and promotions)
NEVER_DELEGABLE = {GranularPermission, User, AuditLog, Payment, WebhookEvent, SiteConfig, NotificationTemplate}
```

- `resolve_delegable_model()` rejects both sets. The rejection happens in the service (so Django Admin cannot bypass it) and in the serializer (so the API returns a clear 400).
- `Refund`, `WarrantyClaim`, `CareCase`, `CaseMessage`, `CallRequest`, `StockMovement`, `SerialNumber`, `Order` (view, and the packing action), `Package`, and report views are delegable.
- Keep the sets in one module so the user can change them in one place. If the user decides `STOCK` may edit products, remove the entries deliberately and add tests, rather than special-casing views.

## 7. Django Admin access

- Admin users sign in through Django Admin with sessions, MFA, and optionally an IP allowlist or VPN. API staff roles do not need `is_staff`; keep `is_staff` for administrators only.
- Define groups ("Full Admin", "Catalog Admin", "Finance Admin", "Support Admin", "Report Viewer") with Django model permissions, and implement `has_*_permission` in each `ModelAdmin` from them. Ship "Full Admin" so the documented ADMIN behavior holds by default.
- Read-only admin for `AuditLog`, `Payment`, `WebhookEvent`, `StockMovement`, and `SerialEvent`: no add, change, or delete.
- Order changes in Admin call the same `transition()` service, not raw field edits, so audit and stock side effects always run. Disable bulk "delete selected" on protected models.
- Permission edits in Admin also call `grant_permission()` or write an audit row; never edit the table directly.

## 8. Permission tests (minimum)

Parametrize over roles and endpoints, and assert:
- Anonymous: 401. Customer on staff endpoints: 403. Customer reading another customer's order, case, receipt, or warranty: 404 (not 403, to avoid confirming existence).
- Default deny: a new `ASSISTANT` with no grants gets 403 on orders and cases.
- Object-level grant: a user with a grant on object 4 sees only object 4 in the list, cannot fetch object 5, and cannot delete object 4 when `can_delete` is false.
- Delegation guarantees: cross-role delegation, granting a flag you lack, granting delegation without holding it, widening scope, self-grant, granting on `ADMIN_ONLY_MODELS` and `NEVER_DELEGABLE`, and depth limit all fail with 403 or 400 and create no row.
- Cascade: revoking a parent removes derived grants; an expired grant stops working.
- Admin: cannot create or delete a superuser unless a superuser; admin grants appear in the audit log.
- Unmapped custom action on a staff viewset returns 403.
