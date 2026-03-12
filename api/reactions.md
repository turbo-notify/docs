# Reactions API

> Add or remove reactions on existing messages.

---

## Endpoint

| Method | Endpoint |
|--------|----------|
| `POST` | `/reactions` |

---

## Add Reaction

```http
POST /reactions
Content-Type: application/json
Authorization: Bearer <access_key>
```

```json
{
  "to": "+5511999999999",
  "message_id": "msg_abc123",
  "reaction": "🎉",
  "from": "financeiro"
}
```

Success:

```http
202 Accepted
```

```json
{
  "operation_id": "op_reaction_001",
  "message_id": "msg_abc123",
  "status": {
    "state": "pending",
    "at": "2025-07-16T10:00:00Z"
  }
}
```

---

## Remove Reaction

Use `reaction: null`.

```json
{
  "to": "+5511999999999",
  "message_id": "msg_abc123",
  "reaction": null
}
```

Success:

```http
202 Accepted
```

```json
{
  "operation_id": "op_reaction_002",
  "message_id": "msg_abc123",
  "status": {
    "state": "pending",
    "at": "2025-07-16T10:00:05Z"
  }
}
```

Operation completion is delivered asynchronously via webhook events:

- `message.reaction` (success, with `action` = `apply` or `remove`)
- `message.reaction.failed`

`202 Accepted` confirms async acceptance only. Final outcome is webhook-only (no status polling endpoint for reactions).

---

## Errors

| Status | Error | Meaning |
|--------|-------|---------|
| `400` | `invalid_json` | Invalid payload |
| `402` | `not_available_in_plan` | Feature unavailable for plan |
| `403` | `not_authorized` | Missing/invalid access key |
| `422` | `contact_not_found` | Recipient has no WhatsApp |
| `422` | `message_not_found` | Unknown/expired `message_id` |
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
