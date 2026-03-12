# Worker Architecture

> Python asyncio worker design following the Automacity pattern.

---

## Overview

Turbo Notify workers are **long-running Python processes** that maintain WhatsApp Web sessions. They follow the **minimal-state runtime** pattern where workers are disposable and can restart without losing data.

### Design Principles

1. **Asyncio-based** - Non-blocking I/O for maximum concurrency
2. **NATS-driven** - All commands and events via message bus
3. **Stateless design** - Persistent state in PostgreSQL, ephemeral in memory
4. **Graceful degradation** - Handle partial failures without crashing
5. **Observable** - Structured logging, metrics, distributed tracing
6. **Shared rate limiting** - Mandatory `rate-sync + redis` across worker fleet

---

## Architecture

```mermaid
graph TB
    subgraph "Worker Process"
        MAIN[Main Loop<br/>asyncio.run]

        subgraph "Session Manager"
            SM[Session Manager]
            S1[Session 1]
            S2[Session 2]
            SN[Session N]
        end

        subgraph "NATS Client"
            NC[NATS Connection]
            SUB[Subscriptions]
            PUB[Publisher]
        end

        subgraph "Services"
            HB[Heartbeat Service]
            RECON[Reconnect Handler]
            MET[Metrics Exporter]
        end
    end

    MAIN --> SM
    MAIN --> NC
    MAIN --> HB
    SM --> S1 & S2 & SN
    SM <--> NC
    HB --> NC
    MET --> PROM[(Prometheus)]
```

---

## Worker Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Starting: Process starts
    Starting --> Initializing: Config loaded
    Initializing --> Registering: NATS connected
    Registering --> Active: Registered with orchestrator
    Active --> Draining: Shutdown signal
    Active --> Recovering: Connection lost
    Recovering --> Active: Reconnected
    Recovering --> Draining: Timeout
    Draining --> [*]: All sessions released
```

### Startup Sequence

1. **Load Configuration** - Environment variables, secrets
2. **Connect to NATS** - Establish JetStream connection
3. **Connect to PostgreSQL** - Initialize connection pool
4. **Register with Orchestrator** - Announce capacity
5. **Subscribe to Commands** - Listen for session assignments
6. **Start Heartbeat** - Begin sending health signals

```python
async def main():
    # Load config
    config = WorkerConfig.from_env()

    # Connect to infrastructure
    nats = await connect_nats(config.nats_url)
    db = await create_pool(config.database_url)

    # Initialize components
    session_manager = SessionManager(nats, db)
    heartbeat = HeartbeatService(nats, config.worker_id)

    # Register and start
    await register_worker(nats, config.worker_id, config.capacity)

    # Run until shutdown
    async with asyncio.TaskGroup() as tg:
        tg.create_task(session_manager.run())
        tg.create_task(heartbeat.run())
        tg.create_task(command_handler(nats, session_manager))
```

---

## Rate Limit Coordination

Workers must use shared limiter policies via `rate-sync + redis`.

Rules:

- Use `rate-sync` for outbound WhatsApp/provider calls and webhook delivery workers.
- Do not use worker-local counters as primary throttling mechanism.
- Keep limiter IDs aligned with Control Plane/API policy definitions.
- Keep tenant-to-tier mapping aligned with Control Plane entitlement definitions.

```python
from ratesync import get_or_clone_limiter

async def send_with_limit(tenant_id: str, tier: str, sender_alias: str, payload: dict):
    limiter = await get_or_clone_limiter(f"whatsapp_send_{tier}", f"{tenant_id}:{sender_alias}")
    async with limiter.acquire_context(timeout=5.0):
        return await whatsapp_client.send(payload)
```

---

## Session Management

### Session States

| State | Description | Memory | NATS |
|-------|-------------|--------|------|
| `cold` | Not loaded | None | N/A |
| `warm` | Credentials loaded, not connected | Minimal | Subscribed |
| `hot` | Active connection | Full | Publishing |
| `disconnected` | Temporary disconnect | Partial | Subscribed |
| `reconnecting` | Attempting reconnect | Partial | Publishing |
| `failed` | Permanent failure | Minimal | Publishing |

### Session State Machine

```mermaid
stateDiagram-v2
    cold --> warm: assign_session
    warm --> hot: connect_session
    hot --> disconnected: ws_disconnect
    disconnected --> reconnecting: reconnect_timer
    reconnecting --> hot: ws_connected
    reconnecting --> disconnected: ws_failed
    disconnected --> failed: max_retries
    hot --> cold: release_session
    failed --> cold: release_session
```

### Session Handler

```python
class SessionHandler:
    """Manages a single WhatsApp Web session."""

    def __init__(
        self,
        session_id: str,
        nats: NatsClient,
        db: AsyncSession,
    ):
        self.session_id = session_id
        self.nats = nats
        self.db = db
        self.state = SessionState.COLD
        self.ws: WebSocket | None = None
        self.reconnect_attempts = 0

    async def connect(self) -> None:
        """Establish WhatsApp Web connection."""
        self.state = SessionState.WARM

        # Load credentials from DB
        creds = await self._load_credentials()

        # Connect to WhatsApp Web
        self.ws = await whatsapp_connect(creds)
        self.state = SessionState.HOT

        await self._publish_event("session.connected")

    async def handle_disconnect(self) -> None:
        """Handle WebSocket disconnection."""
        self.state = SessionState.DISCONNECTED
        await self._publish_event("session.disconnected")

        # Start reconnection with exponential backoff
        await self._schedule_reconnect()

    async def send_message(self, message: SendMessageCommand) -> None:
        """Send a message via WhatsApp Web."""
        if self.state != SessionState.HOT:
            raise SessionNotConnectedError(self.session_id)

        result = await self.ws.send(
            to=message.to,
            content=message.content,
            media=message.media,
        )

        await self._publish_event("message.sent", {
            "message_id": message.message_id,
            "status": "sent",
        })
```

---

## NATS Integration

### Command Subscriptions

Workers subscribe to commands filtered by worker ID:

```python
# Command subjects
turbo.worker.{worker_id}.assign_session
turbo.worker.{worker_id}.release_session
turbo.worker.{worker_id}.send_message
turbo.worker.{worker_id}.connect_session
turbo.worker.{worker_id}.disconnect_session
```

### Event Publishing

Workers publish events to shared streams:

```python
# Event subjects
turbo.session.{session_id}.connected
turbo.session.{session_id}.disconnected
turbo.session.{session_id}.failed
turbo.message.{message_id}.sent
turbo.message.{message_id}.delivered
turbo.message.{message_id}.failed
turbo.worker.{worker_id}.heartbeat
```

### Message Handler

```python
async def command_handler(
    nats: NatsClient,
    session_manager: SessionManager,
) -> None:
    """Handle incoming commands from NATS."""

    async for msg in nats.subscribe(f"turbo.worker.{WORKER_ID}.>"):
        cmd = parse_command(msg)

        match cmd:
            case AssignSessionCommand():
                await session_manager.assign(cmd.session_id)

            case ReleaseSessionCommand():
                await session_manager.release(cmd.session_id)

            case SendMessageCommand():
                await session_manager.send_message(
                    cmd.session_id,
                    cmd.message,
                )

            case _:
                logger.warning(f"Unknown command: {cmd}")

        await msg.ack()
```

---

## Heartbeat System

Workers send periodic heartbeats to prove liveness:

```python
class HeartbeatService:
    """Send periodic heartbeats to orchestrator."""

    def __init__(
        self,
        nats: NatsClient,
        worker_id: str,
        interval: float = 10.0,
    ):
        self.nats = nats
        self.worker_id = worker_id
        self.interval = interval

    async def run(self) -> None:
        """Run heartbeat loop."""
        while True:
            await self.nats.publish(
                f"turbo.worker.{self.worker_id}.heartbeat",
                {
                    "worker_id": self.worker_id,
                    "timestamp": datetime.utcnow().isoformat(),
                    "sessions_active": self._count_active(),
                    "sessions_capacity": self._get_capacity(),
                    "memory_mb": self._get_memory(),
                }
            )
            await asyncio.sleep(self.interval)
```

**Failure detection:**
- Orchestrator expects heartbeat every 10s
- After 30s without heartbeat, worker marked unhealthy
- After 60s, sessions reassigned to other workers

---

## Resource Management

### Memory Limits

| Component | Memory | Notes |
|-----------|--------|-------|
| Base worker | 50-100 MB | Runtime + dependencies |
| Per session (warm) | 5-10 MB | Credentials + metadata |
| Per session (hot) | 20-50 MB | WebSocket + buffers |
| Message buffers | Variable | Auto-flush on pressure |

### Concurrency Limits

```python
class WorkerConfig:
    # Maximum concurrent sessions
    max_sessions: int = 10

    # Maximum concurrent message sends
    max_concurrent_sends: int = 50

    # Connection pool sizes
    db_pool_size: int = 5
    nats_pool_size: int = 2
```

### Graceful Shutdown

```python
async def shutdown(session_manager: SessionManager) -> None:
    """Gracefully shutdown worker."""
    logger.info("Shutdown initiated")

    # Stop accepting new commands
    await nats.drain()

    # Release all sessions
    for session in session_manager.sessions.values():
        await session.release()

    # Wait for pending operations
    await session_manager.wait_pending(timeout=30)

    # Close connections
    await db.close()
    await nats.close()

    logger.info("Shutdown complete")
```

---

## Error Handling

### Retry Strategy

| Error Type | Retry | Backoff | Max Attempts |
|------------|-------|---------|--------------|
| Network timeout | Yes | Exponential | 5 |
| Rate limit | Yes | Fixed 60s | 3 |
| Auth expired | No | N/A | 0 |
| Invalid message | No | N/A | 0 |
| Server error | Yes | Exponential | 3 |

### Circuit Breaker

```python
class CircuitBreaker:
    """Prevent cascade failures."""

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: float = 60.0,
    ):
        self.failures = 0
        self.threshold = failure_threshold
        self.timeout = recovery_timeout
        self.state = CircuitState.CLOSED
        self.last_failure: datetime | None = None

    async def call(self, func: Callable) -> Any:
        if self.state == CircuitState.OPEN:
            if self._should_try_recovery():
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitOpenError()

        try:
            result = await func()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise
```

---

## Observability

### Structured Logging

```python
import structlog

logger = structlog.get_logger()

logger.info(
    "message_sent",
    session_id=session_id,
    message_id=message_id,
    recipient=recipient,
    latency_ms=latency,
)
```

### Metrics

| Metric | Type | Labels |
|--------|------|--------|
| `worker_sessions_active` | Gauge | `worker_id`, `state` |
| `worker_messages_sent_total` | Counter | `worker_id`, `status` |
| `worker_reconnects_total` | Counter | `worker_id`, `reason` |
| `worker_heartbeat_latency_seconds` | Histogram | `worker_id` |
| `session_connection_duration_seconds` | Histogram | `session_id` |

### Tracing

```python
from opentelemetry import trace

tracer = trace.get_tracer(__name__)

async def send_message(self, message: SendMessageCommand) -> None:
    with tracer.start_as_current_span("send_message") as span:
        span.set_attribute("session_id", self.session_id)
        span.set_attribute("message_id", message.message_id)

        # ... send logic ...
```

---

## Directory Structure

```
src/
├── worker/
│   ├── __init__.py
│   ├── main.py              # Entry point
│   ├── config.py            # Configuration
│   ├── session_manager.py   # Session orchestration
│   ├── session_handler.py   # Individual session
│   ├── nats_client.py       # NATS integration
│   ├── heartbeat.py         # Heartbeat service
│   ├── metrics.py           # Prometheus metrics
│   └── commands/
│       ├── __init__.py
│       ├── assign_session.py
│       ├── release_session.py
│       └── send_message.py
├── shared/
│   ├── models/
│   │   ├── session.py
│   │   └── message.py
│   ├── nats/
│   │   ├── events.py
│   │   └── commands.py
│   └── database/
│       ├── connection.py
│       └── repositories/
└── tests/
    └── worker/
```

---

## Deployment

### Docker

```dockerfile
FROM python:3.13-slim

WORKDIR /app

# Install dependencies
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && poetry install --no-dev

# Copy source
COPY src/ ./src/

# Run worker
CMD ["python", "-m", "src.worker.main"]
```

### Environment Variables

```env
# Worker identity
WORKER_ID=worker-001

# Infrastructure
NATS_URL=nats://nats:4222
DATABASE_URL=postgresql+asyncpg://user:pass@postgres/turbo

# Limits
MAX_SESSIONS=10
MAX_CONCURRENT_SENDS=50

# Observability
LOG_LEVEL=INFO
METRICS_PORT=9090
OTEL_EXPORTER_OTLP_ENDPOINT=http://jaeger:4317
```

---

## Related Documentation

- [Ecosystem Architecture](ecosystem-architecture.md) - System overview
- [NATS Events](nats-events.md) - Message contracts
- [Session Lifecycle](session-lifecycle.md) - State machine details
- [Observability Overview](/docs/observability/observability-overview.md) - Monitoring
- [ADR: rate-sync as Mandatory Rate-Limiting Engine](../reference/decisions/2026-03-12-rate-sync-rate-limiting.md)
