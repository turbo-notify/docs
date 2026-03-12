# API Contract Alignment

> Convergence and divergence analysis between user-facing API and internal architecture.

---

## Overview

This document tracks alignment between:
- **User-Facing API** (`/landing/src/content/docs/`) - What we promise customers
- **Internal Architecture** (`/docs/architecture/`) - How we build it

**Source of Truth Priority:**
1. User-facing API defines the **external contract** (what customers see)
2. Internal architecture defines **how** we implement it

---

## Terminology Mapping

| User-Facing Term | Internal Term | Notes |
|------------------|---------------|-------|
| Extra Number | Session | Extra numbers are sessions with `type=extra` |
| Main Number | Session (primary) | Session with `type=primary` |
| Access Key | API Key | Same concept, different naming |
| Alias | phone_name | Session identifier |

---

## Endpoint Alignment

### Messages

| User-Facing | Internal | Status |
|-------------|----------|--------|
| `POST /messages` | `POST /v1/messages` | ✅ Aligned |
| `GET /messages/{id}` | `GET /v1/messages/{id}` | ✅ Aligned |
| — | `GET /v1/messages` (list) | 🔵 Internal only |
| — | `GET /v1/sessions/{id}/messages` | 🔵 Internal only |

**Divergence Notes:**
- User-facing uses simple paths, internal uses `/v1/` prefix
- User-facing returns `202 Accepted`, internal may return `201 Created`

### Extra Numbers (Sessions)

| User-Facing | Internal | Status |
|-------------|----------|--------|
| `POST /extra-numbers` | `POST /v1/sessions` | ⚠️ Different paths |
| `GET /extra-numbers` | `GET /v1/sessions?type=extra` | ⚠️ Query filter |
| `GET /extra-numbers/{alias}/status` | `GET /v1/sessions/{id}` | ⚠️ Uses alias vs ID |
| `POST /extra-numbers/{alias}/activate` | `POST /v1/sessions/{id}/connect` | ⚠️ Different action name |
| `POST /extra-numbers/{alias}/deactivate` | `POST /v1/sessions/{id}/disconnect` | ⚠️ Different action name |
| `DELETE /extra-numbers/{alias}` | `DELETE /v1/sessions/{id}` | ⚠️ Uses alias vs ID |

**Resolution Required:**
- Create alias-to-ID lookup layer in API
- Map `/extra-numbers/` routes to session operations with `type=extra`

### Reactions (Missing in Internal)

| User-Facing | Internal | Status |
|-------------|----------|--------|
| `POST /reactions` | — | ❌ Not documented |

**Action Required:**
- Add `POST /v1/reactions` to architecture docs
- Add `turbo.reaction.send.command` NATS event

### Typing Indicator (Missing in Internal)

| User-Facing | Internal | Status |
|-------------|----------|--------|
| `POST /typing-indicator` | — | ❌ Not documented |

**Action Required:**
- Add `POST /v1/typing-indicator` to architecture docs
- Add `turbo.typing.send.command` NATS event

---

## Webhook Events Alignment

| User-Facing Event | Internal NATS Subject | Status |
|-------------------|----------------------|--------|
| `message.sent` | `turbo.message.sent.event` | ✅ Aligned |
| `message.received` | `turbo.message.received.event` | ✅ Aligned |
| `message.reaction` | — | ❌ Missing |
| `message.edited` | — | ❌ Missing |
| `message.deleted` | — | ❌ Missing |
| `message.replied` | — | ❌ Missing |

**Action Required:**
Add NATS events for:
- `turbo.message.reaction.event`
- `turbo.message.edited.event`
- `turbo.message.deleted.event`
- `turbo.message.replied.event`

---

## Message Status Mapping

| User-Facing State | Internal State | Notes |
|-------------------|----------------|-------|
| `pending` | `pending`, `queued` | Internal has finer granularity |
| `sent` | `sent`, `delivered`, `read` | User sees simplified |
| `failed` | `failed` | ✅ Aligned |

**Resolution:**
- Internal tracks detailed states
- API gateway simplifies for user-facing responses

---

## Extra Number Status Mapping

| User-Facing State | Internal Session State | Notes |
|-------------------|------------------------|-------|
| `active` | `hot` | Connected and ready |
| `inactive` | `warm` | Credentials valid, not connected |
| `pending` | `cold`, `connecting` | Awaiting verification |
| `failed` | `failed` | Error state |

---

## Error Code Alignment

### User-Facing Errors

| Code | HTTP | Internal Mapping |
|------|------|------------------|
| `invalid_json` | 400 | `VALIDATION_ERROR` |
| `not_authorized` | 403 | `UNAUTHORIZED` |
| `sender_not_authorized` | 422 | `SESSION_NOT_CONNECTED` |
| `rate_limit_exceeded` | 429 | `RATE_LIMIT_EXCEEDED` |
| `message_not_found` | 404 | `MESSAGE_NOT_FOUND` |
| `number_not_found` | 404 | `SESSION_NOT_FOUND` |
| `number_already_exists` | 409 | `SESSION_ALREADY_EXISTS` |
| `invalid_status` | 409 | `INVALID_STATE_TRANSITION` |
| `not_available_in_plan` | 402 | `FEATURE_NOT_AVAILABLE` |
| `whatsapp_unavailable` | 503 | `WHATSAPP_UNAVAILABLE` |
| `internal_error` | 500 | `INTERNAL_ERROR` |

---

## Database Schema Additions Required

### sessions table updates

```sql
-- Add type column for primary vs extra
ALTER TABLE sessions ADD COLUMN type VARCHAR(20) DEFAULT 'extra';
-- Values: 'primary', 'extra'

-- Add alias as unique per tenant
ALTER TABLE sessions ADD COLUMN alias VARCHAR(100);
CREATE UNIQUE INDEX idx_sessions_tenant_alias ON sessions(tenant_id, alias);
```

### reactions table (new)

```sql
CREATE TABLE reactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id UUID NOT NULL REFERENCES tenants(id),
    session_id UUID NOT NULL REFERENCES sessions(id),
    message_id UUID NOT NULL REFERENCES messages(id),
    to_number VARCHAR(20) NOT NULL,
    reaction VARCHAR(10),  -- emoji or null to remove
    created_at TIMESTAMP NOT NULL DEFAULT NOW()
);
```

---

## NATS Events Additions Required

### Reaction Events

```
Subject: turbo.reaction.send.command
{
  "command_id": "cmd_react001",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "to": "+5511888888888",
  "message_id": "msg_abc123",
  "reaction": "🎉",
  "requested_at": "2024-01-15T10:30:00Z"
}

Subject: turbo.reaction.sent.event
{
  "event_id": "evt_react001",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "message_id": "msg_abc123",
  "reaction": "🎉",
  "sent_at": "2024-01-15T10:30:02Z"
}
```

### Typing Events

```
Subject: turbo.typing.send.command
{
  "command_id": "cmd_type001",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "to": "+5511888888888",
  "value": true,
  "requested_at": "2024-01-15T10:30:00Z"
}
```

### Message Edit/Delete Events

```
Subject: turbo.message.edited.event
{
  "event_id": "evt_edit001",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "message_id": "msg_abc123",
  "wa_message_id": "wamid.abc123",
  "new_text": "Updated message content",
  "edited_at": "2024-01-15T10:35:00Z"
}

Subject: turbo.message.deleted.event
{
  "event_id": "evt_del001",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "message_id": "msg_abc123",
  "wa_message_id": "wamid.abc123",
  "deleted_at": "2024-01-15T10:40:00Z"
}
```

---

## Implementation Priority

### P0 - Must Have for MVP

- [x] POST /messages (send text)
- [x] GET /messages/{id} (status)
- [x] Webhook: message.sent, message.received
- [ ] POST /extra-numbers (add)
- [ ] GET /extra-numbers (list)
- [ ] DELETE /extra-numbers/{alias} (remove)

### P1 - First Release

- [ ] Activate/deactivate extra numbers
- [ ] Webhook: message.reaction
- [ ] POST /reactions

### P2 - Enhancement

- [ ] POST /typing-indicator
- [ ] Webhook: message.edited, message.deleted, message.replied

---

## Related Documentation

- [User-Facing API](/landing/src/content/docs/) - Customer documentation
- [NATS Events](nats-events.md) - Internal messaging
- [Database Schema](database-schema.md) - Data model
- [API Standards](../standards/api-standards.md) - Conventions
