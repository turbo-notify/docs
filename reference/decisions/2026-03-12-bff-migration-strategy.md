# ADR: BFF Migration Strategy - Next.js to Python + FastAPI

- **Status**: Accepted
- **Date**: 2026-03-12
- **Authors**: Architecture Team
- **Stakeholders**: Engineering, DevOps, Product

---

## Context

Currently, both **Dashboard** and **Landing** applications use Next.js with built-in API routes and Server Actions as their Backend-for-Frontend (BFF) layer.

```
Current Architecture:
┌─────────────────────────────────────────┐
│          Dashboard (Next.js)            │
│  ┌───────────────────────────────────┐  │
│  │     React + API Routes + DB       │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│           Landing (Next.js)             │
│  ┌───────────────────────────────────┐  │
│  │  React + Server Actions + WAHA    │  │
│  └───────────────────────────────────┘  │
└─────────────────────────────────────────┘
```

This tight coupling presents several challenges:

1. **Deployment coupling**: Frontend and backend changes require full redeployment
2. **Scaling constraints**: Cannot scale API layer independently from SSR/SSG
3. **Technology lock-in**: Backend logic tied to Node.js/Next.js runtime
4. **Code organization**: Business logic mixed with presentation concerns
5. **Team boundaries**: Difficult to parallelize frontend and backend work
6. **Testing complexity**: E2E tests required for API validation
7. **Stack fragmentation**: Control Plane API uses Python + FastAPI, creating skill divergence

---

## Decision

**Migrate BFF layers from Next.js API Routes/Server Actions to Python + FastAPI as separate services.**

```
Target Architecture:
┌─────────────────────────────────────────┐
│       Dashboard (Next.js - Frontend)    │
│  ┌───────────────────────────────────┐  │
│  │     React + SSR/SSG only          │  │
│  └────────────────┬──────────────────┘  │
└───────────────────┼─────────────────────┘
                    │ HTTP
                    ▼
┌─────────────────────────────────────────┐
│          Dashboard BFF (FastAPI)        │
│  ┌───────────────────────────────────┐  │
│  │  Auth, Profile, Billing, etc.     │  │
│  └────────────────┬──────────────────┘  │
└───────────────────┼─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│        Control Plane API (FastAPI)      │
└─────────────────────────────────────────┘
```

```
┌─────────────────────────────────────────┐
│        Landing (Next.js - Frontend)     │
│  ┌───────────────────────────────────┐  │
│  │    React + SSR/SSG + MDX only     │  │
│  └────────────────┬──────────────────┘  │
└───────────────────┼─────────────────────┘
                    │ HTTP
                    ▼
┌─────────────────────────────────────────┐
│           Landing BFF (FastAPI)         │
│  ┌───────────────────────────────────┐  │
│  │  Leads, Analytics, Contact, etc.  │  │
│  └────────────────┬──────────────────┘  │
└───────────────────┼─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│     WAHA (WhatsApp) + PostgreSQL        │
└─────────────────────────────────────────┘
```

---

## Rationale

### Why Separate BFF?

| Concern | Monolith (Current) | Separated BFF |
|---------|-------------------|---------------|
| **Deployment** | Full redeploy | Independent releases |
| **Scaling** | Scale together | Scale API independently |
| **Technology** | Node.js only | Best tool for job |
| **Testing** | E2E required | Unit + integration |
| **Teams** | Sequential work | Parallel development |

### Why Python + FastAPI?

1. **Stack unification**: Control Plane API already uses Python + FastAPI
2. **Team expertise**: Python is the primary backend language
3. **Library ecosystem**: Rich async libraries, NATS client, SQLAlchemy
4. **Performance**: FastAPI is comparable to Node.js for I/O bound work
5. **Type safety**: Pydantic provides runtime validation
6. **Documentation**: Auto-generated OpenAPI specs

### Why NOT Keep Next.js API Routes?

1. **Cold start overhead**: Serverless Next.js API routes have higher cold starts
2. **Limited async patterns**: Node.js async is less ergonomic than Python asyncio
3. **Deployment complexity**: Mixing SSR and API in same deployment unit
4. **Monitoring gaps**: Different observability patterns than Control Plane

---

## Consequences

### Positive

- **Unified backend stack**: All backend services in Python + FastAPI
- **Independent scaling**: BFF can scale based on API load, not page views
- **Cleaner frontends**: Next.js focuses purely on presentation
- **Better testability**: BFF APIs can be tested in isolation
- **Shared libraries**: Reuse Control Plane patterns (DI, error handling, etc.)
- **Team velocity**: Parallel development of frontend and backend

### Negative

- **Additional services**: Two more services to deploy and monitor
- **Network latency**: Extra hop between frontend and BFF
- **Migration effort**: Rewrite existing API routes and Server Actions
- **CORS management**: Need to handle cross-origin requests
- **Session management**: Token handling across domains

### Mitigations

| Risk | Mitigation |
|------|------------|
| Extra latency | Deploy BFF in same region, use HTTP/2 |
| Migration complexity | Incremental migration, feature flags |
| CORS issues | Centralized CORS config, same-domain deployment |
| Session complexity | JWT with refresh tokens, HttpOnly cookies |

---

## Implementation Plan

### Phase 1: Infrastructure Setup (Week 1-2)

1. Create `dashboard-bff/` and `landing-bff/` project structures
2. Set up FastAPI with shared patterns from Control Plane
3. Configure deployment pipelines
4. Set up monitoring (Prometheus, Sentry)

### Phase 2: Dashboard BFF Migration (Week 3-6)

Priority order (based on current mock implementations):

1. **Auth endpoints** - Login, logout, session management
2. **Profile endpoints** - User profile CRUD
3. **Access Keys** - API key management
4. **Numbers** - Main and extra number management
5. **Billing** - Plans, invoices, payment methods
6. **Webhooks** - Configuration and delivery history
7. **Messages** - Message history and stats
8. **Overview** - Dashboard KPIs and charts

### Phase 3: Landing BFF Migration (Week 7-9)

Priority order:

1. **Lead activation flow** - Phone, code, profile steps
2. **Contact form** - Form submission and WAHA integration
3. **Analytics** - Event tracking
4. **Waitlist** - Waitlist notifications

### Phase 4: Frontend Adaptation (Week 10-12)

1. Update Dashboard to call BFF endpoints
2. Update Landing to call BFF endpoints
3. Remove deprecated API routes and Server Actions
4. Update deployment configurations

### Phase 5: Cleanup (Week 13)

1. Remove old Next.js API routes
2. Remove Server Actions
3. Update documentation
4. Archive migration artifacts

---

## API Contract Changes

### Dashboard BFF Base URL

```
Development: http://localhost:8001
Production:  https://dashboard-bff.turbonotify.com
```

### Landing BFF Base URL

```
Development: http://localhost:8002
Production:  https://landing-bff.turbonotify.com
```

### Authentication Flow

```
Current (Server Action):
1. Client calls Server Action
2. Server Action validates and creates session
3. Session stored in server memory/cookie

Target (BFF):
1. Client calls POST /bff/auth/login
2. BFF validates credentials
3. BFF returns JWT access_token + refresh_token
4. Client stores tokens (HttpOnly cookie or secure storage)
5. Client includes Authorization header in subsequent requests
```

---

## Endpoint Migration Map

### Dashboard

| Current | Target |
|---------|--------|
| `POST /api/activation-metrics` | `POST /bff/metrics/activation` |
| `GET /api/metrics` | `GET /bff/metrics/prometheus` |
| `GET /api/public-settings/` | `GET /bff/config/public` |
| Mock store operations | REST endpoints (see dashboard/README.md) |

### Landing

| Current | Target |
|---------|--------|
| `POST /api/lead/resend-code` | `POST /bff/leads/resend-code` |
| `POST /api/activation-metrics` | `POST /bff/metrics/activation` |
| `POST /api/analytics/track-event` | `POST /bff/analytics/events` |
| `GET /api/metrics` | `GET /bff/metrics/prometheus` |
| `submitPhoneStep` | `POST /bff/leads/phone` |
| `submitCodeStep` | `POST /bff/leads/verify` |
| `submitNameStep` | `PATCH /bff/leads/profile` |
| `submitEmailStep` | `PATCH /bff/leads/profile` |
| `submitCompanyStep` | `PATCH /bff/leads/profile` |
| `submitSectorStep` | `PATCH /bff/leads/profile` |
| `submitReferralStep` | `POST /bff/leads/referrals` |
| `submitSuccessStep` | `POST /bff/leads/activate` |
| `submitContactLead` | `POST /bff/contact` |
| `sendWaitMessage` | `POST /bff/leads/waitlist` |

---

## Project Structure

### Dashboard BFF

```
dashboard-bff/
├── src/
│   ├── main.py                    # FastAPI app entry
│   ├── core/
│   │   ├── domain/
│   │   │   ├── entities/
│   │   │   ├── repositories/
│   │   │   └── usecases/
│   │   └── infra/
│   │       ├── adapters/
│   │       └── factories/
│   ├── features/
│   │   ├── auth/
│   │   ├── profile/
│   │   ├── access_keys/
│   │   ├── numbers/
│   │   ├── billing/
│   │   ├── webhooks/
│   │   ├── messages/
│   │   └── overview/
│   └── shared/
│       ├── middleware/
│       └── utils/
├── tests/
├── pyproject.toml
└── Dockerfile
```

### Landing BFF

```
landing-bff/
├── src/
│   ├── main.py
│   ├── core/
│   │   ├── domain/
│   │   └── infra/
│   ├── features/
│   │   ├── leads/
│   │   ├── contact/
│   │   ├── analytics/
│   │   └── waitlist/
│   └── shared/
├── tests/
├── pyproject.toml
└── Dockerfile
```

---

## Observability

Both BFFs will follow the same observability patterns as Control Plane:

### Prometheus Metrics

- Request latency histograms
- Request counters by endpoint and status
- Active connections gauge
- Custom business metrics

### Sentry Integration

- Error tracking with context
- Performance monitoring
- Release tracking

### Logging

- Structured JSON logs
- Correlation IDs
- Request tracing

---

## Rollback Plan

If critical issues arise during migration:

1. **Feature flags**: Toggle between old and new endpoints
2. **Parallel operation**: Keep old endpoints running during transition
3. **Quick rollback**: Revert frontend to call old endpoints
4. **Data consistency**: Both systems read from same database

---

## Success Criteria

- [ ] All Dashboard API routes migrated to BFF
- [ ] All Landing Server Actions migrated to BFF
- [ ] No functionality regression
- [ ] Response times within 50ms of current implementation
- [ ] 99.9% uptime maintained during migration
- [ ] Zero data loss
- [ ] Monitoring coverage equivalent to current state

---

## Related Documents

- [Dashboard API Specification](../../api/dashboard/README.md)
- [Landing API Specification](../../api/landing/README.md)
- [Control Plane Architecture](../../architecture/ecosystem-architecture.md)
- [ADR: Python + FastAPI Stack](0001-python-fastapi-stack.md)
