# Authorization Model

**Last Updated:** 2026-03-14
**Status:** Active

---

## ⚠️ CRITICAL: Entity Alignment Issue

**Current Blocker**: The codebase has two different API Key entities that are NOT aligned:

| Aspect | `ApiKey` (public-api) | `AccessKey` (dashboard-api) |
|--------|----------------------|---------------------------|
| **Location** | `public-api/src/domain/entities/api_key.py` | `dashboard-api/src/domain/entities/access_key.py` |
| **Scopes Field** | ✅ `scopes: list[str]` | ❌ Missing |
| **Validation Method** | ✅ `has_scope(scope: str)` | ❌ Missing |
| **Revocation** | `revoked_at: datetime \| None` | `is_active: bool` |

**Impact**: Documentation describes scope enforcement, but `AccessKey` entity cannot implement it without adding the `scopes` field.

**Resolution Options**:
1. **Add scopes to AccessKey** (recommended) - Align entities
2. **Merge entities** - Use shared entity from shared-core
3. **Document divergence** - Accept different capabilities

See implementation roadmap in [README.md](README.md#implementation-status).

---

## Tenant Isolation (CRITICAL)

### Golden Rule

**EVERY database query MUST filter by `tenant_id`**

This prevents cross-tenant data leakage and is THE most critical security requirement.

### Correct Pattern

```python
# ✅ CORRECT - Always include tenant_id filter
async def find_by_id(self, session_id: str, tenant_id: str) -> Session | None:
    result = await self._db.execute(
        select(SessionModel)
        .where(SessionModel.id == session_id)
        .where(SessionModel.tenant_id == tenant_id)  # MANDATORY
    )
    return result.scalar_one_or_none()
```

### WRONG Pattern (Security Vulnerability!)

```python
# ❌ WRONG - Missing tenant_id filter
async def find_by_id(self, session_id: str) -> Session | None:
    result = await self._db.execute(
        select(SessionModel).where(SessionModel.id == session_id)
    )
    return result.scalar_one_or_none()  # Can access any tenant's data!
```

### Code Review Checklist

- [ ] All repository methods accept `tenant_id` parameter
- [ ] All SQL queries include `WHERE tenant_id = ?`
- [ ] No raw SQL without tenant filtering
- [ ] Integration tests verify tenant isolation

---

## API Key Scopes

### Format

```
resource:action

Examples:
- messages:send
- messages:read
- sessions:read
- sessions:write
- numbers:write
- webhooks:read
```

### Scope Enforcement

```python
@router.post("/messages", dependencies=[Depends(require_scope("messages:send"))])
async def send_message(auth: AuthContext, body: SendMessageRequest):
    # Only keys with messages:send scope allowed
    ...

def require_scope(scope: str):
    def dependency(auth: AuthContext):
        if not auth.has_scope(scope):
            raise HTTPException(403, {"error": "insufficient_scope", "required": scope})
        return auth
    return Depends(dependency)
```

### Default Behavior

- **Empty scopes `[]`** → Full access (backward compatibility)
- **Non-empty scopes** → Restricted to listed permissions only

---

## Multi-Tenancy Security

### Tenant Context Extraction

```python
# From JWT (dashboard-api)
payload = jwt.decode(token, ...)
tenant_id = payload["tenant_id"]

# From API Key (public-api)
api_key = await repo.find_by_hash(key_hash)
tenant_id = api_key.tenant_id

# Set in request context
request.state.tenant_id = tenant_id
```

### Dependency Injection Pattern

```python
TenantId = Annotated[str, Depends(get_tenant_id)]

@router.get("/sessions")
async def list_sessions(tenant_id: TenantId):
    # tenant_id automatically extracted and validated
    sessions = await repo.find_by_tenant(tenant_id)
    return sessions
```

---

## Role-Based Access Control (Future)

### Role Hierarchy

| Role | Permissions |
|------|-------------|
| **OWNER** | All permissions, manage billing |
| **ADMIN** | Manage team, API keys, webhooks |
| **MEMBER** | Use API, view analytics |
| **VIEWER** | Read-only access |

### Implementation Pattern

```python
def require_role(role: str):
    async def dependency(user: CurrentUser, db: AsyncSession):
        membership = await get_tenant_membership(user.id, user.tenant_id, db)
        if not membership.has_role(role):
            raise HTTPException(403, "insufficient_permissions")
        return user
    return Depends(dependency)

@router.post("/access-keys", dependencies=[Depends(require_role("ADMIN"))])
async def create_api_key(...):
    ...
```

---

## Related Documentation

- [01-authentication-architecture.md](01-authentication-architecture.md)
- [02-api-key-system.md](02-api-key-system.md)
- [08-threat-model.md](08-threat-model.md)
