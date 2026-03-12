# Glossary

> Terms and definitions used in Turbo Notify.

---

## Business Terms

### Tenant

A customer organization using the Turbo Notify platform. Each tenant has:
- Unique identifier
- API credentials
- One or more sessions
- Webhook configuration

### Session

A WhatsApp Web connection for a specific phone number. Represents the link between a WhatsApp account and the Turbo Notify platform.

### Message

A communication unit sent or received via WhatsApp. Can be text, image, document, audio, video, or location.

### Webhook

An HTTP callback that delivers events to a tenant's server. Used to notify tenants of incoming messages and status updates.

---

## Technical Terms

### Control Plane

The central API service (FastAPI) that manages all administrative operations: tenant management, session CRUD, message dispatch, and billing.

### Worker

A Python process that maintains active WhatsApp Web connections. Each worker manages multiple sessions.

### Orchestrator

Service responsible for allocating sessions to workers, managing leases, and monitoring worker health.

### Lease

Temporary ownership of a session by a worker. Prevents multiple workers from managing the same session simultaneously.

### NATS

A high-performance messaging system used for internal communication between services. JetStream provides persistence and at-least-once delivery.

### JetStream

NATS's persistence layer that provides streams, consumers, and durable message delivery.

---

## Session States

### Cold

Session created but has no authentication credentials. Cannot connect to WhatsApp.

### Warm

Session has valid authentication credentials but is not actively connected. Can be connected on demand.

### Hot

Session is actively connected to WhatsApp and can send/receive messages immediately.

### Disconnected

Session lost connection unexpectedly. Will attempt automatic reconnection.

### Reconnecting

Session is actively trying to re-establish connection after a disconnection.

### Failed

Session has exceeded maximum retry attempts. Requires manual intervention or re-authentication.

---

## Message States

### Pending

Message received by API but not yet sent to worker.

### Queued

Message dispatched to worker, waiting to be sent.

### Sent

Message successfully sent to WhatsApp servers.

### Delivered

Message delivered to recipient's device.

### Read

Message read by recipient (blue checkmarks).

### Failed

Message could not be sent due to an error.

---

## Architecture Patterns

### Minimal-State Session Runtime

Design principle where sessions maintain only essential state in memory. Persistent data is stored in PostgreSQL, allowing workers to be restarted without losing session configuration.

### Event-Driven Architecture

Communication pattern where services publish and subscribe to events via NATS rather than making direct API calls.

### Webhook Outbox

Pattern for reliable webhook delivery. Events are persisted in a queue and delivered with retries, ensuring at-least-once delivery.

### Distributed Lease

Mechanism for coordinating session ownership across workers. Only one worker can hold a lease at a time.

---

## Infrastructure

### VPS

Virtual Private Server - the underlying compute infrastructure.

### S3-Compatible Storage

Object storage for media files, compatible with Amazon S3 API.

### PostgreSQL

Primary database for persistent business data.

### Redis

In-memory data store used for caching and distributed coordination.

---

## Abbreviations

| Abbreviation | Meaning |
|--------------|---------|
| API | Application Programming Interface |
| JWT | JSON Web Token |
| NATS | Neural Autonomic Transport System |
| QR | Quick Response (code) |
| SaaS | Software as a Service |
| UUID | Universally Unique Identifier |
| HMAC | Hash-based Message Authentication Code |
| TLS | Transport Layer Security |

---

## Related Documentation

- [Architecture](../architecture/ecosystem-architecture.md) - System design
- [Session Lifecycle](../architecture/session-lifecycle.md) - Session states in detail
