# Typing Indicator API

> Start or stop typing status for a contact.

---

## Endpoint

| Method | Endpoint |
|--------|----------|
| `POST` | `/typing-indicator` |

---

## Request

```http
POST /typing-indicator
Content-Type: application/json
Authorization: Bearer <access_key>
```

```json
{
  "to": "+5511999999999",
  "value": true,
  "from": "financeiro"
}
```

| Field | Type | Required | Notes |
|-------|------|----------|-------|
| `to` | string | Yes | Recipient phone |
| `value` | boolean | Yes | `true` to start, `false` to stop |
| `from` | string | No | Extra number phone or alias |

Success:

```http
202 Accepted
```

```json
{
  "operation_id": "op_typing_001",
  "status": {
    "state": "pending",
    "at": "2025-07-16T10:00:00Z"
  }
}
```

---

## Behavior

- Turbo Notify can manage typing automatically during send flows.
- This endpoint exists for explicit control in custom processing flows.
- `202 Accepted` confirms async acceptance only.
- Operation completion is delivered asynchronously via webhook:
  - `typing.started` when `value = true`
  - `typing.stopped` when `value = false`
  - `typing.failed` when operation cannot be completed

There is no status polling endpoint for typing operations; webhook is the completion channel.

---

## Errors

| Status | Error | Meaning |
|--------|-------|---------|
| `400` | `invalid_json` | Invalid payload |
| `402` | `not_available_in_plan` | Feature unavailable for plan |
| `403` | `not_authorized` | Missing/invalid access key |
| `422` | `contact_not_found` | Recipient has no WhatsApp |
| `422` | `sender_not_authorized` | `from` is not active/allowed |
| `429` | `rate_limit_exceeded` | Shared `rate-sync` limit exceeded |
| `500` | `internal_error` | Unexpected internal error |
| `503` | `whatsapp_unavailable` | Temporary WhatsApp instability |

---

## Related

- [Messages](messages.md)
- [Webhooks](webhooks.md)
- [Extra Numbers](extra-numbers.md)
- [Rate Limits](rate-limits.md)
- [Errors](errors.md)
