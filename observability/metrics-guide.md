# Metrics Guide

> Required metrics, instrumentation, and PromQL queries.

---

## Overview

All services expose metrics via `/metrics` endpoint in Prometheus format.

### Naming Convention

```
turbo_<domain>_<metric>_<unit>
```

Examples:
- `turbo_session_uptime_seconds`
- `turbo_message_latency_seconds`
- `turbo_webhook_deliveries_total`

---

## Session Metrics

### turbo_sessions_total

Total number of sessions by status.

**Type:** Gauge

**Labels:**
- `tenant_id` - Tenant identifier
- `status` - Session status (cold, warm, hot, failed)

**Example Queries:**

```promql
# Total hot sessions
sum(turbo_sessions_total{status="hot"})

# Sessions by tenant
sum by (tenant_id) (turbo_sessions_total{status="hot"})

# Failed session ratio
sum(turbo_sessions_total{status="failed"}) / sum(turbo_sessions_total)
```

---

### turbo_session_reconnects_total

Counter of session reconnection events.

**Type:** Counter

**Labels:**
- `session_id` - Session identifier
- `tenant_id` - Tenant identifier
- `reason` - Reconnect reason (network, server, timeout)

**Example Queries:**

```promql
# Reconnects in last hour
increase(turbo_session_reconnects_total[1h])

# Reconnects per session per hour
rate(turbo_session_reconnects_total[1h])

# Sessions with high reconnect rate
sum by (session_id) (rate(turbo_session_reconnects_total[1h])) > 5
```

---

### turbo_session_uptime_seconds

Duration of session connectivity.

**Type:** Histogram

**Buckets:** 60, 300, 900, 1800, 3600, 7200, 14400, 28800, 86400

**Labels:**
- `tenant_id` - Tenant identifier

**Example Queries:**

```promql
# Average session uptime
avg(turbo_session_uptime_seconds)

# P95 session uptime
histogram_quantile(0.95, rate(turbo_session_uptime_seconds_bucket[24h]))

# Sessions with <1h uptime
sum(turbo_session_uptime_seconds_bucket{le="3600"})
```

---

### turbo_session_connection_latency_seconds

Time to establish session connection.

**Type:** Histogram

**Buckets:** 0.5, 1, 2, 5, 10, 30, 60

**Labels:**
- `tenant_id` - Tenant identifier

**Example Queries:**

```promql
# Average connection time
rate(turbo_session_connection_latency_seconds_sum[5m]) /
rate(turbo_session_connection_latency_seconds_count[5m])

# P99 connection latency
histogram_quantile(0.99, rate(turbo_session_connection_latency_seconds_bucket[5m]))
```

---

## Message Metrics

### turbo_messages_sent_total

Counter of sent messages.

**Type:** Counter

**Labels:**
- `tenant_id` - Tenant identifier
- `type` - Message type (text, image, document)
- `status` - Final status (sent, failed)

**Example Queries:**

```promql
# Messages sent in last hour
increase(turbo_messages_sent_total[1h])

# Failed message ratio
sum(turbo_messages_sent_total{status="failed"}) /
sum(turbo_messages_sent_total) * 100

# Messages by type
sum by (type) (increase(turbo_messages_sent_total[24h]))
```

---

### turbo_message_latency_seconds

Time from API request to WhatsApp delivery.

**Type:** Histogram

**Buckets:** 0.1, 0.5, 1, 2, 5, 10, 30

**Labels:**
- `tenant_id` - Tenant identifier
- `type` - Message type

**Example Queries:**

```promql
# Average message latency
rate(turbo_message_latency_seconds_sum[5m]) /
rate(turbo_message_latency_seconds_count[5m])

# P95 latency
histogram_quantile(0.95, rate(turbo_message_latency_seconds_bucket[5m]))

# Slow messages (>5s)
sum(rate(turbo_message_latency_seconds_bucket{le="5"}[5m]))
```

---

### turbo_messages_pending

Messages waiting to be sent.

**Type:** Gauge

**Labels:**
- `worker_id` - Worker identifier

**Example Queries:**

```promql
# Total pending messages
sum(turbo_messages_pending)

# Pending by worker
sum by (worker_id) (turbo_messages_pending)

# Workers with high backlog
turbo_messages_pending > 100
```

---

## Webhook Metrics

### turbo_webhook_deliveries_total

Counter of webhook delivery attempts.

**Type:** Counter

**Labels:**
- `tenant_id` - Tenant identifier
- `event_type` - Event type (message.received, message.status)
- `status` - Delivery status (success, failed, timeout)

**Example Queries:**

```promql
# Successful deliveries
sum(increase(turbo_webhook_deliveries_total{status="success"}[1h]))

# Failure rate
sum(turbo_webhook_deliveries_total{status!="success"}) /
sum(turbo_webhook_deliveries_total) * 100

# Failures by tenant
sum by (tenant_id) (increase(turbo_webhook_deliveries_total{status="failed"}[1h]))
```

---

### turbo_webhook_latency_seconds

Time to deliver webhook (including retries).

**Type:** Histogram

**Buckets:** 0.1, 0.5, 1, 2, 5, 10, 30

**Labels:**
- `tenant_id` - Tenant identifier

**Example Queries:**

```promql
# Average webhook latency
rate(turbo_webhook_latency_seconds_sum[5m]) /
rate(turbo_webhook_latency_seconds_count[5m])

# Slow tenant webhooks
topk(10,
  rate(turbo_webhook_latency_seconds_sum[1h]) /
  rate(turbo_webhook_latency_seconds_count[1h])
)
```

---

### turbo_webhook_pending

Webhooks waiting for delivery.

**Type:** Gauge

**Labels:** None (global)

**Example Queries:**

```promql
# Pending webhooks
turbo_webhook_pending

# Trend
deriv(turbo_webhook_pending[1h])
```

---

## Worker Metrics

### turbo_worker_sessions

Number of sessions per worker.

**Type:** Gauge

**Labels:**
- `worker_id` - Worker identifier

**Example Queries:**

```promql
# Sessions per worker
turbo_worker_sessions

# Total sessions across workers
sum(turbo_worker_sessions)

# Unbalanced workers
stddev(turbo_worker_sessions) > 5
```

---

### turbo_worker_memory_bytes

Memory usage per worker.

**Type:** Gauge

**Labels:**
- `worker_id` - Worker identifier

**Example Queries:**

```promql
# Memory by worker
turbo_worker_memory_bytes / 1024 / 1024  # MB

# Memory pressure
turbo_worker_memory_bytes > 1073741824  # >1GB
```

---

### turbo_worker_heartbeat_age_seconds

Seconds since last heartbeat.

**Type:** Gauge

**Labels:**
- `worker_id` - Worker identifier

**Example Queries:**

```promql
# Stale heartbeats (>60s)
turbo_worker_heartbeat_age_seconds > 60

# Workers potentially down
count(turbo_worker_heartbeat_age_seconds > 120)
```

---

## API Metrics

### turbo_api_requests_total

Total API requests.

**Type:** Counter

**Labels:**
- `method` - HTTP method
- `endpoint` - API endpoint
- `status` - HTTP status code

**Example Queries:**

```promql
# Requests per second
rate(turbo_api_requests_total[5m])

# Error rate
sum(rate(turbo_api_requests_total{status=~"5.."}[5m])) /
sum(rate(turbo_api_requests_total[5m])) * 100

# Requests by endpoint
sum by (endpoint) (rate(turbo_api_requests_total[5m]))
```

---

### turbo_api_latency_seconds

API request latency.

**Type:** Histogram

**Buckets:** 0.01, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5

**Labels:**
- `method` - HTTP method
- `endpoint` - API endpoint

**Example Queries:**

```promql
# P95 latency
histogram_quantile(0.95, rate(turbo_api_latency_seconds_bucket[5m]))

# Slow endpoints
topk(5,
  histogram_quantile(0.99, rate(turbo_api_latency_seconds_bucket[5m]))
)
```

---

## Alerting Rules

### Critical Alerts

```yaml
groups:
  - name: turbo-critical
    rules:
      - alert: SessionFailureHigh
        expr: |
          sum(turbo_sessions_total{status="failed"}) /
          sum(turbo_sessions_total) > 0.1
        for: 5m
        labels:
          severity: critical

      - alert: WorkerDown
        expr: turbo_worker_heartbeat_age_seconds > 120
        for: 2m
        labels:
          severity: critical

      - alert: MessageDeliveryDegraded
        expr: |
          histogram_quantile(0.95, rate(turbo_message_latency_seconds_bucket[5m])) > 5
        for: 5m
        labels:
          severity: critical
```

### Warning Alerts

```yaml
groups:
  - name: turbo-warning
    rules:
      - alert: SessionReconnectHigh
        expr: rate(turbo_session_reconnects_total[1h]) > 5
        for: 15m
        labels:
          severity: warning

      - alert: WebhookBacklog
        expr: turbo_webhook_pending > 1000
        for: 5m
        labels:
          severity: warning

      - alert: MemoryPressure
        expr: |
          turbo_worker_memory_bytes / (1024*1024*1024) > 0.8
        for: 10m
        labels:
          severity: warning
```

---

## Dashboard Panels

### Overview Dashboard

- Sessions by status (pie chart)
- Messages per hour (time series)
- Error rate (stat)
- P95 latency (stat)

### Sessions Dashboard

- Hot sessions over time
- Reconnect rate
- Uptime distribution
- Failed sessions table

### Workers Dashboard

- Sessions per worker (bar chart)
- Memory usage (time series)
- Heartbeat status (status map)

---

## Related Documentation

- [Observability Overview](README.md) - Stack overview
- [Runbooks](../runbooks/README.md) - Troubleshooting
