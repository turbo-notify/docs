# Public API Specification

> Source of truth for the Turbo Notify client-facing REST API.

---

## Scope

This directory defines the public contract consumed by Turbo Notify clients.

- This contract has **global impact** (`api`, `landing`, `dashboard`, support, runbooks).
- In this project phase, **documentation is authoritative** and may lead implementation.
- Breaking changes are allowed before production, but cross-module inconsistency is not.

Snapshot date: **March 12, 2026**.

---

## Official Stack

- Backend API stack: **Python + FastAPI**
- Shared rate-limiting stack: **rate-sync + redis** (production required)
- Public contract base URL: `https://api.turbonotify.com`

---

## Canonical Endpoints

| Category | Endpoints |
|----------|-----------|
| Messages | `POST /messages`, `GET /messages/{messageID}/status` |
| Extra Numbers | `POST /extra-numbers`, `GET /extra-numbers`, `GET /extra-numbers/{alias}/status`, `POST /extra-numbers/activate`, `POST /extra-numbers/deactivate`, `DELETE /extra-numbers/{alias}` |
| Reactions | `POST /reactions` |
| Typing Indicator | `POST /typing-indicator` |

Webhook configuration is account-level (single URL/secret in dashboard) and applies to the main sender plus all extra numbers.
Async endpoints (`POST /messages`, `POST /reactions`, `POST /typing-indicator`) return `202 Accepted` and complete via webhook events.
Rate-limit behavior is defined by tenant isolation plus contracted tier entitlement.

---

## Common Error Shape

```json
{
  "error": "internal_error"
}
```

---

## Documents

| Document | Description |
|----------|-------------|
| [Authentication](authentication.md) | Access key requirements and auth behavior |
| [Messages](messages.md) | Send and track messages |
| [Extra Numbers](extra-numbers.md) | Multi-sender number lifecycle |
| [Reactions](reactions.md) | Add and remove reactions |
| [Typing Indicator](typing-indicator.md) | Start/stop typing signal |
| [Webhooks](webhooks.md) | Event delivery contract and security |
| [Rate Limits](rate-limits.md) | Usage policy and 429 behavior |
| [Errors](errors.md) | Complete error reference |
| [Changelog](changelog.md) | Contract history and breaking changes |

---

## Synchronization Rule

Any contract change must update, in the same delivery window:

1. `/docs/api/*`
2. `/landing/src/content/docs/**/*`
3. `/landing/src/features/docs/lib/code-templates/*`
4. `/api/docs/code-examples/*`
5. `/docs/api/changelog.md`

Runtime implementation in `/api` must be aligned as soon as possible after docs updates.

---

## Related

- [Public API Governance](../reference/public-api-governance.md)
- [API Contract Alignment](../architecture/api-contract-alignment.md)
