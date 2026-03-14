# Rate Limiting Strategy

**Last Updated:** 2026-03-14
**Status:** Active

---

## Rate Limiting Engine

**Library**: `rate-sync` (mandatory via ADR)
**Backend**: Redis (production), in-memory (development)
**Algorithm**: Sliding window (recommended)

---

## Limit Definitions

### Public API

| Endpoint | Limit | Window | Dimension |
|----------|-------|--------|-----------|
| Global | 100 req/min | 60s | per tenant |
| POST /messages | 10 req/sec | 1s | per tenant |
| GET /messages/{id} | 100 req/min | 60s | per tenant |
| Messaging quota | 1000 messages/hour | 3600s | per tenant |

### Dashboard API

| Endpoint | Limit | Window | Dimension |
|----------|-------|--------|-----------|
| POST /bff/auth/login | 5 attempts/15min | 900s | per IP + email |
| POST /bff/auth/password/forgot | 3 attempts/hour | 3600s | per email |
| POST /bff/auth/refresh | 60 req/min | 60s | per user |
| POST /bff/access-keys | 10 req/hour | 3600s | per tenant |

---

## Implementation

### Configuration (rate-sync.toml)

```toml
[[limiters]]
name = "public_api_global"
algorithm = "sliding_window"
limit = 100
window_seconds = 60

[[limiters]]
name = "dashboard_login"
algorithm = "sliding_window"
limit = 5
window_seconds = 900  # 15 minutes
```

### FastAPI Integration

```python
from ratesync import RateLimiter

limiter = RateLimiter.from_config("rate-sync.toml")

async def enforce_rate_limit(
    tenant_id: str,
    limiter_name: str = "public_api_global"
):
    key = f"tenant:{tenant_id}"
    allowed = await limiter.check(limiter_name, key)
    if not allowed:
        raise HTTPException(
            status_code=429,
            detail={"error": "rate_limit_exceeded"},
            headers={"Retry-After": "60"}
        )

@router.post("/messages")
async def send_message(
    auth: AuthContext,
    _=Depends(lambda auth: enforce_rate_limit(auth.tenant_id))
):
    ...
```

### Response Headers

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1710417600
Retry-After: 45  # (on 429 only)
```

---

## Resilience and Failure Modes

### Redis Connectivity Issues

**⚠️ CRITICAL**: Rate limiting depends on Redis availability. If Redis is down, the current implementation has **NO fallback**.

| Scenario | Current Behavior | Impact |
|----------|-----------------|--------|
| **Redis down** | rate-sync raises exception | All requests fail (500 error) |
| **Redis slow** | Requests timeout | Degraded performance |
| **Redis network partition** | Connection errors | Service unavailable |

**Recommended Fallback Strategies** (choose one):

1. **Fail-open** (allow all requests):
```python
try:
    allowed = await limiter.check(limiter_name, key)
    if not allowed:
        raise HTTPException(429, "rate_limit_exceeded")
except RedisConnectionError:
    logger.error("Redis unavailable, bypassing rate limit")
    # Allow request to proceed (fail-open)
```

✅ **Pros**: Service stays available
❌ **Cons**: No rate limiting during outage (DDoS risk)

2. **Fail-closed** (reject all requests):
```python
try:
    allowed = await limiter.check(limiter_name, key)
except RedisConnectionError:
    raise HTTPException(503, "rate_limiting_unavailable")
```

✅ **Pros**: Secure (no bypass)
❌ **Cons**: Service becomes unavailable

3. **In-memory fallback** (per-instance limits):
```python
from ratelimit import limits  # fallback library

try:
    allowed = await limiter.check(limiter_name, key)
except RedisConnectionError:
    # Fall back to local in-memory rate limiting
    allowed = local_limiter.check(key)
```

✅ **Pros**: Partial protection
❌ **Cons**: Limits are per-worker (not distributed)

**Recommendation**: Implement **fail-open** for public-api (availability priority) and **fail-closed** for dashboard-api login endpoints (security priority).

### Monitoring

**Required Metrics**:
- `rate_limit_redis_errors_total` - Redis connection failures
- `rate_limit_bypassed_total` - Requests allowed due to fallback
- `rate_limit_exceeded_total` - Requests blocked by rate limiting

**Alerts**:
- Redis connection failure > 1 minute → Page on-call
- Bypass rate > 10% → Investigate Redis health

---

## Tier-Based Limits (Future)

| Plan | Messages/Day | API Calls/Day | Burst Rate |
|------|--------------|---------------|------------|
| Free | 100 | 1,000 | 1 req/sec |
| Pro | 10,000 | 100,000 | 10 req/sec |
| Enterprise | Custom | Custom | Custom |

---

## Related Documentation

- [01-authentication-architecture.md](01-authentication-architecture.md)
- [08-threat-model.md](08-threat-model.md)
