# Token Security

**Last Updated:** 2026-03-14
**Status:** Active

---

## Random Number Generation

### Cryptographically Secure RNG

```python
import secrets

# ✅ CORRECT - Cryptographically secure
api_key = f"tn_live_{secrets.token_urlsafe(24)}"  # 256 bits entropy
session_token = secrets.token_hex(32)  # 256 bits entropy
reset_token = secrets.token_urlsafe(32)

# ❌ WRONG - NOT cryptographically secure
import random
bad_token = str(random.randint(0, 999999))  # Predictable!
```

---

## Token Types and Generation

| Token Type | Generation Method | Length | Entropy |
|------------|------------------|---------|---------|
| **API Key** | `secrets.token_urlsafe(24)` | ~32 chars | 256 bits |
| **JWT** | python-jose (HS256) | Variable | N/A (signed) |
| **Reset Token** | `secrets.token_urlsafe(32)` | ~43 chars | 256 bits |
| **WhatsApp Code** | `secrets.randbelow(1000000)` | 6 digits | ~20 bits |

---

## Hashing Algorithms

| Use Case | Algorithm | Rounds | Speed |
|----------|-----------|--------|-------|
| **Passwords** | bcrypt | 12 (prod), 10 (test) | Slow (intentional) |
| **API Keys** | SHA-256 | N/A | Fast |
| **Reset Tokens** | SHA-256 | N/A | Fast |
| **WhatsApp Codes** | SHA-256 | N/A | Fast |

### Implementation

```python
import hashlib
import bcrypt

# API Key hashing (SHA-256)
def hash_api_key(key: str) -> str:
    return hashlib.sha256(key.encode()).hexdigest()

# Password hashing (bcrypt)
def hash_password(password: str, rounds: int = 12) -> str:
    salt = bcrypt.gensalt(rounds)
    return bcrypt.hashpw(password.encode(), salt).decode()

def verify_password(password: str, hash: str) -> bool:
    return bcrypt.checkpw(password.encode(), hash.encode())
```

---

## Token Storage

### Database Storage

```python
# ✅ CORRECT - Store hashes only
api_key_hash = hashlib.sha256(plain_key.encode()).hexdigest()
await db.execute(
    "INSERT INTO api_keys (key_hash, ...) VALUES (?, ...)",
    api_key_hash
)

# ❌ WRONG - Never store plaintext
await db.execute(
    "INSERT INTO api_keys (plaintext_key, ...) VALUES (?, ...)",  # SECURITY RISK!
    plain_key
)
```

### Redis Token Blacklist (Future)

```python
# Store revoked JTIs with TTL
await redis.setex(f"revoked:{jti}", ttl=604800, value="1")  # 7 days

# Check revocation
if await redis.exists(f"revoked:{jti}"):
    raise HTTPException(401, "token_revoked")
```

---

## Token Transmission

### HTTPS Enforcement

```python
# Production: Force HTTPS via HSTS header
response.headers["Strict-Transport-Security"] = "max-age=31536000; includeSubDomains"

# Development: Allow HTTP (localhost only)
if settings.environment != "production":
    # HTTPS not enforced
    pass
```

### Authorization Header

```
Authorization: Bearer tn_live_aB3xK9pQ2mN5vC8wE1rT4yU7iO0pLkJhGfDsAqWeRtYuIo
```

### Cookie Security (for JWT)

```python
response.set_cookie(
    key="access_token",
    value=token,
    httponly=True,  # Prevents XSS
    secure=True,    # HTTPS only
    samesite="lax", # CSRF protection
    max_age=900     # 15 minutes
)
```

---

## Token Rotation

| Token Type | Rotation Strategy |
|------------|-------------------|
| **API Keys** | Manual (recommended: 90 days) |
| **Refresh Tokens** | Automatic on every use (future) |
| **Password Reset** | One-time use, expires 1 hour |
| **WhatsApp Codes** | One-time use, expires 5 minutes |

---

## Token Lifecycle

```
Generation → Storage → Validation → Expiration → Revocation
     ↓          ↓          ↓             ↓            ↓
  secrets   SHA-256     Lookup      Time Check   Set Flag
```

---

## Security Checklist

- [ ] Use `secrets` module for random generation
- [ ] Hash tokens before database storage (SHA-256 or bcrypt)
- [ ] Never log plaintext tokens
- [ ] Enforce HTTPS in production (HSTS header)
- [ ] Set expiration timestamps
- [ ] Support revocation (flag or blacklist)
- [ ] Use HttpOnly cookies for browser tokens
- [ ] Rotate tokens periodically

---

## Related Documentation

- [02-api-key-system.md](02-api-key-system.md)
- [03-jwt-session-management.md](03-jwt-session-management.md)
- [09-security-headers.md](09-security-headers.md)
