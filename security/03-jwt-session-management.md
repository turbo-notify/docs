# JWT Session Management

**Last Updated:** 2026-03-14
**Status:** Active

---

## Token Architecture

### Dual Token Strategy

| Token | Expiry | Purpose | Claims |
|-------|--------|---------|--------|
| **Access Token** | 15min | API requests | `sub`, `tenant_id`, `typ:"access"`, `exp`, `iat` |
| **Refresh Token** | 7 days | Obtain new access token | `sub`, `tenant_id`, `jti`, `typ:"refresh"`, `exp`, `iat` |

### Token Generation

```python
from jose import jwt
from datetime import datetime, timedelta, UTC
from uuid import uuid4

def create_token_pair(user_id: str, tenant_id: str) -> tuple[str, str]:
    """Generate access + refresh token pair."""
    now = datetime.now(UTC)

    # Access token (15 min)
    access_payload = {
        "sub": user_id,
        "tenant_id": tenant_id,
        "typ": "access",
        "exp": now + timedelta(minutes=15),
        "iat": now
    }
    access_token = jwt.encode(access_payload, settings.jwt_secret, algorithm="HS256")

    # Refresh token (7 days)
    jti = str(uuid4())
    refresh_payload = {
        "sub": user_id,
        "tenant_id": tenant_id,
        "jti": jti,
        "typ": "refresh",
        "exp": now + timedelta(days=7),
        "iat": now
    }
    refresh_token = jwt.encode(refresh_payload, settings.jwt_secret, algorithm="HS256")

    return access_token, refresh_token, jti
```

---

## Session Management

### Database Schema

```sql
CREATE TABLE auth_sessions (
  id UUID PRIMARY KEY,
  user_id UUID NOT NULL REFERENCES users(id),
  tenant_id UUID NOT NULL REFERENCES tenants(id),
  access_token_jti UUID,
  refresh_token_jti UUID UNIQUE,
  access_token_expires_at TIMESTAMP,
  refresh_token_expires_at TIMESTAMP,
  status VARCHAR(20) CHECK (status IN ('active', 'revoked', 'expired')),
  ip_address VARCHAR(45),
  user_agent TEXT,
  created_at TIMESTAMP DEFAULT NOW(),
  last_activity_at TIMESTAMP,
  revoked_at TIMESTAMP,
  revoked_reason TEXT
);

CREATE INDEX idx_sessions_refresh_jti ON auth_sessions(refresh_token_jti);
CREATE INDEX idx_sessions_user ON auth_sessions(user_id, status);
```

### Token Validation Dependency

```python
from fastapi import Depends, HTTPException
from fastapi.security import HTTPBearer

security = HTTPBearer()

async def get_current_user(
    credentials: HTTPAuthorizationCredentials = Depends(security),
    db: AsyncSession = Depends(get_db)
) -> CurrentUser:
    """Validate access token and return current user."""
    token = credentials.credentials

    try:
        payload = jwt.decode(token, settings.jwt_secret, algorithms=["HS256"])

        # Check token type
        if payload.get("typ") != "access":
            raise HTTPException(401, "invalid_token_type")

        user_id = payload.get("sub")
        tenant_id = payload.get("tenant_id")

        # Load user from DB
        user = await user_repo.find_by_id(user_id)
        if not user or user.status != "active":
            raise HTTPException(401, "user_inactive")

        return CurrentUser(user_id=user_id, tenant_id=tenant_id)

    except JWTError:
        raise HTTPException(401, "invalid_token")
```

---

## Refresh Flow

### Implementation

```python
@router.post("/bff/auth/refresh")
async def refresh_token(
    body: RefreshTokenRequest,
    db: AsyncSession = Depends(get_db)
) -> TokenResponse:
    """Refresh access token using refresh token."""

    # Validate refresh token
    try:
        payload = jwt.decode(body.refresh_token, settings.jwt_secret, algorithms=["HS256"])
        if payload.get("typ") != "refresh":
            raise HTTPException(401, "invalid_token_type")
        jti = payload.get("jti")
        user_id = payload.get("sub")
        tenant_id = payload.get("tenant_id")
    except JWTError:
        raise HTTPException(401, "invalid_token")

    # Lookup session by JTI
    session = await session_repo.find_by_refresh_jti(jti)
    if not session or session.status != "active":
        raise HTTPException(401, "session_invalid")

    # Generate new token pair
    new_access, new_refresh, new_jti = create_token_pair(user_id, tenant_id)

    # Update session
    session.access_token_jti = str(uuid4())
    session.refresh_token_jti = new_jti
    session.last_activity_at = datetime.now(UTC)
    await session_repo.save(session)

    return TokenResponse(access_token=new_access, refresh_token=new_refresh)
```

**⚠️ SECURITY WARNING**: The current implementation does NOT detect refresh token reuse. If a refresh token is stolen, an attacker can use it indefinitely within the 7-day expiration window. This is a **known security gap**.

**Mitigations** (choose one):
1. **Implement token families** (recommended) - Detect and revoke on reuse (see roadmap below)
2. **Reduce refresh token TTL** - Change from 7 days to 24 hours (reduces exposure window)
3. **Add device fingerprinting** - Detect suspicious device changes

See [08-threat-model.md](08-threat-model.md#t2-jwt-token-theft) for complete threat analysis.

---

## Security Enhancements (Roadmap)

### Token Families (Recommended)

Detect refresh token reuse attacks:

```python
# On token creation, track family
session.token_family_id = str(uuid4())

# On refresh, mark previous as used
old_session = await find_by_refresh_jti(jti)
old_session.status = "refreshed"
old_session.replaced_by_id = new_session.id

# If reuse detected (already refreshed)
if old_session.status == "refreshed":
    # Revoke entire family
    await revoke_token_family(old_session.token_family_id)
    raise HTTPException(401, "token_reuse_detected")
```

### Redis Token Blacklist

Fast revocation checking:

```python
# On revocation
await redis.setex(f"revoked:{jti}", ttl=604800, value="1")  # 7 days

# On validation
if await redis.exists(f"revoked:{jti}"):
    raise HTTPException(401, "token_revoked")
```

---

## Related Documentation

- [01-authentication-architecture.md](01-authentication-architecture.md)
- [07-token-security.md](07-token-security.md)
- [06-audit-logging.md](06-audit-logging.md)
