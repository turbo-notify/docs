# ADR: Market Test Topology - Azure VM

**Date:** 2025-08-26
**Status:** Active (current infrastructure)
**Language:** Originally in Portuguese, key sections preserved

---

## Context

Minimal viable topology for market validation phase. Single Azure VM (Standard_B2ms: 2 vCPU / 8 GB RAM).

## Current State

### DNS (Namecheap)
- `turbonotify.com` / `www.turbonotify.com` → Landing
- `wahazyc.turbonotify.com` → WAHA (discrete subdomain)

### TLS (Let's Encrypt)
- Certificates via Certbot nginx plugin
- Auto-renewal via `certbot.timer`

### Architecture

```
Internet → Nginx (host, 80/443)
   ├─ turbonotify.com / www → 127.0.0.1:3010 (Landing)
   └─ wahazyc.turbonotify.com → 127.0.0.1:3000 (WAHA, WebSocket)

VM (Azure B2ms)
   ├─ Docker: Postgres 17 (127.0.0.1:5432)
   ├─ Docker: WAHA (127.0.0.1:3000, API Key auth)
   ├─ Docker: node_exporter, cAdvisor, Prometheus, Grafana
   └─ systemd: turbonotify-stack.service (auto-start)
```

### PostgreSQL Tuning (8 GB VM)

```
shared_buffers=1GB
effective_cache_size=4GB
work_mem=32MB
maintenance_work_mem=512MB
max_connections=100
```

## What's Ready

- [x] DNS configured
- [x] TLS certificates
- [x] Nginx reverse proxy
- [x] PostgreSQL with tuning
- [x] WAHA with API Key
- [x] Observability (Prometheus/Grafana)
- [x] Auto-start on boot

## What's Pending

- [ ] Publish Landing (Next.js on port 3010)
- [ ] Publish Dashboard (Next.js on port 3020)
- [ ] API development (Python + FastAPI)
- [ ] CI/CD (GHCR + GitHub Actions)
- [ ] Backup policies

## File Locations

| Component | Path |
|-----------|------|
| Nginx configs | `/etc/nginx/sites-available/` |
| Docker Compose | `/opt/turbonotify/compose/` |
| Volumes | `/opt/turbonotify/volumes/` |
| Prometheus config | `/opt/turbonotify/configs/prometheus.yml` |
| Systemd service | `/etc/systemd/system/turbonotify-stack.service` |

---

## Notes

**2026-03-12:** This remains the active topology for market testing. API to be developed in Python/FastAPI per ADR-0001.
