# Session Troubleshooting

> Diagnose and resolve session connection issues.

---

## Symptoms

- Session status showing `failed` or `disconnected`
- Tenant reporting messages not sending
- High reconnect count in metrics
- QR code needs re-scanning

---

## Impact

- Messages cannot be sent via affected session
- Tenant's downstream users don't receive notifications
- Potential SLA violation

---

## Diagnosis

### Step 1: Check Session Status

```bash
curl -H "Authorization: Bearer $TOKEN" \
  http://localhost:8000/api/v1/sessions/$SESSION_ID | jq
```

Look for:
- `status` - Current state
- `failure_count` - Number of failures
- `last_error_code` - Error type
- `last_error_message` - Error details

### Step 2: Check Session in Database

```sql
SELECT
  id,
  status,
  failure_count,
  last_error_code,
  last_error_message,
  last_connected_at,
  last_disconnected_at,
  assigned_worker_id
FROM sessions
WHERE id = '$SESSION_ID';
```

### Step 3: Check Worker Assignment

If `assigned_worker_id` is set:

```bash
# Check worker health
curl http://$WORKER_HOST:8001/health | jq

# Check worker logs
docker-compose logs --tail=100 worker | grep $SESSION_ID
```

### Step 4: Check NATS Events

```bash
# Recent session events
nats sub "turbo.session.>" --last=10
```

### Step 5: Check Metrics

```promql
# Reconnects for this session
turbo_session_reconnects_total{session_id="$SESSION_ID"}

# Session uptime
turbo_session_uptime_seconds{session_id="$SESSION_ID"}
```

---

## Common Issues & Resolutions

### Issue: AUTH_EXPIRED

**Cause:** WhatsApp session credentials have expired.

**Resolution:**
1. Request tenant re-scan QR code
2. Update session status to `cold`

```sql
UPDATE sessions
SET status = 'cold', auth_blob = NULL, failure_count = 0
WHERE id = '$SESSION_ID';
```

### Issue: CONNECTION_TIMEOUT

**Cause:** Network issues between worker and WhatsApp servers.

**Resolution:**
1. Check worker network connectivity
2. Force reconnection

```bash
# Via API
curl -X POST http://localhost:8000/api/v1/sessions/$SESSION_ID/reconnect
```

### Issue: ACCOUNT_BANNED

**Cause:** WhatsApp has banned the phone number.

**Resolution:**
1. Cannot recover this session
2. Notify tenant to use a different number
3. Mark session as permanently failed

```sql
UPDATE sessions
SET status = 'failed',
    last_error_code = 'ACCOUNT_BANNED',
    last_error_message = 'Phone number has been banned by WhatsApp'
WHERE id = '$SESSION_ID';
```

### Issue: WORKER_UNAVAILABLE

**Cause:** Assigned worker is down or unreachable.

**Resolution:**
1. Clear worker assignment
2. Let orchestrator reallocate

```sql
UPDATE sessions
SET assigned_worker_id = NULL, lease_expires_at = NULL
WHERE id = '$SESSION_ID';
```

### Issue: HIGH_RECONNECT_RATE

**Cause:** Session is unstable, repeatedly disconnecting.

**Resolution:**
1. Check for rate limiting
2. Reduce message frequency
3. Consider session warm-up strategy

```sql
-- Check reconnect history
SELECT * FROM session_events
WHERE session_id = '$SESSION_ID'
  AND event_type = 'reconnect'
  AND created_at > NOW() - INTERVAL '1 hour'
ORDER BY created_at DESC;
```

---

## Prevention

- Monitor reconnect rate and alert when >5/hour
- Implement proper backoff on reconnection
- Use session warm-up before high-volume periods
- Monitor auth expiration and proactively notify tenants

---

## Related Runbooks

- [Worker Recovery](worker-recovery.md)
- [QR Code Authentication](qr-authentication.md)

---

## Related Documentation

- [Session Lifecycle](../architecture/session-lifecycle.md)
- [Metrics Guide](../observability/metrics-guide.md)
