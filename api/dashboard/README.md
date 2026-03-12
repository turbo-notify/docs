# Dashboard API Specification

> Internal API contract for the Turbo Notify admin dashboard.

---

## Scope

This directory defines the **internal API contract** for the Dashboard application.

- **Audience**: Dashboard frontend (React/Next.js)
- **Visibility**: Internal (not exposed to external developers)
- **Current implementation**: Next.js API Routes (BFF pattern)
- **Future implementation**: Python + FastAPI BFF (see [ADR: BFF Migration](../../reference/decisions/2026-03-12-bff-migration-strategy.md))

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                      Dashboard (Next.js)                    │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              React Frontend                          │   │
│  │     (shadcn/ui, Tailwind, Framer Motion)            │   │
│  └────────────────────────┬────────────────────────────┘   │
│                           │                                 │
│  ┌────────────────────────▼────────────────────────────┐   │
│  │           Next.js API Routes (Current BFF)          │   │
│  │    /api/activation-metrics, /api/metrics, etc.      │   │
│  └────────────────────────┬────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │    Control Plane API        │
              │    (Python + FastAPI)       │
              │    /api/turbonotify.com     │
              └─────────────────────────────┘
```

---

## Current API Routes

### Infrastructure Endpoints

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/activation-metrics` | No | Record activation funnel metrics |
| `GET` | `/api/metrics` | No | Prometheus metrics scraping |
| `GET` | `/api/public-settings/` | No | Public configuration (GA, Clarity, Sentry) |
| `GET` | `/api/sentry-example-api` | No | Sentry error test (dev only) |
| `GET` | `/sitemap.xml` | No | SEO sitemap |

---

## Endpoint Details

### POST /api/activation-metrics

Records user activation funnel steps for Prometheus monitoring.

**Request**:
```http
POST /api/activation-metrics
Content-Type: application/json
```

```json
{
  "step": "phone_verified",
  "status": "success",
  "duration": 1500
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `step` | string | Yes | Activation step identifier |
| `status` | string | Yes | Step status (`success`, `failed`, `skipped`) |
| `duration` | number | No | Duration in milliseconds |

**Response**: `204 No Content`

**Prometheus Metrics**:
- `turbo_notify_activation_funnel` (Counter)

---

### GET /api/metrics

Exposes Prometheus metrics for infrastructure monitoring.

**Request**:
```http
GET /api/metrics
```

**Response**:
```http
200 OK
Content-Type: text/plain; version=0.0.4
```

```text
# HELP turbo_notify_activation_funnel Activation funnel metrics
# TYPE turbo_notify_activation_funnel counter
turbo_notify_activation_funnel{step="phone_verified",status="success"} 42
...
```

**Headers**:
- `Cache-Control: no-store`

---

### GET /api/public-settings/

Returns public configuration for client-side initialization.

**Request**:
```http
GET /api/public-settings/
```

**Response**:
```http
200 OK
Content-Type: application/json
```

```json
{
  "LANDING_PAGE_BASE_URL": "https://turbonotify.com",
  "GA_MEASUREMENT_ID": "G-XXXXXXXXXX",
  "CLARITY_PROJECT_ID": "xxxxxxxxxx",
  "SENTRY_DSN": "https://xxx@sentry.io/xxx",
  "ANALYZE": "false"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `LANDING_PAGE_BASE_URL` | string | Landing page base URL |
| `GA_MEASUREMENT_ID` | string? | Google Analytics ID |
| `CLARITY_PROJECT_ID` | string? | Microsoft Clarity project |
| `SENTRY_DSN` | string? | Sentry DSN for error tracking |
| `ANALYZE` | string | Bundle analyzer flag |

---

## Planned BFF API Contract

The following endpoints represent the target BFF contract after migration to Python + FastAPI.

### Authentication

| Method | Endpoint | Description |
|--------|----------|-------------|
| `POST` | `/bff/auth/login` | Email/password login |
| `POST` | `/bff/auth/login/whatsapp` | Request WhatsApp verification |
| `POST` | `/bff/auth/login/whatsapp/verify` | Verify WhatsApp code |
| `POST` | `/bff/auth/logout` | Logout current session |
| `POST` | `/bff/auth/password/forgot` | Request password reset |
| `POST` | `/bff/auth/password/reset` | Reset password with token |
| `GET` | `/bff/auth/session` | Get current session info |

### Profile

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/profile` | Get user profile |
| `PATCH` | `/bff/profile` | Update user profile |
| `POST` | `/bff/profile/avatar` | Upload avatar |

### Account Settings

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/account/settings` | Get account settings |
| `PATCH` | `/bff/account/settings` | Update account settings |
| `GET` | `/bff/account/team` | List team members |
| `POST` | `/bff/account/team/invite` | Invite team member |
| `DELETE` | `/bff/account/team/{memberId}` | Remove team member |

### Access Keys

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/access-keys` | List access keys |
| `POST` | `/bff/access-keys` | Create access key |
| `DELETE` | `/bff/access-keys/{keyId}` | Revoke access key |
| `PATCH` | `/bff/access-keys/{keyId}` | Update key metadata |

### Numbers

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/numbers` | List all numbers (main + extra) |
| `GET` | `/bff/numbers/main` | Get main number details |
| `GET` | `/bff/numbers/extra` | List extra numbers |
| `POST` | `/bff/numbers/extra` | Add extra number |
| `DELETE` | `/bff/numbers/extra/{alias}` | Remove extra number |

### Messages

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/messages` | List messages (paginated) |
| `GET` | `/bff/messages/{messageId}` | Get message details |
| `GET` | `/bff/messages/stats` | Message statistics |

### Webhooks

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/webhooks/config` | Get webhook configuration |
| `PUT` | `/bff/webhooks/config` | Update webhook configuration |
| `GET` | `/bff/webhooks/deliveries` | List delivery attempts |
| `POST` | `/bff/webhooks/test` | Send test webhook |

### Billing

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/billing/plan` | Get current plan |
| `GET` | `/bff/billing/plans` | List available plans |
| `POST` | `/bff/billing/plan/change` | Request plan change |
| `GET` | `/bff/billing/invoices` | List invoices |
| `GET` | `/bff/billing/invoices/{invoiceId}` | Get invoice details |
| `GET` | `/bff/billing/payment-methods` | List payment methods |
| `POST` | `/bff/billing/payment-methods` | Add payment method |
| `DELETE` | `/bff/billing/payment-methods/{methodId}` | Remove payment method |

### Overview/Analytics

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/overview` | Dashboard overview data |
| `GET` | `/bff/overview/charts` | Chart data for dashboard |
| `GET` | `/bff/overview/kpis` | KPI metrics |

### Support

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/bff/support/tickets` | List support tickets |
| `POST` | `/bff/support/tickets` | Create support ticket |
| `GET` | `/bff/support/tickets/{ticketId}` | Get ticket details |
| `POST` | `/bff/support/tickets/{ticketId}/reply` | Reply to ticket |

---

## Authentication Model

### Current State (Mock)

Authentication is currently stubbed for development:

```typescript
// src/shared/lib/mock/auth.ts
export const isAuthenticated = true;
```

### Target State

JWT-based authentication with:

- Access token (short-lived, 15 min)
- Refresh token (long-lived, 7 days)
- Secure HttpOnly cookies

**Headers**:
```http
Authorization: Bearer <access_token>
Cookie: refresh_token=<refresh_token>
```

---

## Error Contract

BFF endpoints follow standardized error responses:

```json
{
  "error": {
    "code": "invalid_credentials",
    "message": "Email or password is incorrect",
    "details": {}
  }
}
```

### Common Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `unauthorized` | 401 | Missing or invalid token |
| `forbidden` | 403 | Insufficient permissions |
| `not_found` | 404 | Resource not found |
| `validation_error` | 422 | Invalid request data |
| `rate_limited` | 429 | Too many requests |
| `internal_error` | 500 | Server error |

---

## Rate Limiting

BFF endpoints are rate-limited per user:

| Endpoint Category | Limit |
|-------------------|-------|
| Authentication | 5 req/min |
| Read operations | 100 req/min |
| Write operations | 30 req/min |
| File uploads | 10 req/min |

---

## Observability

### Sentry Integration

- DSN configured via `SENTRY_DSN` env var
- Environment via `SENTRY_ENV`
- Route wrapping with `wrapRouteHandlerWithSentry`
- Tunnel route: `/monitoring`

### Prometheus Metrics

- Endpoint: `GET /api/metrics`
- Metrics: activation funnel, route latency

### Analytics

- Google Analytics: `GA_MEASUREMENT_ID`
- Microsoft Clarity: `CLARITY_PROJECT_ID`

---

## Related

- [API Overview](../README.md)
- [BFF Migration Strategy](../../reference/decisions/2026-03-12-bff-migration-strategy.md)
- [Authentication Flow](../../architecture/authentication-flow.md)
