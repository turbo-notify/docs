# HTTP Security Headers

**Last Updated:** 2026-03-14
**Status:** Active

---

## Required Security Headers

| Header | Value | Purpose |
|--------|-------|---------|
| `Strict-Transport-Security` | `max-age=31536000; includeSubDomains` | Force HTTPS (production only) |
| `X-Content-Type-Options` | `nosniff` | Prevent MIME type sniffing |
| `X-Frame-Options` | `DENY` | Prevent clickjacking |
| `X-XSS-Protection` | `1; mode=block` | Enable XSS filter (legacy browsers) |
| `Content-Security-Policy` | `default-src 'none'` | Restrict resource loading |
| `Referrer-Policy` | `strict-origin-when-cross-origin` | Control Referer header |
| `Cache-Control` | `no-store` | Prevent caching sensitive data |

---

## CORS Configuration

### Dashboard API (BFF)

Allows requests from frontend:

```python
from fastapi.middleware.cors import CORSMiddleware

app.add_middleware(
    CORSMiddleware,
    allow_origins=["https://dashboard.turbonotify.com"],
    allow_credentials=True,
    allow_methods=["GET", "POST", "PUT", "DELETE", "PATCH"],
    allow_headers=["*"],
    expose_headers=["X-RateLimit-Limit", "X-RateLimit-Remaining", "X-RateLimit-Reset"]
)
```

### Public API (Control Plane)

NO CORS needed (server-to-server):

```python
# No CORS middleware
# Clients call API from backend, not browser
```

---

## Security Headers Middleware

### FastAPI Implementation

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.requests import Request
from starlette.responses import Response

class SecurityHeadersMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response: Response = await call_next(request)

        # Security headers
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"
        response.headers["Cache-Control"] = "no-store"
        response.headers["Referrer-Policy"] = "strict-origin-when-cross-origin"

        # CSP: Relaxed for docs, strict for API
        if request.url.path in ["/docs", "/redoc", "/openapi.json"]:
            response.headers["Content-Security-Policy"] = (
                "default-src 'self'; "
                "script-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
                "style-src 'self' 'unsafe-inline' https://cdn.jsdelivr.net; "
                "img-src 'self' data: https://fastapi.tiangolo.com; "
                "font-src 'self' https://cdn.jsdelivr.net"
            )
        else:
            response.headers["Content-Security-Policy"] = "default-src 'none'"

        # HSTS: Production only (requires HTTPS)
        if settings.environment == "production":
            response.headers["Strict-Transport-Security"] = (
                "max-age=31536000; includeSubDomains"
            )

        return response


# Apply middleware
app.add_middleware(SecurityHeadersMiddleware)
```

---

## Rate Limit Headers

Custom headers for rate limiting feedback:

```
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 87
X-RateLimit-Reset: 1710417600
```

On 429 (Too Many Requests):

```
Retry-After: 45
```

---

## Internal Headers

Non-standard headers for internal tracking:

```
X-Request-ID: 880e8400-e29b-41d4-a716-446655440003
X-Tenant-ID: 550e8400-e29b-41d4-a716-446655440000
```

**Note**: Do NOT expose tenant_id to external clients.

---

## Nginx/Proxy Configuration (Optional)

If using reverse proxy, headers can be set at proxy level:

```nginx
# HSTS
add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

# Security headers
add_header X-Content-Type-Options "nosniff" always;
add_header X-Frame-Options "DENY" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;

# CSP (strict)
add_header Content-Security-Policy "default-src 'none'" always;

# Cache control
add_header Cache-Control "no-store" always;
```

---

## Testing Security Headers

### securityheaders.com

Test your deployed API:

```bash
curl -I https://api.turbonotify.com
```

Check for all required headers.

### Automated Testing

```python
import pytest
from fastapi.testclient import TestClient

def test_security_headers(client: TestClient):
    response = client.get("/messages")

    assert response.headers["X-Content-Type-Options"] == "nosniff"
    assert response.headers["X-Frame-Options"] == "DENY"
    assert response.headers["X-XSS-Protection"] == "1; mode=block"
    assert "Content-Security-Policy" in response.headers

    # HSTS only in production
    if settings.environment == "production":
        assert "Strict-Transport-Security" in response.headers
```

---

## Related Documentation

- [01-authentication-architecture.md](01-authentication-architecture.md)
- [07-token-security.md](07-token-security.md)
- [08-threat-model.md](08-threat-model.md)
