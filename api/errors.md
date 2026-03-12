# Error Codes Reference

> Error contract for Turbo Notify public API.

---

## Error Shape

```json
{
  "error": "invalid_json"
}
```

---

## Core Error Catalog

| Error | Meaning |
|-------|---------|
| `invalid_json` | Invalid payload format |
| `not_authorized` | Missing/invalid bearer key |
| `not_available_in_plan` | Feature unavailable for current plan |
| `sender_not_authorized` | Sender (`from`) is not active/allowed |
| `message_not_found` | Message not found or expired |
| `number_not_found` | Extra number alias not found |
| `number_already_exists` | Number already registered |
| `invalid_status` | Action not allowed for current status |
| `contact_not_found` | Recipient has no WhatsApp |
| `rate_limit_exceeded` | Operation throughput limited |
| `messages_per_minute_limit_exceeded` | Minute cap exceeded |
| `messages_per_month_limit_exceeded` | Monthly cap exceeded |
| `recipients_per_day_limit_exceeded` | Daily recipients cap exceeded |
| `whatsapp_unavailable` | Temporary WhatsApp instability |
| `internal_error` | Unexpected internal failure |

---

## Endpoint Mapping

### `POST /messages`

- `400 invalid_json`
- `403 not_authorized`
- `422 sender_not_authorized`
- `429 rate_limit_exceeded`
- `500 internal_error`
- `503 whatsapp_unavailable`

### `GET /messages/{messageID}/status`

- `403 not_authorized`
- `404 message_not_found`
- `500 internal_error`
- `503 whatsapp_unavailable`

### `POST /extra-numbers`

- `400 invalid_json`
- `402 not_available_in_plan`
- `403 not_authorized`
- `409 number_already_exists`
- `500 internal_error`
- `503 whatsapp_unavailable`

### `GET /extra-numbers`

- `402 not_available_in_plan`
- `403 not_authorized`
- `500 internal_error`

### `GET /extra-numbers/{alias}/status`

- `402 not_available_in_plan`
- `403 not_authorized`
- `404 number_not_found`
- `500 internal_error`

### `POST /extra-numbers/activate`

- `400 invalid_json`
- `403 not_authorized`
- `409 invalid_status`
- `422 number_not_found`
- `500 internal_error`

### `POST /extra-numbers/deactivate`

- `400 invalid_json`
- `403 not_authorized`
- `409 invalid_status`
- `422 number_not_found`
- `500 internal_error`

### `DELETE /extra-numbers/{alias}`

- `402 not_available_in_plan`
- `403 not_authorized`
- `404 number_not_found`
- `500 internal_error`
- `503 whatsapp_unavailable`

### `POST /reactions`

- `400 invalid_json`
- `402 not_available_in_plan`
- `403 not_authorized`
- `422 contact_not_found`
- `422 message_not_found`
- `422 sender_not_authorized`
- `429 rate_limit_exceeded`
- `500 internal_error`
- `503 whatsapp_unavailable`

### `POST /typing-indicator`

- `400 invalid_json`
- `402 not_available_in_plan`
- `403 not_authorized`
- `422 contact_not_found`
- `422 sender_not_authorized`
- `429 rate_limit_exceeded`
- `500 internal_error`
- `503 whatsapp_unavailable`

---

## Related

- [Messages](messages.md)
- [Extra Numbers](extra-numbers.md)
- [Reactions](reactions.md)
- [Typing Indicator](typing-indicator.md)
