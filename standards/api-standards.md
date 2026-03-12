# API Standards

> REST API conventions for Turbo Notify.

---

## URL Structure

### Base URL

```
https://api.turbonotify.com/v1
```

### Resource Naming

- Use plural nouns: `/sessions`, `/messages`
- Use kebab-case for multi-word: `/webhook-deliveries`
- Nest related resources: `/sessions/{id}/messages`

### Examples

```
GET    /v1/sessions                 # List sessions
POST   /v1/sessions                 # Create session
GET    /v1/sessions/{id}            # Get session
PUT    /v1/sessions/{id}            # Update session
DELETE /v1/sessions/{id}            # Delete session
POST   /v1/sessions/{id}/reconnect  # Action on session
```

---

## HTTP Methods

| Method | Usage | Idempotent |
|--------|-------|------------|
| GET | Read resources | Yes |
| POST | Create resource or action | No |
| PUT | Replace resource | Yes |
| PATCH | Partial update | No |
| DELETE | Remove resource | Yes |

---

## Request Format

### Headers

```http
Content-Type: application/json
Authorization: Bearer <token>
X-Request-ID: <uuid>
X-Idempotency-Key: <key>  # For POST requests
```

### Body

Use camelCase for JSON keys:

```json
{
  "phoneNumber": "+5511999999999",
  "webhookUrl": "https://example.com/webhook",
  "settings": {
    "autoReconnect": true
  }
}
```

---

## Response Format

### Success Response

```json
{
  "data": {
    "id": "sess_abc123",
    "phoneNumber": "+5511999999999",
    "status": "hot",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

### List Response

```json
{
  "data": [
    { "id": "sess_abc123", ... },
    { "id": "sess_def456", ... }
  ],
  "pagination": {
    "total": 50,
    "limit": 20,
    "offset": 0,
    "hasMore": true
  }
}
```

### Error Response

```json
{
  "error": {
    "code": "SESSION_NOT_FOUND",
    "message": "Session with id 'sess_xyz' was not found",
    "details": {
      "sessionId": "sess_xyz"
    }
  }
}
```

---

## Status Codes

### Success

| Code | Usage |
|------|-------|
| 200 | Successful GET, PUT, PATCH |
| 201 | Successful POST (created) |
| 202 | Accepted (async operation) |
| 204 | Successful DELETE |

### Client Errors

| Code | Usage |
|------|-------|
| 400 | Bad request (validation) |
| 401 | Unauthorized |
| 403 | Forbidden |
| 404 | Not found |
| 409 | Conflict |
| 422 | Unprocessable entity |
| 429 | Rate limited |

### Server Errors

| Code | Usage |
|------|-------|
| 500 | Internal server error |
| 502 | Bad gateway |
| 503 | Service unavailable |

---

## Error Codes

Use uppercase snake_case:

```
VALIDATION_ERROR
SESSION_NOT_FOUND
SESSION_ALREADY_EXISTS
SESSION_NOT_CONNECTED
MESSAGE_SEND_FAILED
WEBHOOK_DELIVERY_FAILED
RATE_LIMIT_EXCEEDED
INTERNAL_ERROR
```

---

## Pagination

Use offset-based pagination:

```http
GET /v1/sessions?limit=20&offset=40
```

Response includes pagination info:

```json
{
  "data": [...],
  "pagination": {
    "total": 100,
    "limit": 20,
    "offset": 40,
    "hasMore": true
  }
}
```

---

## Filtering

Use query parameters:

```http
GET /v1/sessions?status=hot&createdAfter=2024-01-01
GET /v1/messages?sessionId=sess_abc123&type=text
```

---

## Sorting

Use `sort` parameter with `+` (asc) or `-` (desc):

```http
GET /v1/sessions?sort=-createdAt
GET /v1/messages?sort=+sentAt
```

---

## Field Selection

Use `fields` parameter:

```http
GET /v1/sessions?fields=id,status,phoneNumber
```

---

## Versioning

- Version in URL path: `/v1/`, `/v2/`
- Major version for breaking changes
- Support previous version for 6 months

---

## Rate Limiting

Headers in response:

```http
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 950
X-RateLimit-Reset: 1705320000
```

When exceeded:

```json
{
  "error": {
    "code": "RATE_LIMIT_EXCEEDED",
    "message": "Rate limit exceeded. Retry after 60 seconds",
    "details": {
      "retryAfter": 60
    }
  }
}
```

---

## Idempotency

For POST requests, use `X-Idempotency-Key` header:

```http
POST /v1/messages
X-Idempotency-Key: order_123_notification
```

Server returns same response for duplicate requests with same key.

---

## Webhooks

### Signature

Include HMAC signature in header:

```http
X-Webhook-Signature: sha256=<hmac>
```

### Payload

```json
{
  "event": "message.received",
  "timestamp": "2024-01-15T10:30:00Z",
  "data": {
    "messageId": "msg_abc123",
    "from": "+5511888888888",
    "content": "Hello!"
  }
}
```

### Verification

```python
import hmac
import hashlib

def verify_webhook(payload: bytes, signature: str, secret: str) -> bool:
    expected = hmac.new(
        secret.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(f"sha256={expected}", signature)
```

---

## Related Documentation

- [NATS Events](../architecture/nats-events.md) - Internal event contracts
- [Glossary](../reference/glossary.md) - Terminology
