# ADR-0001: Python + FastAPI for All Backend Services

**Date:** 2026-03-12
**Status:** Accepted
**Supersedes:** 2025-08-25-go-vs-fastapi-vs-fastify.md (in api/docs/)

## Context

The Turbo Notify project initially had a Go-based API implementation (Go + Gin). A previous ADR recommended Go for performance reasons.

However, after reassessing the project requirements and team capabilities, a decision was made to standardize on Python for all backend services.

## Decision

**All backend services will use Python:**

| Service | Technology |
|---------|------------|
| Control Plane API | Python 3.13 + FastAPI |
| Session Workers | Python + asyncio |
| Orchestrator | Python |

**Frontend remains JavaScript/TypeScript:**

| Service | Technology |
|---------|------------|
| Dashboard | Next.js + TypeScript |
| Landing | Next.js + TypeScript |

## Rationale

### 1. Team Velocity
- Single language for all backend reduces context switching
- Easier to hire Python developers
- Shared libraries across services

### 2. Ecosystem
- Better WhatsApp Web libraries available in Python
- Rich async ecosystem for long-lived connections
- Strong FastAPI tooling (automatic OpenAPI, Pydantic validation)

### 3. Automacity Pattern
- Existing automacity project uses Python workers successfully
- Proven pattern for NATS + asyncio workers
- Reusable infrastructure code

### 4. Acceptable Performance Trade-off
- FastAPI performance (1k-5k req/s) is sufficient for MVP and early customers
- Can scale horizontally with multiple instances
- Premature optimization avoided

## Consequences

### Positive
- Simpler technology stack
- Faster development velocity
- Easier onboarding for new developers
- Code sharing between API and workers

### Negative
- Existing Go code in `/api/` must be migrated or discarded
- Previous ADR (2025-08-25-go-vs-fastapi-vs-fastify.md) is now obsolete
- Lower per-instance throughput (mitigated by horizontal scaling)

### Neutral
- Package management: Poetry (already planned)
- Testing: pytest
- Async framework: asyncio (native)

## Migration Plan

1. **Phase 1:** Create new Python FastAPI structure in `/api/`
2. **Phase 2:** Port existing endpoints from Go to Python
3. **Phase 3:** Remove Go code and dependencies
4. **Phase 4:** Implement workers following automacity pattern

## References

- Automacity workers: `/Users/zyc/dev/automacity/`
- FastAPI documentation: https://fastapi.tiangolo.com/
- Original Go ADR (superseded): `/api/docs/decision-history/2025-08-25-go-vs-fastapi-vs-fastify.md`
