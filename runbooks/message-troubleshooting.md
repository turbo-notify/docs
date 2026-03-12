# Message Troubleshooting

> Diagnose and resolve message send and delivery issues.

---

## Symptoms

- `POST /messages` returns `4xx` or `5xx`.
- Message remains in `pending` status for too long.
- Delivery rate drops suddenly.
- Support reports customer complaints about delayed notifications.

---

## Impact

- End users do not receive transactional or conversational messages.
- Automation workflows stall waiting for delivery confirmation.
- Support load and churn risk increase.

---

## Diagnosis

1. Validate API response codes and error payloads.
2. Check current route contract and request format:
   - `/messages`
   - `/messages/{messageID}/status`
3. Confirm sender and recipient fields are present and correctly formatted.
4. Verify if failures are concentrated by account, sender number, or region.
5. Check service logs for `message_not_found`, `invalid_json`, or `internal_error` frequency spikes.
6. Inspect queue/backlog and worker health before retrying large batches.
7. Validate `rate-sync + redis` limiter health (backend connectivity, limiter saturation, timeout spikes).
8. Check if throttling concentration is tenant-specific or tier-profile-wide.

---

## Resolution

1. For client payload issues, return actionable feedback and retry with corrected data.
2. For temporary platform instability (`5xx`), apply controlled retries with backoff.
3. For systemic failures, throttle outbound traffic and prioritize critical flows.
4. For number-specific failures, isolate the number and reroute to healthy senders when possible.
5. Communicate incident scope and ETA through support/status channels.
6. If limits are misconfigured, adjust centralized `rate-sync` limiter definitions instead of local ad-hoc throttles.
7. If entitlement mismatch is confirmed, correct tenant-to-tier mapping before widening limits.

---

## Prevention

- Keep client integrations aligned with public examples in `landing` code templates.
- Track delivery SLO and alert on sustained pending queue growth.
- Test fallback sender strategy for high-priority traffic.
- Review API contract changes using [Public API Governance](../reference/public-api-governance.md).
- Keep API and worker limiter IDs aligned in shared `rate-sync` config.
- Keep tenant-to-tier limiter mapping aligned with billing/plan definitions.

---

## Escalation

Escalate to platform owners when:

- Error rate is above baseline for more than 10 minutes.
- Pending queue keeps growing after throttling.
- Failures affect multiple tenants or regions.

---

## Related

- [Public API Governance](../reference/public-api-governance.md)
- [API Errors](../api/errors.md)
- [Rate Limits](../api/rate-limits.md)
- [Worker Recovery](worker-recovery.md)
