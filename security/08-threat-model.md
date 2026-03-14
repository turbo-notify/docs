# Threat Model and Mitigations

**Last Updated:** 2026-03-14
**Status:** Active

---

## STRIDE Analysis

| Threat Category | Example | Mitigation |
|----------------|---------|------------|
| **Spoofing** | Attacker steals API key | SHA-256 hashing, HTTPS enforcement |
| **Tampering** | Modify JWT claims | JWT signature verification (HS256) |
| **Repudiation** | Deny malicious actions | Immutable audit logs |
| **Information Disclosure** | Leak tenant data | Tenant isolation (filter by tenant_id) |
| **Denial of Service** | Flood API with requests | Rate limiting (100 req/min per tenant) |
| **Elevation of Privilege** | Access other tenant's data | Repository-level tenant_id enforcement |

---

## Specific Threats and Mitigations

### T1: API Key Compromise

**Threat**: Attacker obtains API key via phishing, code leak, or man-in-the-middle attack

**Impact**:
- Send unauthorized messages
- Access tenant's data
- Exhaust rate limits

**Mitigations**:
- ✅ HTTPS enforced (HSTS header)
- ✅ Key rotation policy (90 days recommended)
- ✅ Scope-based permissions (least privilege)
- ✅ Monitor failed auth attempts
- ✅ Revocation API

**Detection**:
- Spike in failed authentication attempts
- Unusual IP addresses or geo-locations
- High request volume anomaly

---

### T2: JWT Token Theft

**Threat**: XSS attack steals access token from localStorage

**Impact**: Session hijacking, impersonate user

**Mitigations**:
- ✅ HttpOnly cookies (prevent JavaScript access)
- ✅ Short access token expiry (15 minutes)
- ✅ CSP headers (block inline scripts)
- ⚠️ Refresh token rotation (future)

**Detection**:
- Session used from different IP/device
- Multiple concurrent sessions from different locations

---

### T3: Brute Force Login

**Threat**: Attacker tries many passwords to guess credentials

**Impact**: Account takeover

**Mitigations**:
- ✅ Rate limiting (5 attempts / 15 min per IP + email)
- ✅ Account lockout (30 min after 5 failed attempts)
- ✅ Strong password policy (12+ chars, mixed case, digits, special)
- ⚠️ CAPTCHA (future)

**Detection**:
- Audit log shows repeated failed login attempts
- Alert on 5+ failures from same IP

---

### T4: Tenant Isolation Bypass

**Threat**: Bug in code allows accessing another tenant's data

**Impact**: Data breach, GDPR violation, loss of customer trust

**Mitigations**:
- ✅ Repository pattern enforces `tenant_id` filtering
- ✅ Code review checklist (verify tenant_id in queries)
- ✅ Integration tests verify isolation
- ✅ Type system forces explicit tenant_id passing

**Detection**:
- Audit logs show cross-tenant access patterns
- Anomaly detection on data access

---

### T5: SQL Injection

**Threat**: Attacker injects SQL code via user input

**Impact**: Data breach, data corruption, unauthorized access

**Mitigations**:
- ✅ SQLAlchemy ORM (parameterized queries)
- ✅ Input validation (Pydantic schemas)
- ✅ Least privilege database user
- ✅ No raw SQL without parameterization

**Detection**:
- WAF logs show SQL injection patterns
- Database query monitoring for suspicious queries

---

### T6: DoS via Rate Limit Exhaustion

**Threat**: Attacker floods endpoint, exhausts tenant's rate limit

**Impact**: Legitimate users cannot access service

**Mitigations**:
- ✅ Rate limiting per IP (pre-authentication)
- ✅ Rate limiting per tenant (post-authentication)
- ⚠️ CDN/WAF protection (future)
- ⚠️ Advanced rate limiting (tiered plans)

**Detection**:
- Spike in 429 (Too Many Requests) responses
- Monitoring alerts on rate limit threshold

---

## Security Testing

### Penetration Testing

- **Frequency**: Annual
- **Scope**: External penetration test by certified firm
- **Focus**: Authentication, authorization, data leakage

### Vulnerability Scanning

- **Frequency**: Monthly
- **Tools**: Automated security scanners
- **Scope**: Dependencies, infrastructure, code

### Dependency Auditing

- **Frequency**: Every commit (CI/CD pipeline)
- **Tools**: `pip-audit`, `safety`, `snyk`
- **Action**: Block deployment on high-severity vulnerabilities

---

## Incident Response Playbook

### API Key Compromise

1. **Immediately revoke** compromised key
2. **Review audit logs** for unauthorized activity
3. **Notify customer** of suspected compromise
4. **Generate new key** if needed
5. **Investigate** how key was compromised
6. **Implement additional controls** if systemic issue

### Token Theft

1. **Revoke session** (set status to "revoked")
2. **Force logout** on all devices
3. **Notify user** via email
4. **Require password reset** if credentials suspected
5. **Review audit logs** for malicious activity

### Data Breach

1. **Isolate affected systems**
2. **Preserve logs and evidence**
3. **Notify security team immediately**
4. **Assess scope** (which tenants affected)
5. **Notify affected customers** within 72 hours (GDPR)
6. **Report to authorities** if required

---

## Risk Matrix

| Threat | Likelihood | Impact | Risk Level | Mitigation Status |
|--------|-----------|--------|------------|-------------------|
| API Key Compromise | Medium | High | **High** | ✅ Implemented |
| JWT Theft | Low | Medium | **Medium** | ⚠️ Partial |
| Brute Force | High | Medium | **Medium** | ✅ Implemented |
| Tenant Isolation Bypass | Low | Critical | **High** | ✅ Implemented |
| SQL Injection | Low | Critical | **High** | ✅ Implemented |
| DoS | Medium | Low | **Low** | ⚠️ Partial |

---

## Related Documentation

- [01-authentication-architecture.md](01-authentication-architecture.md)
- [04-authorization-model.md](04-authorization-model.md)
- [05-rate-limiting.md](05-rate-limiting.md)
- [06-audit-logging.md](06-audit-logging.md)
