# Webhooks

> Async event delivery contract for client integrations.

---

## Purpose

Turbo Notify sends webhook events for message lifecycle updates and async operation results.

For endpoints that return `202 Accepted`, webhook events are the primary completion channel.
`202` only confirms async enqueue/acceptance, not final WhatsApp outcome.

---

## Scope

- Exactly one webhook configuration per account.
- The same webhook is used for the main sender number and all extra numbers.
- URL and secret are configured in dashboard.

---

## Receiver Requirements

- Public HTTPS endpoint accepting `POST`.
- Fast `2xx` acknowledgment (process async when possible).
- Idempotent event handling.
- Raw request body preserved for signature verification.

---

## Security

Turbo Notify signs each webhook request with HMAC SHA-256:

```http
X-Webhook-Signature: sha256=<hmac_hex>
```

Receiver must verify signature before processing payload.

Recommended:

- HTTPS only
- secret rotation policy
- deny requests with invalid/missing signature

Signature base:

```text
HMAC_SHA256(secret, raw_request_body)
```

---

## Event Envelope

```json
{
  "id": "evt_123",
  "type": "message.sent",
  "timestamp": "2026-03-12T12:34:56Z",
  "data": {
    "message_id": "msg_abc123",
    "operation_id": "op_msg_001",
    "to": {
      "number": "+5511999999999"
    },
    "from": {
      "number": "+5511888888888",
      "alias": "laboratorio_central"
    }
  }
}
```

Common fields:

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Unique event ID for idempotency |
| `type` | string | Event type |
| `timestamp` | string (ISO-8601 UTC) | Event creation time |
| `data` | object | Event-specific payload |

---

## Event Types

| Event | Description |
|-------|-------------|
| `message.sent` | Outbound message accepted by WhatsApp |
| `message.delivered` | Outbound message delivered |
| `message.read` | Outbound message read |
| `message.failed` | Outbound message failed |
| `message.received` | Message received from contact |
| `message.reaction` | Reaction applied/removed on message |
| `message.reaction.failed` | Reaction operation failed |
| `typing.started` | Typing indicator enabled |
| `typing.stopped` | Typing indicator disabled |
| `typing.failed` | Typing indicator operation failed |
| `message.edited` | Message was edited |
| `message.deleted` | Message was deleted |
| `message.replied` | Message received direct reply |

---

## Source To Webhook Mapping

| Source | API return | Webhook events |
|--------|------------|----------------|
| `POST /messages` | `202 Accepted` | `message.sent`, `message.delivered`, `message.read`, `message.failed` |
| `POST /reactions` | `202 Accepted` | `message.reaction`, `message.reaction.failed` |
| `POST /typing-indicator` | `202 Accepted` | `typing.started`, `typing.stopped`, `typing.failed` |
| Inbound WhatsApp activity (no REST call) | N/A | `message.received`, `message.edited`, `message.deleted`, `message.replied` |

---

## Event Data Fields

`data` fields by event:

- `message.sent`: `message_id`, `operation_id`, `to`, `from`, optional `wa_message_id`.
- `message.delivered`: `message_id`, optional `wa_message_id`.
- `message.read`: `message_id`, optional `wa_message_id`.
- `message.failed`: `message_id`, `operation_id`, `reason`, optional `provider_error`.
- `message.received`: `message_id`, `from`, `to`, `content`.
- `message.reaction`: `message_id`, `action` (`apply` or `remove`), `reaction`, `actor`, optional `operation_id`.
- `message.reaction.failed`: `message_id`, `operation_id`, `reason`, `reaction`.
- `typing.started`: `operation_id`, `to`, `from`.
- `typing.stopped`: `operation_id`, `to`, `from`.
- `typing.failed`: `operation_id`, `to`, `from`, `reason`.
- `message.edited`: `message_id`, `from`, `to`, `content`.
- `message.deleted`: `message_id`, `from`, `to`.
- `message.replied`: `message_id`, `parent_message_id`, `from`, `to`, `content`.

---

## Delivery Policy

- Up to **5 attempts** when receiver does not return `2xx`.
- Timeout per attempt: **10 seconds**.
- Delivery is **at-least-once**.
- Duplicate and out-of-order events can happen.
- Delivery pacing and concurrency controls use shared `rate-sync` policies.
- Production delivery control is mandatory `rate-sync + redis`.
- Delivery throughput is isolated per tenant (noisy-neighbor protection).
- Contracted tier can change webhook dispatch throughput/concurrency ceilings.
- When tenant/tier limits are saturated, events remain queued and follow retry policy.

| Attempt | Timing |
|---------|--------|
| 1 | immediate |
| 2 | +12s |
| 3 | +24s |
| 4 | +48s |
| 5 | +96s |

After max retries, event is moved to dead-letter and no further delivery attempts are made.

---

## Related

- [Messages](messages.md)
- [Reactions](reactions.md)
- [Typing Indicator](typing-indicator.md)
- [Rate Limits](rate-limits.md)
- [Errors](errors.md)
