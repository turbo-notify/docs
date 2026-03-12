# Rate Limits

> Contract for usage limits and overload protection.

---

## Policy

Turbo Notify applies tier, tenant, and safety limits to preserve platform stability and reduce WhatsApp blocking risk.

Limits can vary by contracted plan tier and by sender number.

---

## Enforcement Standard

Rate-limit enforcement is standardized across platform services with `rate-sync`:

- Package: `rate-sync` (`https://pypi.org/project/rate-sync/`)
- Team source reference: `/Users/zyc/dev/rate-sync/python`
- Applies to Control Plane API, workers, and webhook delivery flows

Production environments must use `rate-sync + redis` (required). In-memory mode is only for local dev/tests.

---

## Tenant and Tiering Model

Rate limiting follows a combined model:

| Axis | Purpose | Contract behavior |
|------|---------|-------------------|
| Tenant | Fairness/isolation | One tenant cannot consume another tenant's quota budget |
| Tier (plan) | Entitlement | Contracted tier defines baseline throughput/quota profile |
| Sender/endpoint | Operational safety | Additional controls can apply per sender number and operation type |

Evaluation rule:

1. Tier entitlement is applied for the tenant.
2. Tenant isolation limits are enforced.
3. Sender/endpoint and anti-abuse controls are enforced.

If any applied limiter denies the operation, API returns `429` with the corresponding limit error.

---

## Limit Dimensions

- Messages per minute
- Messages per month
- Recipients per day
- Internal anti-abuse controls for burst behavior

For plan details, check pricing policy.

---

## Limit Exceeded Response

When limit is exceeded, API returns:

```http
429 Too Many Requests
Retry-After: 30
```

```json
{
  "error": "messages_per_minute_limit_exceeded",
  "limit": 200
}
```

---

## Common Limit Errors

| Error | Meaning |
|-------|---------|
| `messages_per_minute_limit_exceeded` | Per-minute cap exceeded |
| `messages_per_month_limit_exceeded` | Monthly cap exceeded |
| `recipients_per_day_limit_exceeded` | Daily recipients cap exceeded |

---

## Operational Notes

- Limits may apply independently per extra number inside the same tenant.
- Unlimited plans can still be throttled by anti-abuse controls.
- Client integrations should implement retry/backoff for `429`.
- API, worker, and webhook-dispatch limits must come from shared `rate-sync` limiter definitions.

---

## Related

- [Messages](messages.md)
- [Errors](errors.md)
- [Changelog](changelog.md)
- [ADR: rate-sync as Mandatory Rate-Limiting Engine](../reference/decisions/2026-03-12-rate-sync-rate-limiting.md)
