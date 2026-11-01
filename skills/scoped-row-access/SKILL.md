---
name: scoped-row-access
description: Use when a backend must limit which rows each caller sees or changes by role or identity (admin sees all, user sees own, API client sees its tenant), when adding ownership checks, multi-tenant filtering, or RBAC row visibility to layered API → service → repository code, or when reviewing for IDOR / broken object-level authorization.
---

# Scoped Row Access

## Overview
Translate "who is calling" into one immutable **scope value** at the API edge, pass it unchanged through the service, and turn it into a SQL predicate in **exactly one** repository function. Ownership is never checked by fetch-then-compare.

```
 HTTP request
     │  auth dependency → principal (user / API key)
     ▼
 API layer      scope = scope_for_principal(principal)     ← role → scope, ONLY here
     │          (optional admin narrowing: owner(user_id))
     ▼
 Service        passes scope through; trusted internal paths use Scope.all()
     │          row is None → NotFound (never 403: don't leak existence)
     ▼
 Repository     stmt = apply_scope(stmt, scope, Model.tenant_col)  ← ONLY place scope becomes SQL
     ▼
 DB             WHERE tenant_col IN (SELECT id FROM tenants WHERE owner = :uid)
```

## When to Use
- Roles differ in *which rows* they see, not only *which endpoints* they hit.
- Several caller types (human users, admins, machine clients) hit the same tables.
- Not for endpoint-level gating alone: that is role checks (`require_roles`). Use both: role check decides *may call*, scope decides *which rows*.

## Core Pattern

**1. Scope value (core/domain, no framework imports)**
```python
class ScopeKind(StrEnum):
    ALL = "all"            # admins, trusted internal lookups
    OWNER = "owner"        # rows of tenants owned by one user
    TENANT = "tenant"      # one tenant (a machine client's own)

_REQUIRED_FIELD = {ScopeKind.OWNER: "owner_user_id", ScopeKind.TENANT: "tenant_id"}

@dataclass(frozen=True)
class AccessScope:
    kind: ScopeKind
    owner_user_id: uuid.UUID | None = None
    tenant_id: uuid.UUID | None = None

    def __post_init__(self) -> None:
        # Invalid scope can't exist: OWNER without a user id fails at construction.
        required = _REQUIRED_FIELD.get(self.kind)
        if required is not None and getattr(self, required) is None:
            raise ValueError(f"{self.kind} scope requires {required}")

    @classmethod
    def all(cls): return cls(ScopeKind.ALL)
    @classmethod
    def owner(cls, user_id): return cls(ScopeKind.OWNER, owner_user_id=user_id)
    @classmethod
    def tenant(cls, tenant_id): return cls(ScopeKind.TENANT, tenant_id=tenant_id)
```

**2. Principal → scope, one mapper per caller type (auth layer)**
```python
def scope_for_principal(p: Principal) -> AccessScope:
    if p.is_admin():
        return AccessScope.all()
    return AccessScope.owner(p.user_id)

def scope_for_api_key(k: AuthenticatedKey) -> AccessScope:
    # From the authenticated credential, never from request params.
    return AccessScope.tenant(k.tenant_id)
```

**3. Scope → SQL, single function (repository layer)**
```python
def apply_scope(stmt, scope: AccessScope, tenant_col):
    match scope.kind:
        case ScopeKind.ALL:
            return stmt
        case ScopeKind.OWNER:
            # Subquery, not join: caller may already join tenants.
            owned = select(Tenant.id).where(Tenant.user_id == scope.owner_user_id)
            return stmt.where(tenant_col.in_(owned))
        case ScopeKind.TENANT:
            return stmt.where(tenant_col == scope.tenant_id)
        case _:
            raise ValueError(f"unknown scope kind: {scope.kind}")  # fail closed
```

**4. Every repo read takes scope; writes load through a scoped get**
```python
# repo
async def get(session, scope, tenant_id, lock=RowLock.NONE):
    q = apply_scope(select(Tenant).where(Tenant.id == tenant_id), scope, Tenant.id)
    ...
# service
async def delete_tenant(scope, tenant_id):
    tenant = await tenants_repo.get(session, scope, tenant_id)
    if tenant is None:
        raise NotFoundError("tenant", tenant_id)   # missing OR not yours: same answer
    await tenants_repo.delete(session, tenant)
```

**5. Admin narrowing at the API edge only**
```python
scope = scope_for_principal(principal)
if principal.is_admin() and user_id is not None:
    scope = AccessScope.owner(user_id)     # users ignore ?user_id: already pinned
```

## Quick Reference

| Layer | Owns | Must not |
|---|---|---|
| Core | `AccessScope`, `ScopeKind` | import ORM / web framework |
| Auth | principal → scope mappers | read scope ids from request body/query |
| API | build scope, admin narrowing | build SQL, compare owner ids |
| Service | pass scope, map `None` → 404, use `all()` for trusted internal lookups | fetch then `if row.owner != user` |
| Repo | `apply_scope` on every scoped query | accept raw `user_id` for ownership |

Child tables: pass the column holding the tenant id (`Delta.tenant_id`), so one `apply_scope` covers all tables.

## Common Mistakes
- **Fetch-then-compare in service** → forgotten on one endpoint = IDOR. Predicate in query instead.
- **403 for others' rows** → leaks existence. Return 404.
- **`case _: return stmt`** → new kind silently unfiltered. Raise.
- **Scope from request params** (`?tenant_id=`) for machine clients → derive from credential.
- **Boolean `is_admin` / `user_id=None` meaning "all"** → `None` by accident = full access. Use explicit `ALL`.
- **Join to owner table inside `apply_scope`** → duplicate-join errors when caller already joined. Use `IN (subquery)`.
- **Writes without scoped load** → load the row via scoped `get` (with `FOR UPDATE` if needed) before mutating.

## Tests to Write
- Unit: each kind constructs; missing required id raises; mapper returns `ALL` for admin, `OWNER` for user.
- Integration per scoped endpoint: user A cannot get/list/update/delete user B's row (expect 404 / absent from list); admin sees both.
