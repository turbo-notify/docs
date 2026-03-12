# API Changelog

> Public API contract history.

---

## 2026-03-12 (Current)

Contract reaffirmed as documentation-first and globally synchronized.

### Breaking updates

- Canonical path group renamed from `/numbers` to `/extra-numbers`.
- Canonical typing endpoint renamed from `/typing` to `/typing-indicator`.
- Per-number webhook configuration endpoint removed: `PUT /extra-numbers/{alias}/webhook`.
- Webhook configuration is now explicitly account-level (single webhook for main + extra numbers).
- `POST /reactions` and `POST /typing-indicator` use async `202 Accepted` completion via webhook.
- Webhook auth standardized with HMAC signature header (`X-Webhook-Signature`).
- Public contract authority kept in `/docs/api/*` even while runtime is in migration.
- Stack reference normalized to Python + FastAPI.

### Contract clarifications

- Rate-limit enforcement standardized as `rate-sync + redis` across API, workers, and webhook delivery dispatcher (no endpoint shape change).
- Tenant isolation and tier-based entitlement model documented for rate-limit behavior (`429` contract unchanged).

### Canonical endpoints

- `POST /messages`
- `GET /messages/{messageID}/status`
- `POST /extra-numbers`
- `GET /extra-numbers`
- `GET /extra-numbers/{alias}/status`
- `POST /extra-numbers/activate`
- `POST /extra-numbers/deactivate`
- `DELETE /extra-numbers/{alias}`
- `POST /reactions`
- `POST /typing-indicator`

---

## Related

- [API Overview](README.md)
- [Public API Governance](../reference/public-api-governance.md)
