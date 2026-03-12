# ADR: rate-sync as Mandatory Rate-Limiting Engine

**Date:** 2026-03-12
**Status:** Accepted

## Context

Turbo Notify needs consistent rate-limit behavior across Control Plane API, workers, and webhook dispatcher.

When different modules implement custom counters or local throttling logic, limits become inconsistent across instances and hard to operate safely.

## Decision

All rate-limit and concurrency-limit controls must use **rate-sync** as the shared engine:

- PyPI package: `rate-sync` (`https://pypi.org/project/rate-sync/`)
- Development source used by team: `/Users/zyc/dev/rate-sync/python`

### Mandatory Scope

| Area | Requirement |
|------|-------------|
| API ingress | Enforce tenant/key/endpoint limits using `rate-sync` |
| Worker outbound | Enforce provider and destination limits using `rate-sync` |
| Webhook dispatcher | Enforce delivery throughput/concurrency using `rate-sync` |
| Background jobs | Enforce queue/job throttling using `rate-sync` |
| Tenant isolation | Enforce per-tenant fairness to prevent noisy-neighbor saturation |
| Tiering (plan) | Map contracted plan tiers to baseline limiter profiles |

### Implementation Rules

1. Production environments must use `rate-sync` with `redis` backend.
2. In-memory limiter backend is allowed only for local development and isolated tests.
3. Limiter definitions must be centralized and versioned (no ad-hoc limiter IDs in random modules).
4. New custom throttling logic outside `rate-sync` is not allowed.
5. Tier profiles and tenant-to-tier mapping must be documented and versioned with contract updates.

## Consequences

### Positive

- Consistent limits across API, worker, and webhook delivery flows
- Predictable operational behavior under load
- Reusable limiter patterns (tenant, sender, endpoint, provider)
- Clear entitlement boundaries across plan tiers

### Trade-off

- Shared dependency for all services that enforce limits
- Requires disciplined limiter naming/config governance

## Related

- [Rate Limits](../../api/rate-limits.md)
- [Ecosystem Architecture](../../architecture/ecosystem-architecture.md)
- [Worker Architecture](../../architecture/worker-architecture.md)
- [Project Guidelines](../project-guidelines.md)
