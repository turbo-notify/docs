# Public API Governance

> Governance rules for Turbo Notify client-facing API contracts.

---

## Purpose

This document defines how API decisions with global impact are governed across:

- `landing-api`
- `dashboard-api`
- `public-api` (legacy)
- `landing-web`
- `dashboard-web`
- `docs`
- support/runbooks

In this phase, **documentation is authoritative**. Implementation may lag temporarily, but inconsistency across published docs is not allowed.

---

## Canonical Contract

As of **March 12, 2026**:

| Method | Endpoint |
|--------|----------|
| `POST` | `/messages` |
| `GET` | `/messages/{messageID}/status` |
| `POST` | `/extra-numbers` |
| `GET` | `/extra-numbers` |
| `GET` | `/extra-numbers/{alias}/status` |
| `POST` | `/extra-numbers/activate` |
| `POST` | `/extra-numbers/deactivate` |
| `DELETE` | `/extra-numbers/{alias}` |
| `POST` | `/reactions` |
| `POST` | `/typing-indicator` |

---

## Cross-Cutting Commitments

- Async operations that return `202 Accepted` must document webhook completion events.
- Webhook configuration is account-level only (single webhook for main and extra numbers).
- Shared rate limiting must be documented and enforced as `rate-sync + redis` in production (API, workers, webhook dispatcher).
- Rate-limit documentation must explicitly cover tenant isolation and tier (plan) entitlement behavior.

---

## Required Sources

| Source | Role |
|--------|------|
| `/docs/api/*.md` | Authoritative public contract |
| `/landing-web/src/content/docs/**/*.mdx` | Customer-facing narrative docs |
| `/landing-web/src/features/docs/lib/code-templates/*.ts` | Customer code examples |
| `/landing-api/docs/code-examples/*.ts` | API module code examples |
| `/landing-api` + `/dashboard-api` FastAPI apps | Runtime implementation |

---

## Mandatory Workflow

1. Update `/docs/api/*` first.
2. Update `landing-web` docs and examples.
3. Update `landing-api/docs/code-examples`.
4. Register change in `/docs/api/changelog.md`.
5. Align FastAPI runtime implementation.

A change is incomplete if one of the first four steps is missing.

---

## PR Checklist

- [ ] Endpoint names, payloads and status codes are consistent across `/docs/api`, `landing-web` docs, and code examples.
- [ ] Rate-limit behavior is documented with tenant + tiering rules and aligned `429` semantics.
- [ ] Changelog has exact date and breaking-change notes when applicable.
- [ ] FastAPI implementation impact is recorded when runtime still differs from docs.

---

## Related

- [API Overview](../api/README.md)
- [API Contract Alignment](../architecture/api-contract-alignment.md)
- [API Changelog](../api/changelog.md)
