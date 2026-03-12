# ADR: Database, Queue, and Architecture

**Date:** 2025-08-21
**Status:** Accepted
**Language:** Originally in Portuguese, key decisions preserved

---

## Context

Initial discussion about technology choices for the Turbo Notify platform covering frameworks, queues, and databases.

## Decisions

### API Framework
- **FastAPI** chosen for productivity, type safety, auto-documentation, and good performance
- Go considered for extreme performance but requires higher learning curve
- **Conclusion:** FastAPI sufficient for current needs; Go only if extreme performance required

### Message Queue
- **NATS JetStream** recommended for lightness, speed, easy scaling
- RabbitMQ heavier but more features
- Redis Streams fast but RAM-limited
- Kafka overkill for current scale

### Database
- **PostgreSQL** for efficiency, transactions, ledger support, rich queries
- MongoDB flexible but higher RAM usage
- Redis for temporary data only
- DynamoDB for managed scalable workloads

### Frontend
- **Next.js** for SSR/SSG, API Routes, productivity, performance

## Consequences

- FastAPI + PostgreSQL + NATS JetStream as core stack
- Worker should be separate from API
- Next.js for Landing and Dashboard

---

## Updates

**2026-03-12:** This ADR is still valid for the overall architecture direction. The framework choice has been reconfirmed as Python/FastAPI per ADR-0001.
