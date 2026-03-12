# Runbooks Index

> Operational procedures for common issues.

---

## Overview

Runbooks provide step-by-step procedures for diagnosing and resolving operational issues.

### Format

Each runbook follows this structure:

1. **Symptoms** - How to identify the issue
2. **Impact** - What's affected
3. **Diagnosis** - Steps to understand the problem
4. **Resolution** - Steps to fix the issue
5. **Prevention** - How to avoid recurrence

---

## Available Runbooks

### Session Management

| Runbook | Description |
|---------|-------------|
| [Session Troubleshooting](session-troubleshooting.md) | Session connection issues |
| [QR Code Authentication](qr-authentication.md) | QR scan failures |

### Worker Operations

| Runbook | Description |
|---------|-------------|
| [Worker Recovery](worker-recovery.md) | Worker crash/failure handling |
| [Worker Scaling](worker-scaling.md) | Adding/removing workers |

### Message Delivery

| Runbook | Description |
|---------|-------------|
| [Message Troubleshooting](message-troubleshooting.md) | Message send failures |
| [Media Delivery](media-delivery.md) | Media upload/send issues |

### Webhook Delivery

| Runbook | Description |
|---------|-------------|
| [Webhook Troubleshooting](webhook-troubleshooting.md) | Webhook delivery failures |
| [Webhook Backlog](webhook-backlog.md) | Large webhook queue |

### Infrastructure

| Runbook | Description |
|---------|-------------|
| [Database Issues](database-issues.md) | PostgreSQL problems |
| [NATS Issues](nats-issues.md) | Message bus problems |

---

## Quick Reference

### Check Session Status

```bash
# Via API
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/v1/sessions/$SESSION_ID

# Via database
psql -c "SELECT id, status, failure_count, last_error_code FROM sessions WHERE id = '$SESSION_ID'"
```

### Check Worker Health

```bash
# Worker health endpoint
curl http://worker-01:8001/health

# Via NATS
nats sub "turbo.worker.heartbeat" --count=10
```

### View Recent Logs

```bash
# API logs
docker-compose logs -f --tail=100 api

# Worker logs
docker-compose logs -f --tail=100 worker
```

### Common NATS Commands

```bash
# List streams
nats stream ls

# View stream info
nats stream info SESSIONS

# View consumers
nats consumer ls SESSIONS

# Purge dead-letter
nats stream purge WEBHOOKS_DEADLETTER
```

---

## Escalation

### Severity Levels

| Level | Response Time | Examples |
|-------|---------------|----------|
| **P1 - Critical** | Immediate | All sessions down, data loss |
| **P2 - High** | 1 hour | High failure rate, major tenant affected |
| **P3 - Medium** | 4 hours | Single session issues, degraded performance |
| **P4 - Low** | Next business day | Minor issues, cosmetic bugs |

### Escalation Path

1. **L1** - On-call engineer (runbook execution)
2. **L2** - Senior engineer (complex diagnosis)
3. **L3** - Platform team (architectural issues)

---

## Related Documentation

- [Observability](../observability/README.md) - Monitoring and alerts
- [Architecture](../architecture/ecosystem-architecture.md) - System design
