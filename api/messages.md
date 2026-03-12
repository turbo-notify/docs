# Messages API

> Message send and status contract.

---

## Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/messages` | Send a WhatsApp message |
| `GET` | `/messages/{messageID}/status` | Get processing/delivery status |

---

## Send Message

### Request

```http
POST /messages
Content-Type: application/json
Authorization: Bearer <access_key>
```

```json
{
  "to": "+5511999999999",
  "text": "Olá, esta é uma mensagem enviada pelo Turbo Notify!",
  "from": "financeiro"
}
```

### Fields

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `to` | string | Yes | Recipient phone number |
| `text` | string | Yes | Message text |
| `from` | string | No | Extra number phone or alias |

### Success

```http
202 Accepted
```

```json
{
  "message_id": "msg_abc123",
  "operation_id": "op_msg_001",
  "status": {
    "state": "pending",
    "at": "2025-07-16T10:00:00Z"
  }
}
```

Final result is delivered asynchronously via webhook events:

- `message.sent`
- `message.delivered`
- `message.read`
- `message.failed`

### Errors

| Status | Error | Meaning |
|--------|-------|---------|
| `400` | `invalid_json` | Invalid payload |
| `403` | `not_authorized` | Missing/invalid access key |
| `422` | `sender_not_authorized` | `from` is not active/allowed |
| `429` | `rate_limit_exceeded` | Tenant/tier/internal limits exceeded |
| `500` | `internal_error` | Unexpected internal error |
| `503` | `whatsapp_unavailable` | Temporary WhatsApp unavailability |

---

## Message Status

### Request

```http
GET /messages/{messageID}/status
Authorization: Bearer <access_key>
```

### Success

```http
200 OK
```

```json
{
  "state": "sent",
  "at": "2025-07-16T10:01:00Z"
}
```

Possible `state` values:

- `pending`
- `sent`
- `delivered`
- `read`
- `failed`

When `state` is `failed`, `reason` can be present, for example:

- `whatsapp_unavailable`
- `blocked`

`GET /messages/{messageID}/status` returns the latest known state. Webhook remains the primary completion channel.

### Errors

| Status | Error | Meaning |
|--------|-------|---------|
| `403` | `not_authorized` | Missing/invalid access key |
| `404` | `message_not_found` | Unknown or expired message ID |
| `500` | `internal_error` | Unexpected internal error |
| `503` | `whatsapp_unavailable` | Temporary WhatsApp instability |

---

## Related

- [Webhooks](webhooks.md)
- [Rate Limits](rate-limits.md)
- [Errors](errors.md)
