# API Contract Alignment

> Cross-module alignment for Turbo Notify public API.

---

## Scope

This alignment covers:

- canonical contract docs in `/docs/api`
- customer docs in `landing-web`
- code examples in `landing-web` and `landing-api/docs`
- FastAPI runtime in `landing-api` and `dashboard-api`

Snapshot date: **March 12, 2026**.

---

## Canonical Public Endpoints

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

## Source Layers

| Layer | Source |
|-------|--------|
| Contract authority | `/docs/api/*.md` |
| User docs | `/landing-web/src/content/docs/**/*.mdx` |
| Landing code samples | `/landing-web/src/features/docs/lib/code-templates/*.ts` |
| API module code samples | `/landing-api/docs/code-examples/*.ts` |
| Runtime implementation | FastAPI in `/landing-api` and `/dashboard-api` |

---

## Alignment Policy

1. One canonical contract only.
2. Docs are allowed to lead runtime during development.
3. Runtime must converge to documented contract.
4. Any contract change requires synchronized docs/examples update and changelog entry.

---

## Related

- [Public API Governance](../reference/public-api-governance.md)
- [API Overview](../api/README.md)
- [API Changelog](../api/changelog.md)
