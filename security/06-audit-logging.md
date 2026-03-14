# Security Audit Logging

**Last Updated:** 2026-03-14
**Status:** Documented (Not Implemented)

---

## ⚠️ IMPLEMENTATION STATUS

**Current State**: This audit logging system is **fully documented but NOT yet implemented** in the codebase.

**Implementation Checklist**:
- [ ] Create `audit_logs` table via Alembic migration (see schema below)
- [ ] Create SQLAlchemy model `AuditLogModel`
- [ ] Implement `AuditLogger` service class
- [ ] Integrate into use cases (login, API key creation, etc.)
- [ ] Add audit middleware for automatic correlation_id tracking
- [ ] Configure retention policy (1 year)

**Priority**: High - Required for compliance (GDPR, SOC 2)

See [README.md](README.md#implementation-status) for roadmap.

---

## Audit Event Types

### Authentication Events

- `auth.login.success` - Successful login
- `auth.login.failure` - Failed login attempt
- `auth.logout` - User logout
- `auth.token.refresh` - Refresh token used
- `auth.password.reset.requested` - Password reset requested
- `auth.password.reset.completed` - Password successfully reset

### Authorization Events

- `api_key.created` - API key created
- `api_key.revoked` - API key revoked
- `api_key.used` - API key used (sampled: 1%)
- `session.created` - Session created
- `session.terminated` - Session ended

### Resource Events (High-Value)

- `message.sent` - Message sent (sampled: 1%)
- `webhook.configured` - Webhook endpoint changed
- `tenant.member.added` - Team member invited

---

## Database Schema

```sql
CREATE TABLE audit_logs (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  event_type VARCHAR(100) NOT NULL,
  tenant_id UUID REFERENCES tenants(id),
  actor_type VARCHAR(20) CHECK (actor_type IN ('user', 'api_key', 'system')),
  actor_id UUID,
  actor_name TEXT,
  resource_type VARCHAR(50),
  resource_id UUID,
  action VARCHAR(50),
  result VARCHAR(20) CHECK (result IN ('success', 'failure')),
  metadata JSONB DEFAULT '{}',
  ip_address VARCHAR(45),
  user_agent TEXT,
  timestamp TIMESTAMP NOT NULL DEFAULT NOW(),
  correlation_id UUID  -- Request ID for tracing
);

CREATE INDEX idx_audit_tenant_time ON audit_logs(tenant_id, timestamp DESC);
CREATE INDEX idx_audit_event_type ON audit_logs(event_type, timestamp DESC);
CREATE INDEX idx_audit_actor ON audit_logs(actor_id, timestamp DESC);
CREATE INDEX idx_audit_correlation ON audit_logs(correlation_id);
```

**⚠️ Note**: This table does NOT exist in the current database. Create it using:

```bash
# Generate migration
poetry run alembic revision -m "create_audit_logs_table"

# Copy SQL schema above into upgrade() function
# Run migration
poetry run alembic upgrade head
```

---

## Structured Logging Example

```json
{
  "event_type": "auth.login.success",
  "tenant_id": "550e8400-e29b-41d4-a716-446655440000",
  "actor_type": "user",
  "actor_id": "660e8400-e29b-41d4-a716-446655440001",
  "actor_name": "user@example.com",
  "action": "login",
  "result": "success",
  "metadata": {
    "auth_method": "email_password",
    "session_id": "770e8400-e29b-41d4-a716-446655440002"
  },
  "ip_address": "203.0.113.42",
  "user_agent": "Mozilla/5.0...",
  "timestamp": "2026-03-14T10:30:00Z",
  "correlation_id": "880e8400-e29b-41d4-a716-446655440003"
}
```

---

## Implementation

### Audit Logger Service

```python
class AuditLogger:
    def __init__(self, db: AsyncSession):
        self._db = db

    async def log_event(
        self,
        event_type: str,
        tenant_id: str | None,
        actor_type: str,
        actor_id: str | None,
        result: str,
        metadata: dict | None = None,
        ip_address: str | None = None,
        correlation_id: str | None = None
    ):
        audit_log = AuditLog(
            event_type=event_type,
            tenant_id=tenant_id,
            actor_type=actor_type,
            actor_id=actor_id,
            result=result,
            metadata=metadata or {},
            ip_address=ip_address,
            correlation_id=correlation_id
        )
        self._db.add(audit_log)
        await self._db.commit()

    async def log_login_success(self, user_id: str, tenant_id: str, ip: str):
        await self.log_event(
            event_type="auth.login.success",
            tenant_id=tenant_id,
            actor_type="user",
            actor_id=user_id,
            result="success",
            ip_address=ip
        )

    async def log_login_failure(self, email: str, ip: str, reason: str):
        # Hash email for privacy
        email_hash = hashlib.sha256(email.encode()).hexdigest()[:8]
        await self.log_event(
            event_type="auth.login.failure",
            tenant_id=None,
            actor_type="user",
            actor_id=None,
            result="failure",
            metadata={"email_hash": email_hash, "reason": reason},
            ip_address=ip
        )
```

### Use Case Integration

```python
class LoginUseCase:
    def __init__(self, audit: AuditLogger):
        self._audit = audit

    async def execute(self, email: str, password: str, ip: str):
        user = await self._user_repo.find_by_email(email)
        if not user:
            await self._audit.log_login_failure(email, ip, "invalid_credentials")
            raise InvalidCredentialsError()

        # ... authenticate ...

        await self._audit.log_login_success(user.id, user.tenant_id, ip)
        return session
```

---

## Sensitive Data Masking

| Data Type | Masking Strategy |
|-----------|------------------|
| **Email** | SHA-256 hash first 8 chars: `abc12345@...` |
| **IP Address** | Store full (compliance requirement) |
| **Password** | NEVER log |
| **API Keys** | Log only `key_prefix` (first 12 chars) |

---

## Common Queries

### Detect Brute Force

```sql
-- Failed login attempts in last hour
SELECT COUNT(*), ip_address, metadata->>'email_hash'
FROM audit_logs
WHERE event_type = 'auth.login.failure'
  AND timestamp > NOW() - INTERVAL '1 hour'
GROUP BY ip_address, metadata->>'email_hash'
HAVING COUNT(*) >= 5;
```

### API Key Usage

```sql
-- API key activity
SELECT resource_id, COUNT(*), MAX(timestamp) as last_used
FROM audit_logs
WHERE event_type = 'api_key.used'
  AND tenant_id = ?
GROUP BY resource_id;
```

### Security Incident Investigation

```sql
-- All events for correlation_id
SELECT *
FROM audit_logs
WHERE correlation_id = ?
ORDER BY timestamp;
```

---

## Retention and Compliance

- **Retention**: 1 year (365 days)
- **Immutability**: Append-only (no updates/deletes)
- **GDPR**: Logs can be anonymized on user deletion (replace `actor_id`, keep `event_type`)

---

## Related Documentation

- [01-authentication-architecture.md](01-authentication-architecture.md)
- [02-api-key-system.md](02-api-key-system.md)
- [08-threat-model.md](08-threat-model.md)
