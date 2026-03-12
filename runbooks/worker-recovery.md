# Worker Recovery

> Handle worker failures and restore session service.

---

## Symptoms

- Worker heartbeat missing (>60 seconds stale)
- Sessions stuck in `connecting` or `reconnecting` state
- Alert: `WorkerDown` triggered
- Messages for affected sessions not being sent

---

## Impact

- Sessions assigned to failed worker are unavailable
- Messages queued but not delivered
- Potential tenant SLA impact

---

## Diagnosis

### Step 1: Identify Failed Worker

```promql
# Workers with stale heartbeat
turbo_worker_heartbeat_age_seconds > 60
```

Or via database:

```sql
SELECT id, hostname, status, last_heartbeat_at,
       EXTRACT(EPOCH FROM (NOW() - last_heartbeat_at)) as age_seconds
FROM workers
WHERE status != 'stopped'
ORDER BY last_heartbeat_at ASC;
```

### Step 2: Check Worker Process

```bash
# Check if container is running
docker ps | grep $WORKER_ID

# Check container logs
docker logs --tail=100 $WORKER_ID

# Check system resources
docker stats $WORKER_ID
```

### Step 3: Identify Affected Sessions

```sql
SELECT id, tenant_id, phone_number, status
FROM sessions
WHERE assigned_worker_id = '$WORKER_ID'
  AND status IN ('hot', 'connecting', 'reconnecting');
```

---

## Resolution

### Option 1: Restart Worker

If the worker process crashed but the host is healthy:

```bash
# Restart container
docker-compose restart worker-$WORKER_ID

# Or if using systemd
sudo systemctl restart turbo-worker@$WORKER_ID
```

### Option 2: Reallocate Sessions

If the worker cannot be recovered quickly:

```sql
-- Release all sessions from failed worker
UPDATE sessions
SET
  assigned_worker_id = NULL,
  lease_expires_at = NULL,
  status = 'disconnected'
WHERE assigned_worker_id = '$WORKER_ID'
  AND status IN ('hot', 'connecting', 'reconnecting');
```

The orchestrator will automatically reallocate these sessions to healthy workers.

### Option 3: Manual Session Reallocation

For critical sessions that need immediate attention:

```bash
# Force reconnect via API
curl -X POST \
  -H "Authorization: Bearer $ADMIN_TOKEN" \
  http://localhost:8000/admin/sessions/$SESSION_ID/force-reconnect
```

### Option 4: Replace Worker

If the worker host is faulty:

1. Deploy new worker on different host
2. Update worker registry
3. Reallocate sessions (Option 2)
4. Decommission failed worker

```sql
-- Mark worker as stopped
UPDATE workers
SET status = 'stopped', stopped_at = NOW()
WHERE id = '$WORKER_ID';
```

---

## Verification

After recovery, verify:

### 1. Worker Health

```bash
curl http://$WORKER_HOST:8001/health | jq
```

Expected:
```json
{
  "status": "healthy",
  "sessions_active": 15
}
```

### 2. Session Status

```sql
SELECT status, COUNT(*)
FROM sessions
WHERE assigned_worker_id = '$WORKER_ID'
GROUP BY status;
```

All sessions should be `hot`.

### 3. Message Flow

```bash
# Check pending messages
nats consumer info MESSAGES message-processor
```

### 4. Metrics

```promql
# Worker heartbeat should be fresh
turbo_worker_heartbeat_age_seconds{worker_id="$WORKER_ID"} < 30

# Sessions should be connected
turbo_sessions_total{status="hot"}
```

---

## Prevention

- Run multiple workers for redundancy
- Set up automatic worker health checks
- Configure orchestrator to detect and reallocate quickly
- Use gradual session allocation (don't overload single worker)
- Monitor memory usage and set limits

---

## Escalation

If multiple workers fail simultaneously:

1. Check infrastructure (network, host resources)
2. Check NATS connectivity
3. Check for WhatsApp-side issues
4. Escalate to P1 incident

---

## Related Runbooks

- [Session Troubleshooting](session-troubleshooting.md)
- [Message Troubleshooting](message-troubleshooting.md)

---

## Related Documentation

- [Ecosystem Architecture](../architecture/ecosystem-architecture.md)
- [Observability](../observability/README.md)
- [NATS Events](../architecture/nats-events.md)
