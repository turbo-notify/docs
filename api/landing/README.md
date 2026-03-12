# Landing API Specification

> Internal API contract for the Turbo Notify marketing landing page.

---

## Scope

This directory defines the **internal API contract** for the Landing application.

- **Audience**: Landing frontend (React/Next.js), Marketing team
- **Visibility**: Internal (not exposed to external developers)
- **Current implementation**: Next.js API Routes + Server Actions
- **Future implementation**: Python + FastAPI BFF (see [ADR: BFF Migration](../../reference/decisions/2026-03-12-bff-migration-strategy.md))

---

## Architecture Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                      Landing (Next.js)                          │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              React Frontend                              │   │
│  │     (Marketing pages, Docs, Activation flow)            │   │
│  └────────────────────────┬────────────────────────────────┘   │
│                           │                                     │
│  ┌────────────────────────▼────────────────────────────────┐   │
│  │    Server Actions + API Routes (Current BFF)            │   │
│  │    Lead capture, Analytics, Verification                │   │
│  └────────────────────────┬────────────────────────────────┘   │
└───────────────────────────┼─────────────────────────────────────┘
                            │
            ┌───────────────┼───────────────┐
            ▼               ▼               ▼
      ┌──────────┐   ┌──────────┐   ┌──────────┐
      │   WAHA   │   │ PostgreSQL│   │  Sentry  │
      │(WhatsApp)│   │  (Leads)  │   │(Errors)  │
      └──────────┘   └──────────┘   └──────────┘
```

---

## Current API Routes

### Lead Management

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/lead/resend-code` | No | Resend verification code |

### Analytics & Infrastructure

| Method | Endpoint | Auth | Description |
|--------|----------|------|-------------|
| `POST` | `/api/activation-metrics` | No | Record activation funnel metrics |
| `POST` | `/api/analytics/track-event` | No | Track custom events |
| `GET` | `/api/metrics` | No | Prometheus metrics scraping |
| `GET` | `/api/sentry-example-api` | No | Sentry error test (dev only) |

---

## Endpoint Details

### POST /api/lead/resend-code

Resends the verification code to a WhatsApp number.

**Request**:
```http
POST /api/lead/resend-code
Content-Type: application/json
```

```json
{
  "whatsapp_number": "11999999999",
  "country_code": "+55"
}
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `whatsapp_number` | string | Yes | Regex: `^\d{8,15}$` |
| `country_code` | string | Yes | Regex: `^\+?\d{1,3}$` |

**Success Response**:
```http
200 OK
Content-Type: application/json
```

```json
{
  "success": true,
  "data": {
    "verification_code_resend_after": "2026-03-12T12:35:00Z",
    "verification_code_expires_at": "2026-03-12T12:44:00Z",
    "verification_code_length": 4,
    "message": "Code resent successfully"
  }
}
```

**Error Response**:
```json
{
  "success": false,
  "error": {
    "code": "lead_not_found",
    "message": "No lead found with this number"
  }
}
```

| Error Code | HTTP Status | Description |
|------------|-------------|-------------|
| `lead_not_found` | 404 | Lead not registered |
| `recipient_not_found` | 422 | WhatsApp number invalid |
| `timeout_error` | 504 | WAHA timeout |
| `service_unavailable` | 503 | WhatsApp unavailable |

---

### POST /api/activation-metrics

Records activation funnel steps for Prometheus monitoring.

**Request**:
```http
POST /api/activation-metrics
Content-Type: application/json
```

```json
{
  "step": "code_verified",
  "status": "success",
  "duration_ms": 2500
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `step` | string | Yes | Step identifier |
| `status` | string | Yes | `success`, `failed`, `skipped` |
| `duration_ms` | number | No | Duration in milliseconds |

**Response**: `200 OK`

```json
{
  "ok": true
}
```

**Prometheus Metrics**:
- `turbonotify_activation_step_total` (Counter)
- `turbonotify_activation_step_duration_seconds` (Histogram)
- `turbonotify_activation_in_progress` (Gauge)

---

### POST /api/analytics/track-event

Records custom analytics events from the client.

**Request**:
```http
POST /api/analytics/track-event
Content-Type: application/json
```

```json
{
  "name": "pricing_plan_clicked",
  "sessionId": "sess_abc123",
  "route": "/pricing",
  "details": {
    "plan": "professional",
    "source": "hero_cta"
  }
}
```

| Field | Type | Required | Validation |
|-------|------|----------|------------|
| `name` | string | Yes | 1-100 chars |
| `sessionId` | string | Yes | Session identifier |
| `route` | string | Yes | Current page route |
| `details` | object | No | Additional event data (sanitized) |

**Auto-captured**:
- IP address (from `x-forwarded-for`)
- User-Agent (from header)

**Response**:
```json
{
  "success": true
}
```

**Sanitization**: Characters `<>` are removed from string values.

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
# HELP turbonotify_activation_step_total Total activation steps
# TYPE turbonotify_activation_step_total counter
turbonotify_activation_step_total{step="phone_submitted",status="success"} 150
turbonotify_activation_step_total{step="code_verified",status="success"} 120
...
```

---

## Server Actions

Server Actions handle the lead activation flow. They execute on the server and are called directly from React components.

### Activation Flow

```
Phone → Code → Name → Email → Company → Sector → Referrals → Success
  │       │                                          │
  │       └── JWT token generated                    │
  │                                                  │
  └── Lead created, code sent via WAHA              └── Referrals sent via WAHA
```

---

### submitPhoneStep

Initiates lead capture with WhatsApp number.

**Input**:
```typescript
{
  country_code: string;      // e.g., "+55"
  whatsapp_number: string;   // e.g., "11999999999"
  sessionId: string;         // Analytics session
  referrer_code?: string;    // Optional referral code
}
```

**Output**:
```typescript
{
  success: true;
  data: {
    lead_id: string;
    status: "pending" | "verified" | "activated";
    country_code: string;
    whatsapp_number: string;
    name: string | null;
    verification_code_expires_at: string;    // ISO 8601
    verification_code_resend_after: string;  // ISO 8601
    verification_code_length: number;        // Default: 4
  }
}
```

**Behavior**:
1. Creates or updates lead in database
2. Generates verification code
3. Sends code via WAHA to WhatsApp
4. Records `phone_step_submitted` event

**Errors**:
- `recipient_not_found`: Invalid WhatsApp number
- `service_unavailable`: WAHA unavailable
- `timeout_error`: WAHA timeout

---

### submitCodeStep

Verifies the code and generates JWT access token.

**Input**:
```typescript
{
  whatsapp_number: string;
  country_code: string;
  code: string;              // Verification code
  code_length: number;       // Expected length
  sessionId: string;
}
```

**Output**:
```typescript
{
  success: true;
  data: {
    access_token: string;    // JWT for subsequent steps
    expires_at: string;      // Token expiration
    lead_id: string;
    verified: boolean;
    name: string | null;
    company: string | null;
    sector: string | null;
    email: string | null;
  }
}
```

**Behavior**:
1. Validates code against stored hash
2. Generates JWT access token
3. Records `code_verification_success` event

**Errors**:
- `invalid_code`: Code doesn't match
- `code_expired`: Code has expired
- `max_attempts_exceeded`: Too many failed attempts (includes `retry_after`)

**Verification Config** (via env):
- `VERIFICATION_CODE_LENGTH`: 4 (default)
- `VERIFICATION_CODE_EXPIRATION_SECONDS`: 600 (10 min)
- `VERIFICATION_CODE_RESEND_AFTER_SECONDS`: 60 (1 min)
- `VERIFICATION_CODE_MAX_ATTEMPTS`: 3

---

### submitNameStep

Updates lead name.

**Input**:
```typescript
{
  access_token: string;
  name: string;
  sessionId: string;
}
```

**Output**:
```typescript
{
  success: true;
  data: {
    access_token: string;
    name: string;
    lead_id: string;
  }
}
```

---

### submitEmailStep

Updates lead email.

**Input**:
```typescript
{
  access_token: string;
  email: string;           // Validated with regex
  sessionId: string;
}
```

**Output**:
```typescript
{
  success: true;
  data: {
    access_token: string;
    email: string;
    lead_id: string;
  }
}
```

**Email Validation**: `^[^@\s]+@[^@\s]+\.[^@\s]+$`

---

### submitCompanyStep

Updates lead company.

**Input**:
```typescript
{
  access_token: string;
  company: string;
  sessionId: string;
}
```

---

### submitSectorStep

Updates lead sector/industry.

**Input**:
```typescript
{
  access_token: string;
  sector: string;
  sessionId: string;
}
```

---

### submitReferralStep

Records referrals and sends invitations via WAHA.

**Input**:
```typescript
{
  access_token: string;
  referrals: Array<{
    country_code: string;
    whatsapp_number: string;
  }>;
  sessionId: string;
}
```

**Output**:
```typescript
{
  success: true;
  data: {
    referrals: Array<{
      country_code: string;
      whatsapp_number: string;
    }>
  }
}
```

**Behavior**:
1. Stores referrals in database
2. Sends invitation messages via WAHA
3. Records `step_referral_submitted` event (no PII)

---

### submitSuccessStep

Marks activation as complete.

**Input**:
```typescript
{
  access_token: string;
  sessionId: string;
}
```

**Output**:
```typescript
{
  success: true;
  data: {
    lead_id: string;
    status: "activated";
  }
}
```

**Behavior**:
1. Updates lead status to `activated`
2. Records `activation_completed` event

---

### submitContactLead

Submits contact form and sends message via WAHA.

**Input**:
```typescript
{
  session_id: string;
  name: string;
  email: string;
  whatsapp: string;
  company: string;
  country: string;
  message: string;
}
```

**Output**:
```typescript
{
  success: true;
}
```

**Behavior**:
1. Validates required fields
2. Stores in `contact_message` table
3. Sends message to `+5571999258283` via WAHA
4. Records `contact_form_submitted` event

---

### sendWaitMessage

Sends waitlist notification to lead.

**Input**:
```typescript
{
  access_token: string;
}
```

**Behavior**:
- Sends WhatsApp message informing lead is on waitlist

---

## Database Schema

**Schema**: `tn_marketing`

| Table | Description |
|-------|-------------|
| `lead` | Prospects with activation status, personal data, verification codes |
| `event` | User tracking events |
| `session_lead` | Many-to-many session/lead mapping |
| `referral` | Referrals made by leads |
| `contact_message` | Contact form submissions |

---

## External Integrations

### WAHA (WhatsApp HTTP API)

**Configuration**:
- `WAHA_BASE_URL`: API base URL
- `WAHA_API_KEY`: Authentication key
- `WAHA_SESSION`: Session identifier

**Gateways**:
| Gateway | Purpose |
|---------|---------|
| `WahaVerificationCodeSenderGateway` | Send verification codes |
| `WahaWelcomeMessageSenderGateway` | Send welcome messages |
| `WahaInvitationSenderGateway` | Send referral invitations |
| `WahaContactMessageSenderGateway` | Send contact form to team |
| `WahaWaitMessageSenderGateway` | Send waitlist notification |
| `WahaReferrerFollowUpGateway` | Follow-up with referrers |
| `WahaContactCardSenderGateway` | Send contact cards |

### Sentry

- Tunnel route: `/monitoring`
- Captures server and client errors
- HTTP request metrics

### Analytics

- **Google Analytics**: `GA_MEASUREMENT_ID`
- **Microsoft Clarity**: `CLARITY_PROJECT_ID`

---

## Planned BFF Migration

The following changes are planned for the Python + FastAPI migration:

| Current | Future |
|---------|--------|
| Server Actions | REST endpoints |
| Next.js API Routes | FastAPI routes |
| Inline validation | Pydantic models |
| Direct WAHA calls | Abstracted gateway |

**Endpoint Mapping**:

| Server Action | Future Endpoint |
|---------------|-----------------|
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

## Error Contract

Server Actions return standardized error responses:

```typescript
{
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, any>;
  }
}
```

### Common Error Codes

| Code | Description |
|------|-------------|
| `validation_error` | Invalid input data |
| `lead_not_found` | Lead doesn't exist |
| `recipient_not_found` | Invalid WhatsApp number |
| `invalid_code` | Verification code incorrect |
| `code_expired` | Verification code expired |
| `max_attempts_exceeded` | Too many verification attempts |
| `token_expired` | JWT access token expired |
| `service_unavailable` | WAHA or external service down |
| `timeout_error` | Operation timed out |

---

## Related

- [API Overview](../README.md)
- [BFF Migration Strategy](../../reference/decisions/2026-03-12-bff-migration-strategy.md)
- [Lead Activation Flow](../../architecture/lead-activation-flow.md)
