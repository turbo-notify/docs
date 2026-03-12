# Webhook Troubleshooting

> Diagnose and resolve webhook delivery failures.

---

## Symptoms

- Customer endpoint misses expected `message.*` events.
- Delivery retries increase.
- Webhook latency or timeout alerts fire.
- Event consumers report duplicates or ordering issues.

---

## Impact

- Client systems lose near-real-time state updates.
- Automations dependent on inbound events fail or delay.
- Support receives reconciliation and data consistency tickets.

---

## Diagnosis

1. Confirm customer webhook URL is reachable and returning `2xx`.
2. Validate webhook secret/header checks on the receiver side.
3. Check receiver timeout behavior and response time budget.
4. Inspect retry patterns and dead-letter/backlog metrics.
5. Verify event payload compatibility with current published contract.
6. Correlate failures by tenant, endpoint host, and event type.
7. Validate `rate-sync + redis` health for webhook dispatcher (connectivity, saturation, acquire timeouts).
8. Verify whether impacted tenants share the same tier profile or tenant override.

---

## Resolution

1. Ask receiver to acknowledge quickly (`2xx`) and process asynchronously.
2. Reprocess failed events from backlog when safe and idempotent.
3. Reduce payload coupling by validating only stable fields first.
4. If a tenant endpoint is unstable, isolate retries to prevent global impact.
5. Provide customer-facing remediation steps with sample payload validation.
6. If throughput/concurrency limits are misconfigured, adjust centralized `rate-sync` limiter definitions (avoid ad-hoc local throttles).
7. If only one tier is impacted, validate tier baseline profile before changing global limits.

---

## Prevention

- Enforce idempotency using event IDs.
- Alert on retry growth and webhook timeout rate.
- Validate contract changes through governance workflow before release.
- Keep webhook consumer examples updated in docs and integration kits.
- Keep webhook limiter IDs aligned with API/worker shared `rate-sync` policies.
- Keep tenant-to-tier webhook limiter mapping aligned with contract and billing policies.

---

## Escalation

Escalate to platform owners when:

- Webhook failures affect multiple tenants.
- Retry queues continue to grow after receiver-side fix.
- There is evidence of data loss risk.

---

## Related

- [Webhooks API](../api/webhooks.md)
- [Public API Governance](../reference/public-api-governance.md)
- [Observability Overview](../observability/README.md)
- [Message Troubleshooting](message-troubleshooting.md)
