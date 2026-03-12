# NATS Events

> Message subjects, contracts, and patterns for NATS JetStream.

---

## Overview

Turbo Notify uses NATS JetStream for:
- Command dispatch (API → Workers)
- Event publication (Workers → Consumers)
- Webhook outbox pattern
- Worker coordination

### Naming Convention

```
turbo.{domain}.{action}.{qualifier}
```

Examples:
- `turbo.session.connect.request`
- `turbo.message.send.command`
- `turbo.webhook.delivery.pending`

---

## Streams

### SESSIONS Stream

Session lifecycle events.

```
Stream: SESSIONS
Subjects: turbo.session.>
Retention: Limits (7 days)
Storage: File
```

### MESSAGES Stream

Message send/receive events.

```
Stream: MESSAGES
Subjects: turbo.message.>
Retention: Limits (24 hours)
Storage: File
```

### WEBHOOKS Stream

Webhook delivery outbox.

```
Stream: WEBHOOKS
Subjects: turbo.webhook.>
Retention: WorkQueue
Storage: File
```

### HEARTBEAT Stream

Worker health monitoring.

```
Stream: HEARTBEAT
Subjects: turbo.worker.>
Retention: Limits (1 hour)
Storage: Memory
```

---

## Message Contracts

### Session Commands

#### Connect Session

**Subject:** `turbo.session.connect.command`

```json
{
  "command_id": "cmd_abc123",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "phone_number": "+5511999999999",
  "auth_blob": "base64_encoded_credentials",
  "requested_at": "2024-01-15T10:30:00Z"
}
```

#### Disconnect Session

**Subject:** `turbo.session.disconnect.command`

```json
{
  "command_id": "cmd_def456",
  "session_id": "sess_xyz789",
  "reason": "user_requested",
  "requested_at": "2024-01-15T11:00:00Z"
}
```

### Session Events

#### Session Connected

**Subject:** `turbo.session.connected.event`

```json
{
  "event_id": "evt_001",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "worker_id": "worker_02",
  "connected_at": "2024-01-15T10:30:05Z"
}
```

#### Session Disconnected

**Subject:** `turbo.session.disconnected.event`

```json
{
  "event_id": "evt_002",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "reason": "connection_lost",
  "disconnected_at": "2024-01-15T12:00:00Z",
  "will_reconnect": true
}
```

#### Session Failed

**Subject:** `turbo.session.failed.event`

```json
{
  "event_id": "evt_003",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "error_code": "AUTH_EXPIRED",
  "error_message": "QR code needs re-scan",
  "failed_at": "2024-01-15T12:00:00Z",
  "recoverable": false
}
```

---

### Message Commands

#### Send Message

**Subject:** `turbo.message.send.command`

```json
{
  "command_id": "cmd_msg001",
  "message_id": "msg_abc123",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "to": "+5511888888888",
  "type": "text",
  "content": {
    "text": "Hello, your order is ready!"
  },
  "idempotency_key": "order_123_notification",
  "requested_at": "2024-01-15T10:30:00Z"
}
```

**Message Types:**

| Type | Content Fields |
|------|----------------|
| `text` | `text` |
| `image` | `url`, `caption` |
| `document` | `url`, `filename`, `caption` |
| `audio` | `url` |
| `video` | `url`, `caption` |
| `location` | `latitude`, `longitude`, `name` |

### Message Events

#### Message Sent

**Subject:** `turbo.message.sent.event`

```json
{
  "event_id": "evt_msg001",
  "message_id": "msg_abc123",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "to": "+5511888888888",
  "wa_message_id": "wamid.abc123xyz",
  "sent_at": "2024-01-15T10:30:02Z"
}
```

#### Message Delivered

**Subject:** `turbo.message.delivered.event`

```json
{
  "event_id": "evt_msg002",
  "message_id": "msg_abc123",
  "wa_message_id": "wamid.abc123xyz",
  "delivered_at": "2024-01-15T10:30:05Z"
}
```

#### Message Read

**Subject:** `turbo.message.read.event`

```json
{
  "event_id": "evt_msg003",
  "message_id": "msg_abc123",
  "wa_message_id": "wamid.abc123xyz",
  "read_at": "2024-01-15T10:32:00Z"
}
```

#### Message Failed

**Subject:** `turbo.message.failed.event`

```json
{
  "event_id": "evt_msg004",
  "message_id": "msg_abc123",
  "session_id": "sess_xyz789",
  "error_code": "RECIPIENT_NOT_FOUND",
  "error_message": "Phone number not on WhatsApp",
  "failed_at": "2024-01-15T10:30:03Z"
}
```

#### Message Received (Incoming)

**Subject:** `turbo.message.received.event`

```json
{
  "event_id": "evt_msg005",
  "session_id": "sess_xyz789",
  "tenant_id": "tenant_001",
  "from": "+5511777777777",
  "wa_message_id": "wamid.incoming123",
  "type": "text",
  "content": {
    "text": "Hi, I have a question about my order"
  },
  "timestamp": "2024-01-15T10:35:00Z"
}
```

#### Message Reaction

**Subject:** `turbo.message.reaction.event`

```json
{
  "event_id": "evt_msg006",
  "message_id": "msg_abc123",
  "tenant_id": "tenant_001",
  "action": "apply",
  "reaction": "🎉",
  "actor": {
    "number": "+5511999999999"
  },
  "operation_id": "op_reaction_001",
  "timestamp": "2024-01-15T10:36:00Z"
}
```

#### Message Reaction Failed

**Subject:** `turbo.message.reaction.failed.event`

```json
{
  "event_id": "evt_msg007",
  "message_id": "msg_abc123",
  "tenant_id": "tenant_001",
  "operation_id": "op_reaction_001",
  "reason": "whatsapp_unavailable",
  "failed_at": "2024-01-15T10:36:01Z"
}
```

#### Typing Started

**Subject:** `turbo.typing.started.event`

```json
{
  "event_id": "evt_typing001",
  "tenant_id": "tenant_001",
  "operation_id": "op_typing_001",
  "to": "+5511888888888",
  "started_at": "2024-01-15T10:37:00Z"
}
```

#### Typing Stopped

**Subject:** `turbo.typing.stopped.event`

```json
{
  "event_id": "evt_typing002",
  "tenant_id": "tenant_001",
  "operation_id": "op_typing_002",
  "to": "+5511888888888",
  "stopped_at": "2024-01-15T10:37:10Z"
}
```

#### Typing Failed

**Subject:** `turbo.typing.failed.event`

```json
{
  "event_id": "evt_typing003",
  "tenant_id": "tenant_001",
  "operation_id": "op_typing_003",
  "to": "+5511888888888",
  "reason": "whatsapp_unavailable",
  "failed_at": "2024-01-15T10:37:02Z"
}
```

---

### Webhook Events

Webhook delivery is asynchronous and must be paced by shared limiter policies (`rate-sync + redis` in production).

#### Webhook Pending

**Subject:** `turbo.webhook.delivery.pending`

```json
{
  "delivery_id": "del_001",
  "tenant_id": "tenant_001",
  "webhook_url": "https://tenant-app.com/webhooks/whatsapp",
  "event_type": "message.received",
  "payload": { ... },
  "limiter_id": "webhook_delivery:tenant_001",
  "attempt": 1,
  "max_attempts": 5,
  "created_at": "2024-01-15T10:35:01Z"
}
```

#### Webhook Delivered

**Subject:** `turbo.webhook.delivery.success`

```json
{
  "delivery_id": "del_001",
  "tenant_id": "tenant_001",
  "status_code": 200,
  "delivered_at": "2024-01-15T10:35:02Z",
  "latency_ms": 150
}
```

#### Webhook Failed

**Subject:** `turbo.webhook.delivery.failed`

```json
{
  "delivery_id": "del_001",
  "tenant_id": "tenant_001",
  "attempt": 3,
  "error": "Connection timeout",
  "next_retry_at": "2024-01-15T10:40:02Z"
}
```

---

### Worker Events

#### Worker Heartbeat

**Subject:** `turbo.worker.heartbeat`

```json
{
  "worker_id": "worker_02",
  "hostname": "worker-02.internal",
  "active_sessions": 15,
  "memory_mb": 512,
  "cpu_percent": 25.5,
  "uptime_seconds": 86400,
  "timestamp": "2024-01-15T10:30:00Z"
}
```

#### Worker Started

**Subject:** `turbo.worker.started.event`

```json
{
  "worker_id": "worker_02",
  "hostname": "worker-02.internal",
  "version": "1.2.3",
  "started_at": "2024-01-15T08:00:00Z"
}
```

#### Worker Stopping

**Subject:** `turbo.worker.stopping.event`

```json
{
  "worker_id": "worker_02",
  "reason": "graceful_shutdown",
  "active_sessions": 15,
  "stopping_at": "2024-01-15T20:00:00Z"
}
```

---

## Consumer Groups

### Message Processor

Processes outbound message commands.

```
Consumer: message-processor
Stream: MESSAGES
Filter: turbo.message.send.command
Deliver: Push
AckPolicy: Explicit
MaxDeliver: 3
```

### Webhook Dispatcher

Delivers webhooks to tenants.
Uses shared `rate-sync` limiter definitions backed by Redis in production.

```
Consumer: webhook-dispatcher
Stream: WEBHOOKS
Filter: turbo.webhook.delivery.pending
Deliver: Pull
AckPolicy: Explicit
MaxDeliver: 5
AckWait: 30s
```

### Event Persister

Stores events in PostgreSQL.

```
Consumer: event-persister
Stream: MESSAGES
Filter: turbo.message.*.event
Deliver: Push
AckPolicy: Explicit
```

---

## Best Practices

### Idempotency

Always include `idempotency_key` in commands:

```json
{
  "idempotency_key": "order_123_notification_v1"
}
```

Workers must check before processing to avoid duplicates.

### Error Handling

Use error codes for programmatic handling:

| Code | Meaning |
|------|---------|
| `AUTH_EXPIRED` | Session needs re-authentication |
| `RECIPIENT_NOT_FOUND` | Phone not on WhatsApp |
| `RATE_LIMITED` | Too many messages |
| `SESSION_OFFLINE` | Session not connected |
| `MEDIA_TOO_LARGE` | File exceeds limit |

### Ordering

Messages within a session should be ordered. Use session_id as partition key:

```
Subject: turbo.message.send.command.{session_id}
```

---

## Related Documentation

- [Ecosystem Architecture](ecosystem-architecture.md) - System overview
- [Session Lifecycle](session-lifecycle.md) - Session states
