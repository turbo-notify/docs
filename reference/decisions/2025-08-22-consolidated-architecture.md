# ADR: Consolidated Architecture with Ledger and Integration

**Date:** 2025-08-22
**Status:** Accepted (Updated 2026-03-12)
**Language:** Originally in Portuguese, key decisions preserved

---

## Context

After discussing languages, frameworks, queues, and databases, we consolidated the base architecture for Turbo Notify. Considerations included:
- Resource efficiency (RAM/CPU)
- Operational simplicity
- Scalability for volume growth
- Business model support (plans, billing, auditing)

## Technology Comparison

### Languages and Frameworks

| Stack | RAM idle/load | Startup | Throughput | Notes |
|-------|---------------|---------|------------|-------|
| Go (chi/echo) | 10-25 MB | 10-40 ms | High | Single binary, lightweight |
| Python (FastAPI) | 70-150 MB | 150-700 ms | Medium | Productive but heavier |
| Node.js (Fastify) | 70-150 MB | 100-500 ms | Good I/O | Vast ecosystem |

### Queue Systems

| Technology | Strengths | Fit |
|------------|-----------|-----|
| NATS JetStream | Light, high throughput, DLQ | **Excellent for jobs** |
| Postgres FOR UPDATE | Simple, one dependency | Good for low/medium volume |
| RabbitMQ | Sophisticated routing | Good for complex fanout |
| Kafka | Massive scale | Overkill for this stage |

### Databases

| Database | RAM | CPU | Notes |
|----------|-----|-----|-------|
| PostgreSQL | 100-500 MB | Low-moderate | Efficient, great for ledger |
| MongoDB | 300 MB-2 GB+ | Moderate | Flexible but higher RAM |
| Redis | 100 MB-GBs | Very low | In-memory only |

## Architecture Decisions

### Components

| Component | Technology | Responsibility |
|-----------|------------|----------------|
| **API** | Python + FastAPI | Authentication, quota control, ledger writing, job enqueuing |
| **Worker** | Python + asyncio | Job consumption, WAHA integration, status updates, webhooks |
| **Landing** | Next.js | Lead capture, events, Postgres persistence |
| **Dashboard** | Next.js | Customer access, consumption display, number/token management |
| **Ledger** | PostgreSQL | Definitive accounting/audit record, billing base |
| **Queue** | NATS JetStream | Persistent queue with DLQ and backoff |
| **Rate Limit Engine** | rate-sync + Redis | Shared distributed limits for API, workers, and webhook dispatcher |
| **Gateway** | WAHA | WhatsApp sending, operated by Worker |

## Consequences

- Clear and scalable architecture with low initial operational cost
- NATS handles real-time orchestration; PostgreSQL is the accounting truth
- Unified language (Python) simplifies maintenance
- Unified frontend (Next.js) covers Landing and Dashboard

## Next Steps

1. Finalize Landing and put in production
2. Start Dashboard development in Next.js
3. Define SQL ledger schema in Postgres (with billing indexes)
4. Configure NATS JetStream with retry and DLQ policies
5. Implement and document throttling per number/tenant with `rate-sync + redis`

---

## Updates

**2026-03-12:** Updated to reflect Python stack decision (was Go). See ADR-0001.
