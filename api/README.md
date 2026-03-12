# API Documentation

> Complete API specification for Turbo Notify platform.

---

## Overview

This directory contains API documentation organized by **audience**:

| Directory | Audience | Visibility | Description |
|-----------|----------|------------|-------------|
| [`public/`](public/README.md) | External developers | Public | Client-facing REST API for Turbo Notify integrations |
| [`dashboard/`](dashboard/README.md) | Dashboard frontend | Internal | Admin dashboard BFF endpoints |
| [`landing/`](landing/README.md) | Landing frontend | Internal | Marketing site BFF endpoints |

---

## Architecture

```
                          External Developers
                                  │
                                  ▼
                    ┌─────────────────────────┐
                    │     Public API          │
                    │  api.turbonotify.com    │
                    │  (Python + FastAPI)     │
                    └─────────────────────────┘
                                  │
        ┌─────────────────────────┼─────────────────────────┐
        │                         │                         │
        ▼                         ▼                         ▼
┌───────────────┐       ┌───────────────┐       ┌───────────────┐
│   Dashboard   │       │    Landing    │       │   Workers     │
│   Frontend    │       │   Frontend    │       │   (Python)    │
│   (Next.js)   │       │   (Next.js)   │       └───────────────┘
└───────┬───────┘       └───────┬───────┘
        │                       │
        ▼                       ▼
┌───────────────┐       ┌───────────────┐
│ Dashboard BFF │       │  Landing BFF  │
│  (FastAPI)*   │       │  (FastAPI)*   │
└───────────────┘       └───────────────┘

* Currently Next.js API Routes, migrating to FastAPI
  See ADR: BFF Migration Strategy
```

---

## API Categories

### Public API (`/public/`)

**For external developers integrating Turbo Notify.**

Core capabilities:
- **Messages**: Send WhatsApp messages, track delivery status
- **Extra Numbers**: Manage multiple sender numbers
- **Reactions**: Add/remove message reactions
- **Typing Indicator**: Show typing status
- **Webhooks**: Receive async event notifications

Base URL: `https://api.turbonotify.com`

[Full specification →](public/README.md)

---

### Dashboard API (`/dashboard/`)

**For the admin dashboard frontend.**

Core capabilities:
- **Authentication**: Login, session management
- **Profile**: User profile management
- **Access Keys**: API key CRUD
- **Numbers**: Main and extra number management
- **Billing**: Plans, invoices, payments
- **Messages**: Message history and stats
- **Webhooks**: Configuration and delivery history
- **Overview**: Dashboard KPIs and analytics

Base URL (target): `https://dashboard-bff.turbonotify.com`

[Full specification →](dashboard/README.md)

---

### Landing API (`/landing/`)

**For the marketing landing page frontend.**

Core capabilities:
- **Lead Activation**: Multi-step onboarding flow
- **Analytics**: Event tracking
- **Contact**: Contact form submission
- **Waitlist**: Waitlist management

Base URL (target): `https://landing-bff.turbonotify.com`

[Full specification →](landing/README.md)

---

## Technology Stack

| Component | Stack | Status |
|-----------|-------|--------|
| **Public API** | Python + FastAPI | Target |
| **Dashboard BFF** | Next.js API Routes → FastAPI | Migrating |
| **Landing BFF** | Next.js Server Actions → FastAPI | Migrating |
| **Rate Limiting** | rate-sync + Redis | Production |

---

## BFF Migration

Dashboard and Landing currently use Next.js built-in API capabilities. We are migrating to separate Python + FastAPI BFF services for:

- **Deployment independence**: Scale frontend and backend separately
- **Stack unification**: All backend services in Python
- **Team velocity**: Parallel frontend/backend development

See [ADR: BFF Migration Strategy](../reference/decisions/2026-03-12-bff-migration-strategy.md) for details.

---

## Synchronization Rules

### Public API Changes

Any change to public API contract must update:

1. `/docs/api/public/*` - Source of truth
2. `/landing/src/content/docs/**/*` - User-facing docs
3. `/docs/api/public/changelog.md` - Change history

### Internal API Changes

Dashboard and Landing API changes should update:

1. Respective documentation in `/docs/api/{dashboard,landing}/`
2. Architecture docs if significant

---

## Related

- [Public API Governance](../reference/public-api-governance.md)
- [API Contract Alignment](../architecture/api-contract-alignment.md)
- [API Standards](../standards/api-standards.md)
- [BFF Migration Strategy](../reference/decisions/2026-03-12-bff-migration-strategy.md)
