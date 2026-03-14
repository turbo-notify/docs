# Security Documentation

**Last Updated:** 2026-03-14
**Status:** Active
**Reviewers:** Security Team, Backend Team

---

## Overview

This directory contains comprehensive security documentation for Turbo Notify's authentication and authorization architecture. The platform employs a dual-domain security model:

- **public-api (Control Plane)**: Customer-facing REST API authenticated via API Keys
- **dashboard-api (BFF)**: Admin dashboard backend authenticated via JWT sessions

Both APIs enforce strict multi-tenant isolation and share a unified audit trail.

---

## Security Domains

### Public API (Control Plane)

**Authentication**: API Key-based (Bearer tokens)
**Authorization**: Scope-based permissions
**Primary Users**: Customer applications, integrations
**Purpose**: Send messages, manage sessions, configure webhooks

### Dashboard API (BFF)

**Authentication**: JWT (access + refresh tokens)
**Authorization**: Role-based access control (RBAC)
**Primary Users**: Tenant administrators via web dashboard
**Purpose**: Manage API keys, view analytics, configure platform

---

## Documentation Index

### Core Architecture
- [01-authentication-architecture.md](01-authentication-architecture.md) - Complete authentication system design
- [04-authorization-model.md](04-authorization-model.md) - Authorization, scopes, roles, tenant isolation

### Implementation Guides
- [02-api-key-system.md](02-api-key-system.md) - API Key lifecycle management
- [03-jwt-session-management.md](03-jwt-session-management.md) - JWT tokens and session management
- [05-rate-limiting.md](05-rate-limiting.md) - Rate limiting strategy
- [06-audit-logging.md](06-audit-logging.md) - Security audit trail
- [07-token-security.md](07-token-security.md) - Token generation and storage best practices
- [09-security-headers.md](09-security-headers.md) - HTTP security headers

### Security Analysis
- [08-threat-model.md](08-threat-model.md) - Threat model and mitigations

---

## Security Principles

### 1. Defense in Depth
Multiple layers of security controls protect against threats:
- HTTPS enforced via HSTS headers
- Token hashing (SHA-256 for API keys, bcrypt for passwords)
- Rate limiting (distributed via Redis)
- Audit logging (immutable, append-only)

### 2. Least Privilege
- API keys support granular scopes (`messages:send`, `sessions:read`, etc.)
- Empty scopes default to full access (backward compatibility)
- Future: Role-based permissions in dashboard

### 3. Zero Trust
- Every query filters by `tenant_id` (repository-level enforcement)
- No implicit trust between tenants
- Session validation on every request

### 4. Security by Default
- Secure defaults in configuration
- Production validation (fail-fast on insecure settings)
- HSTS headers in production environment only

---

## Compliance Considerations

### GDPR
- **Right to Access**: Audit logs provide activity trail
- **Right to Erasure**: Audit logs can be anonymized (not deleted)
- **Data Minimization**: PII masked in logs (email hashing)
- **Retention**: 1-year audit log retention policy

### SOC 2 (Future)
- Audit trail for all security events
- Access control enforcement (scopes, roles)
- Encryption at rest and in transit
- Security incident response procedures

---

## Quick Navigation

### For New Developers
1. Start with [01-authentication-architecture.md](01-authentication-architecture.md) - understand overall design
2. Read [04-authorization-model.md](04-authorization-model.md) - learn tenant isolation rules
3. Review [02-api-key-system.md](02-api-key-system.md) or [03-jwt-session-management.md](03-jwt-session-management.md) based on which API you're working on

### For Implementers
1. [02-api-key-system.md](02-api-key-system.md) - API key validation middleware
2. [03-jwt-session-management.md](03-jwt-session-management.md) - JWT token handling
3. [06-audit-logging.md](06-audit-logging.md) - Audit event capture
4. [09-security-headers.md](09-security-headers.md) - Security headers middleware

### For Security Auditors
1. [08-threat-model.md](08-threat-model.md) - Threat analysis and mitigations
2. [06-audit-logging.md](06-audit-logging.md) - Audit trail capabilities
3. [04-authorization-model.md](04-authorization-model.md) - Access control model
4. [05-rate-limiting.md](05-rate-limiting.md) - DoS protection

---

## Critical Security Requirements

### MUST Follow
- ✅ **Tenant Isolation**: ALL repository queries MUST filter by `tenant_id`
- ✅ **HTTPS Only**: Production MUST enforce HTTPS (HSTS header)
- ✅ **Token Hashing**: NEVER store plaintext API keys or passwords
- ✅ **Audit Logging**: Log all authentication and authorization events
- ✅ **Rate Limiting**: Enforce rate limits on all public endpoints

### SHOULD Follow
- ⚠️ **Scope Enforcement**: Validate API key scopes before endpoint execution
- ⚠️ **Token Rotation**: Rotate refresh tokens on every use
- ⚠️ **Security Headers**: Apply security headers to all responses
- ⚠️ **Password Policy**: Enforce strong password requirements

### RECOMMENDED
- 💡 **Token Families**: Implement JTI-based token family tracking
- 💡 **Redis Blacklist**: Use Redis for revoked token tracking
- 💡 **Geo-Anomaly Detection**: Alert on unusual login locations
- 💡 **2FA**: Implement TOTP-based two-factor authentication

---

## Implementation Status

### ✅ Implemented

| Feature | public-api | dashboard-api | Notes |
|---------|-----------|---------------|-------|
| **API Key Entity** | ✅ | ✅ | Entity exists with SHA-256 hashing |
| **JWT Authentication** | N/A | ✅ | Access + refresh tokens |
| **Tenant Isolation** | ✅ | ✅ | Repository-level enforcement |
| **Rate Limiting** | ✅ | ✅ | rate-sync + Redis |
| **Password Hashing** | N/A | ✅ | bcrypt (12 rounds production) |

### ⚠️ Partially Implemented

| Feature | Status | Blocker |
|---------|--------|---------|
| **API Key Scopes** | public-api has entity field, dashboard-api missing | AccessKey entity missing `scopes` field |
| **Scope Enforcement** | Not enforced in endpoints | Middleware not implemented |
| **Audit Logging** | Not implemented | Database table missing |
| **Security Headers** | Not implemented | Middleware missing |

### ❌ Not Implemented (Roadmap)

| Feature | Priority | Target |
|---------|----------|--------|
| **Token Families** | High | Q2 2026 |
| **Redis Token Blacklist** | Medium | Q2 2026 |
| **TOTP 2FA** | Medium | Q3 2026 |
| **Advanced Rate Limiting** | Low | Q4 2026 |

---

## Known Gaps and Roadmap

### Current Limitations
- ⚠️ **Scope enforcement not yet implemented** in public-api
- ⚠️ **Token families not implemented** (refresh token reuse not detected)
- ⚠️ **Security headers middleware missing** in both APIs
- ⚠️ **Comprehensive audit logging incomplete**
- ⚠️ **Entity divergence**: `ApiKey` (public-api) has scopes, `AccessKey` (dashboard-api) does not

### Short-Term Roadmap (Q2 2026)
1. Implement API key scope enforcement middleware
2. Add security headers to all responses
3. Implement comprehensive audit logging
4. Deploy token rotation for refresh tokens

### Medium-Term Roadmap (Q3-Q4 2026)
1. Token families with JTI tracking
2. Redis-based token blacklist
3. Advanced rate limiting (per-key, tiered plans)
4. TOTP 2FA for dashboard login

---

## Change Log

| Date | Change | Author |
|------|--------|--------|
| 2026-03-14 | Initial security documentation | AI Assistant |

---

## Contact

For security concerns or questions:
- **Security Team**: security@turbonotify.com
- **Vulnerability Reports**: security-reports@turbonotify.com

For documentation updates:
- Create PR against `/docs/security/`
- Tag: `@security-team` for review
