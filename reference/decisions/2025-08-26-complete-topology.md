# ADR: Complete Topology - VPS Hostinger (32 GB RAM)

**Date:** 2025-08-26
**Status:** Reference (future production)
**Language:** Originally in Portuguese, key sections preserved

---

## Context

Production topology for a 32 GB RAM VPS with high reliability requirements.

## Architecture Overview

```
Internet → Nginx (host, 80/443, TLS Let's Encrypt)
  ├─ turbonotify.com / www → landing (Next.js, 3010)
  ├─ dashboard.turbonotify.com → dashboard (Next.js, 3020)
  ├─ api.turbonotify.com → API (multiple instances)
  └─ wahazyc.turbonotify.com → WAHA (WebSocket, API Key)

VPS (32 GB RAM)
  ├─ Docker network: front (reverse proxy) / back (services)
  ├─ Postgres 17 + PgBouncer
  ├─ Redis (rate-limit / cache)
  ├─ NATS (queues/events)
  ├─ API — 2+ instances behind Nginx
  ├─ Landing & Dashboard (Next.js)
  ├─ WAHA (NOWEB, sessions in volume)
  └─ Observability: node_exporter, cadvisor, postgres-exporter,
                    nats-exporter, redis-exporter, prometheus, grafana
```

## Domain Routing

| Domain | Service |
|--------|---------|
| `turbonotify.com`, `www` | Landing |
| `dashboard.turbonotify.com` | Dashboard |
| `api.turbonotify.com` | API |
| `wahazyc.turbonotify.com` | WAHA (discrete) |

## PostgreSQL Tuning (32 GB)

```
shared_buffers=8GB
effective_cache_size=24GB
work_mem=64MB
maintenance_work_mem=2GB
wal_compression=on
max_wal_size=8GB
checkpoint_timeout=15min
max_connections=400 (via PgBouncer)
```

## Resource Allocation

| Component | Memory |
|-----------|--------|
| PostgreSQL | 12-16 GB |
| API (2-4 instances) | 200-400 MB each |
| Redis | 512 MB - 2 GB |
| NATS | 512 MB |
| Next.js (2 apps) | 0.5-1.5 GB total |
| Prometheus + Grafana | 1-2 GB |

## Security

- SSH with key only, `PasswordAuthentication no`
- UFW: 22, 80, 443 only
- WAHA: require `X-Api-Key` with sha512 hash
- Secrets: `.env` with 600 permissions

---

## Notes

**2026-03-12:** This topology remains valid. Update API technology references from Go to Python per ADR-0001.
