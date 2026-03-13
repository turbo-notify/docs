# Alert Response Procedures

> **Relacionado:** [Database Metrics Guide](../observability/database-metrics-guide.md) | [NATS Troubleshooting](nats-troubleshooting.md) | [Database Pool Troubleshooting](database-pool-troubleshooting.md)

---

## Overview

Este runbook documenta os procedimentos de resposta a alertas no Turbo Notify. Alertas são disparados automaticamente pelo Prometheus Alertmanager e enviados para Slack (#alerts) e PagerDuty (para alertas críticos).

**Fluxo de alertas**:
```
Prometheus → Alertmanager → Slack (#alerts) / PagerDuty → On-call Engineer → Este Runbook
```

**Grafana dashboards**:
- NATS: https://grafana.turbo-notify.com/d/nats/nats
- Database: https://grafana.turbo-notify.com/d/database/database
- Sessions: https://grafana.turbo-notify.com/d/sessions/sessions
- Messages: https://grafana.turbo-notify.com/d/messages/messages

---

## Alert Severity Levels

| Severidade | Resposta Inicial | Tempo de Resposta | Escalação | Notificação |
|------------|------------------|-------------------|-----------|-------------|
| **CRITICAL (P1)** | DevOps on-call | < 5 min | Após 5min sem ação | PagerDuty + Slack + CEO/CTO |
| **HIGH (P2)** | DevOps on-call | < 15 min | Após 30min sem resolução | PagerDuty + Slack + CTO |
| **MEDIUM (P3)** | Eng on-call | < 1 hora | Após 2h sem resolução | Slack + Lead |
| **LOW (P4)** | Async review | Next business day | Após 24h | Slack apenas |

**SLA de resolução**:
- P1: < 15 minutos
- P2: < 1 hora
- P3: < 4 horas
- P4: < 1 dia útil

---

## A. Session Worker Alerts

### A.1 SessionWorkerOffline (CRITICAL - P1)

**Severidade**: CRITICAL
**Threshold**: 0 workers ativos por mais de 2 minutos

**Sintoma**:
- Nenhum session worker disponível
- Sessões não podem ser criadas ou gerenciadas
- API retorna erro ao criar sessões

**Triage**:
```bash
# Verificar status dos workers
kubectl get pods -l app=session-worker
# ou docker
docker ps | grep session-worker

# Verificar logs
kubectl logs -l app=session-worker --tail=100
# ou docker
docker logs session-worker --tail=100

# Verificar se deployment existe
kubectl get deployment session-worker
```

**Causas comuns**:
1. Deployment foi deletado acidentalmente
2. Todos os workers crasharam (crash loop)
3. Cluster sem recursos (OOM, CPU limit)
4. Image pull error

**Ação imediata**:
```bash
# Opção 1: Restart deployment (se existe)
kubectl rollout restart deployment/session-worker

# Opção 2: Scale up (se scaled down)
kubectl scale deployment session-worker --replicas=2

# Opção 3: Recrear deployment (se deletado)
kubectl apply -f k8s/session-worker-deployment.yaml

# Docker (se local/staging)
docker restart session-worker
```

**Validação**:
```bash
# Verificar workers online
kubectl get pods -l app=session-worker

# Verificar health
curl http://session-worker:8080/health

# Verificar se está consumindo NATS
nats consumer info turbo_sessions_create session-worker-sessions-consumer
```

**SLA**: < 5 minutos
**Escalação**: Se não resolver em 5min, escalar para CTO

---

### A.2 SessionWorkerCrashLoop (HIGH - P2)

**Severidade**: HIGH
**Threshold**: Worker reinicia > 3 vezes em 10 minutos

**Sintoma**:
- Worker reinicia constantemente
- Logs: `Error: Failed to initialize WhatsApp session`
- Kubernetes: Pod em `CrashLoopBackOff`
- Docker: Container restart count > 3

**Triage**:
```bash
# Verificar status do pod
kubectl describe pod <worker-pod-name>

# Verificar logs (últimos crash)
kubectl logs <worker-pod-name> --previous

# Verificar eventos
kubectl get events --sort-by='.lastTimestamp' | grep session-worker
```

**Causas comuns**:
1. **QR Code Timeout**: Nenhum usuário escaneou QR code
   - Log: `QR code not scanned within 60s`
2. **Credenciais Inválidas**: Session expirou no WhatsApp Web
   - Log: `Authentication failed`
3. **Memory Leak (Playwright)**: OOMKilled
   - `kubectl describe pod` mostra `OOMKilled`
4. **Missing env vars**: `NATS_URL`, `DATABASE_URL` não configurados
5. **Database connection error**: Pool esgotado, PgBouncer down

**Ação por causa**:

**QR Code Timeout**:
```bash
# Aumentar timeout de QR code
kubectl set env deployment/session-worker QR_CODE_TIMEOUT=120

# Ou deletar sessões pendentes antigas
psql $DATABASE_URL -c "DELETE FROM sessions WHERE status='PENDING' AND created_at < NOW() - INTERVAL '10 minutes';"
```

**Credenciais Inválidas**:
```bash
# Deletar sessão e reautenticar
psql $DATABASE_URL -c "UPDATE sessions SET status='TERMINATED' WHERE id='<session_id>';"
# Cliente precisará criar nova sessão
```

**OOMKilled**:
```bash
# Aumentar memory limit
kubectl set resources deployment session-worker --limits=memory=4Gi

# Verificar Playwright cleanup
kubectl logs <pod> | grep "browser.close\|context.close"
```

**Missing env vars**:
```bash
# Verificar ConfigMap/Secret
kubectl describe configmap session-worker-config
kubectl describe secret session-worker-secrets

# Corrigir e aplicar
kubectl apply -f k8s/session-worker-config.yaml
kubectl rollout restart deployment/session-worker
```

**Database connection error**:
```bash
# Ver Database Pool Troubleshooting Runbook
# Verificar pool saturation
curl http://public-api:8000/health | jq '.components.database'
```

**Validação**:
```bash
# Verificar restarts pararam
watch kubectl get pods -l app=session-worker

# Verificar logs sem erros
kubectl logs -f -l app=session-worker
```

**SLA**: < 30 minutos
**Escalação**: Se não resolver em 30min, escalar para CTO

---

### A.3 SessionWorkerHighMemory (MEDIUM - P3)

**Severidade**: MEDIUM
**Threshold**: Memory usage > 80% do limit por mais de 10 minutos

**Sintoma**:
- Worker usando muita memória (próximo do limit)
- Performance degradada
- Risco de OOMKill

**Triage**:
```bash
# Verificar uso de memória
kubectl top pod -l app=session-worker

# Verificar número de sessões ativas
psql $DATABASE_URL -c "SELECT count(*) FROM sessions WHERE status='ACTIVE';"

# Verificar memory leak (crescimento constante)
watch -n 5 'kubectl top pod -l app=session-worker'
```

**Ação**:
```bash
# Curto prazo: Aumentar limit
kubectl set resources deployment session-worker --limits=memory=4Gi

# Médio prazo: Escalar horizontalmente
kubectl scale deployment session-worker --replicas=3

# Longo prazo: Investigar memory leak
# Ver logs de Playwright, garantir cleanup de browsers
```

**SLA**: < 2 horas
**Escalação**: Se não resolver em 2h, escalar para Lead

---

## B. Database Alerts

### B.1 DatabasePoolSaturation (HIGH - P2)

**Severidade**: HIGH
**Threshold**: `pool_active == pool_max` por mais de 5 minutos

**Sintoma**:
- Pool de conexões saturado
- Requests demoram muito (timeouts)
- Logs: `Pool timeout ao adquirir conexão`

**Triage**:
```bash
# Verificar pool stats
curl http://public-api:8000/health | jq '.components.database'

# Verificar PgBouncer
docker exec turbo-pgbouncer sh -c \
  'psql -h localhost -p 6432 -U turbo_notify -d pgbouncer -c "SHOW POOLS;"'

# Identificar tenant saturando
docker logs turbo-public-api --since 10m 2>&1 | \
  grep "Pool timeout" | grep -o '"tenant_id":"[^"]*"' | sort | uniq -c | sort -rn
```

**Ação imediata**:
```bash
# Aumentar pool size
# Editar .env ou ConfigMap
DATABASE_POOL_SIZE=16  # era 8

# Aplicar
kubectl rollout restart deployment/turbo-public-api

# Docker
docker restart turbo-public-api
```

**Ação por tenant**:
```bash
# Se tenant específico saturando (> 60% do pool)
# Implementar rate limiting
# Ver /docs/standards/api-standards.md
```

**Ação médio prazo**:
```bash
# Escalar horizontalmente
kubectl scale deployment turbo-public-api --replicas=3

# Otimizar queries lentas
# Ver Database Pool Troubleshooting Runbook
```

**Validação**:
```bash
# Verificar pool_idle > 0
curl http://public-api:8000/health | jq '.components.database.pool_idle'

# Verificar timeouts pararam
docker logs turbo-public-api --since 5m | grep -c "Pool timeout"
```

**SLA**: < 30 minutos
**Escalação**: Se não resolver em 30min, escalar para CTO

**Runbook relacionado**: [Database Pool Troubleshooting](database-pool-troubleshooting.md#pool-exhaustion)

---

### B.2 DatabaseProtocolViolation (MEDIUM - P3)

**Severidade**: MEDIUM
**Threshold**: > 10 protocol violations em 15 minutos

**Sintoma**:
- Logs: `psycopg.errors.ProtocolViolation: client_idle_timeout`
- Connections fechadas inesperadamente

**Triage**:
```bash
# Verificar timeouts configurados
grep DATABASE_POOL_MAX_IDLE .env
docker exec turbo-pgbouncer cat /etc/pgbouncer/pgbouncer.ini | grep client_idle_timeout
```

**Ação**:
```bash
# Ajustar hierarquia: app max_idle < PgBouncer client_idle_timeout
# .env
DATABASE_POOL_MAX_IDLE=240  # 4min

# PgBouncer (se necessário)
# /etc/pgbouncer/pgbouncer.ini
client_idle_timeout = 300  # 5min

# Aplicar
kubectl rollout restart deployment/turbo-public-api
docker exec turbo-pgbouncer sh -c 'kill -HUP 1'
```

**Validação**:
```bash
# Verificar protocol violations pararam
docker logs turbo-public-api --since 10m | grep -c "ProtocolViolation"
```

**SLA**: < 2 horas
**Escalação**: Se não resolver em 2h, escalar para Lead

**Runbook relacionado**: [Database Pool Troubleshooting](database-pool-troubleshooting.md#protocol-violation)

---

### B.3 DatabaseSlowQueries (MEDIUM - P3)

**Severidade**: MEDIUM
**Threshold**: P95 query duration > 1s por mais de 10 minutos

**Sintoma**:
- Queries lentas
- Latência alta na API
- Possivelmente causando pool saturation

**Triage**:
```sql
-- Conectar ao PostgreSQL
psql $DATABASE_URL

-- Ver queries lentas
SELECT query, calls, mean_exec_time, max_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- > 1s
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Ver queries ativas por tenant
SELECT
  CASE
    WHEN query LIKE '%tenant_id%'
    THEN regexp_replace(query, '.*tenant_id[^'']*''([^'']+)''.*', '\1')
    ELSE 'unknown'
  END as tenant_id,
  count(*) as queries,
  round(avg(EXTRACT(EPOCH FROM (now() - query_start))), 2) as avg_duration
FROM pg_stat_activity
WHERE state != 'idle'
GROUP BY tenant_id
ORDER BY avg_duration DESC;
```

**Ação**:
```bash
# Identificar query problemática
# Adicionar índice (se missing)
# Reescrever query (se ineficiente)

# Exemplo: session queries lentas
CREATE INDEX CONCURRENTLY idx_sessions_tenant_id_status
  ON sessions (tenant_id, status)
  WHERE status IN ('ACTIVE', 'CONNECTING');
```

**SLA**: < 4 horas
**Escalação**: Se não resolver em 4h, escalar para Lead

---

## C. NATS Alerts

### C.1 NatsConsumerLag (HIGH - P2)

**Severidade**: HIGH
**Threshold**: `num_pending` > 100 por mais de 5 minutos

**Sintoma**:
- Mensagens acumulando no stream
- Consumer não processando
- Sessões ou mensagens não sendo processadas

**Triage**:
```bash
# Verificar pending messages
nats consumer info turbo_sessions_create session-worker-sessions-consumer
nats consumer info turbo_messages_send session-worker-messages-consumer

# Verificar workers
kubectl get pods -l app=session-worker
```

**Ação imediata**:
```bash
# Escalar workers
kubectl scale deployment session-worker --replicas=3

# Verificar se workers estão processando
kubectl logs -l app=session-worker --tail=100 | grep "Processing message"
```

**Ação por causa**:

**Workers offline**:
```bash
kubectl rollout restart deployment/session-worker
```

**Workers lentos (browser automation)**:
```bash
# Aumentar concurrency
kubectl set env deployment/session-worker WORKER_CONCURRENCY=5

# Verificar recursos
kubectl top pod -l app=session-worker
```

**Database pool saturado**:
```bash
# Ver DatabasePoolSaturation alert
```

**Validação**:
```bash
# Verificar pending decreasing
watch nats consumer info turbo_sessions_create session-worker-sessions-consumer
```

**SLA**: < 30 minutos
**Escalação**: Se não resolver em 30min, escalar para CTO

**Runbook relacionado**: [NATS Troubleshooting](nats-troubleshooting.md#21-consumer-lag-alto)

---

### C.2 NatsDLQAccumulating (CRITICAL - P1)

**Severidade**: CRITICAL
**Threshold**: DLQ > 0 mensagens

**Sintoma**:
- Mensagens na Dead Letter Queue
- Processamento falhando repetidamente
- Dados não sendo processados

**Triage**:
```bash
# Ver mensagens na DLQ
nats stream view turbo_dead_letter --last 20

# Contar por stream original
nats stream view turbo_dead_letter --last 100 | \
  grep "original_stream" | sort | uniq -c

# Identificar tenant (se aplicável)
nats stream view turbo_dead_letter --last 20 | grep "tenant_id"
```

**Ação**:
```bash
# 1. Analisar mensagem
nats stream get turbo_dead_letter 1

# 2. Identificar causa raiz
# - Erro de código? (fix + deploy)
# - Dados inválidos? (corrigir dados)
# - Configuração errada? (ajustar config)

# 3. Fix aplicado, reprocessar (se implementado)
# Ver NATS Troubleshooting Runbook para reprocessamento
```

**Validação**:
```bash
# Verificar DLQ vazia
nats stream info turbo_dead_letter | grep "Messages:"
```

**SLA**: < 15 minutos (análise), < 4 horas (resolução)
**Escalação**: Análise imediata, escalar para Lead se causa não óbvia

**Runbook relacionado**: [NATS Troubleshooting](nats-troubleshooting.md#22-mensagens-na-dead-letter-queue)

---

### C.3 NatsSlowConsumer (MEDIUM - P3)

**Severidade**: MEDIUM
**Threshold**: NATS reporta slow consumer por mais de 1 minuto

**Sintoma**:
- NATS detecta consumer lento
- Pode causar backpressure

**Triage**:
```bash
# Ver slow consumers
nats server report connections | grep -i slow

# Verificar logs NATS
kubectl logs -l app=nats --tail=200 | grep -i slow
```

**Ação**:
```bash
# Aumentar concurrency do worker
kubectl set env deployment/session-worker WORKER_CONCURRENCY=10

# Escalar workers
kubectl scale deployment session-worker --replicas=3

# Verificar recursos
kubectl top pod -l app=session-worker
```

**SLA**: < 1 hora
**Escalação**: Se não resolver em 1h, escalar para Lead

---

### C.4 NatsServerOffline (CRITICAL - P1)

**Severidade**: CRITICAL
**Threshold**: NATS server down por mais de 1 minuto

**Sintoma**:
- Nenhum componente consegue publicar/consumir eventos
- Sistema completamente degradado

**Triage**:
```bash
# Verificar NATS
kubectl get pods -l app=nats
docker ps | grep nats

# Tentar conectar
nats server ping
```

**Ação imediata**:
```bash
# Restart NATS
kubectl rollout restart statefulset/nats
# ou docker
docker restart nats

# Restart dependentes (reconectarão)
kubectl rollout restart deployment/session-worker
kubectl rollout restart deployment/turbo-public-api
kubectl rollout restart deployment/turbo-orchestrator
```

**Validação**:
```bash
# Verificar NATS online
nats server ping

# Verificar consumers reconectados
nats server report connections
```

**SLA**: < 5 minutos
**Escalação**: Se não resolver em 5min, escalar para CEO/CTO

---

## D. Webhook Alerts

### D.1 WebhookDeliveryFailures (MEDIUM - P3)

**Severidade**: MEDIUM
**Threshold**: Taxa de falha > 5% por mais de 15 minutos

**Sintoma**:
- Webhooks não sendo entregues
- Cliente não recebe eventos

**Triage**:
```bash
# Verificar logs de webhook delivery
kubectl logs -l app=webhook-worker --tail=100 | grep "delivery failed"

# Verificar por tenant
kubectl logs -l app=webhook-worker --tail=200 | \
  grep "delivery failed" | grep -o '"tenant_id":"[^"]*"' | sort | uniq -c

# Verificar URL de destino
psql $DATABASE_URL -c "SELECT webhook_url FROM tenants WHERE id='<tenant_id>';"
```

**Causas comuns**:
1. URL cliente offline/invalida (HTTP 4xx/5xx)
2. Timeout de conexão (firewall, SSL issues)
3. Rate limiting do cliente
4. Payload inválido (schema mismatch)

**Ação**:
```bash
# Testar URL manualmente
curl -X POST <webhook_url> \
  -H "Content-Type: application/json" \
  -d '{"event":"test"}'

# Se URL inválida: contatar cliente
# Se timeout: verificar DNS, SSL, firewall

# Retry manual (se implementado)
# Ver webhook delivery runbook
```

**Validação**:
```bash
# Verificar taxa de sucesso > 95%
# Ver Grafana: Webhooks > Delivery Success Rate
```

**SLA**: < 2 horas
**Escalação**: Se não resolver em 2h, escalar para Lead

---

## E. Session Alerts

### E.1 SessionStuckPending (MEDIUM - P3)

**Severidade**: MEDIUM
**Threshold**: Sessões em PENDING por mais de 5 minutos

**Sintoma**:
- Sessões não transitam para CONNECTING/ACTIVE
- Workers não processando

**Triage**:
```bash
# Ver sessões stuck
psql $DATABASE_URL -c "
  SELECT id, tenant_id, status, created_at, NOW() - created_at as age
  FROM sessions
  WHERE status='PENDING'
    AND created_at < NOW() - INTERVAL '5 minutes'
  ORDER BY created_at DESC
  LIMIT 20;"

# Verificar consumer lag
nats consumer info turbo_sessions_create session-worker-sessions-consumer
```

**Ação**:
```bash
# Ver NatsConsumerLag alert
# Escalar workers se necessário
```

**SLA**: < 1 hora
**Escalação**: Se não resolver em 1h, escalar para Lead

---

### E.2 SessionDisconnectionRate (HIGH - P2)

**Severidade**: HIGH
**Threshold**: Taxa de desconexão > 10% por mais de 15 minutos

**Sintoma**:
- Muitas sessões desconectando
- Possivelmente WhatsApp Web instabilidade

**Triage**:
```bash
# Ver rate de disconnection
psql $DATABASE_URL -c "
  SELECT
    COUNT(*) FILTER (WHERE status='DISCONNECTED') as disconnected,
    COUNT(*) FILTER (WHERE status='ACTIVE') as active,
    ROUND(100.0 * COUNT(*) FILTER (WHERE status='DISCONNECTED') / NULLIF(COUNT(*), 0), 2) as disconnect_pct
  FROM sessions
  WHERE updated_at > NOW() - INTERVAL '15 minutes';"

# Verificar logs de workers
kubectl logs -l app=session-worker --tail=200 | grep -i "disconnect\|connection lost"
```

**Causas comuns**:
1. WhatsApp Web instabilidade (global)
2. Network issues (worker side)
3. IP bloqueado por WhatsApp
4. Worker crash (todas sessões daquele worker desconectam)

**Ação**:
```bash
# Se WhatsApp global issue: aguardar (notificar clientes)
# Se worker crash: verificar SessionWorkerCrashLoop
# Se IP block: trocar IP (cloud provider)
# Se network: verificar conectividade
```

**SLA**: < 30 minutos (análise), resolução depende da causa
**Escalação**: Se não identificar causa em 30min, escalar para CTO

---

## On-Call Contacts

### Equipe

| Função | Nome/Handle | Slack | PagerDuty | Phone |
|--------|-------------|-------|-----------|-------|
| **DevOps On-call** | @oncall-devops | #alerts | ✅ | (disponível no PagerDuty) |
| **Eng On-call** | @oncall-engineering | #alerts | ✅ | (disponível no PagerDuty) |
| **Lead Engineer** | @lead-engineer | #alerts | ❌ | (escalação via Slack) |
| **CTO** | @cto | #alerts | ❌ | (escalação via Slack/Phone) |
| **CEO** | @ceo | #executive-alerts | ❌ | (somente P1 critical) |

### Canais

- **#alerts**: Todos os alertas (P1-P4)
- **#oncall-handoff**: Handoff notes entre turnos
- **#incident-response**: War room para P1/P2

### PagerDuty

- **P1 (CRITICAL)**: Liga para on-call imediatamente
- **P2 (HIGH)**: Notificação push, SMS, chamada após 5min
- **P3 (MEDIUM)**: Notificação push apenas
- **P4 (LOW)**: Email apenas

---

## Escalation Matrix

```
┌─────────────────┐
│   Alert Fires   │
└────────┬────────┘
         │
         ▼
    ┌────────────┐
    │ Severity?  │
    └──┬──┬──┬──┘
       │  │  │
   ┌───┘  │  └────┐
   │      │       │
   ▼      ▼       ▼
  P1     P2      P3/P4
   │      │       │
   ▼      ▼       ▼
DevOps  DevOps   Eng
on-call on-call  on-call
   │      │       │
   │      │       ▼
   │      │   Resolve or
   │      │   Escalate (2h)
   │      │       │
   │      ▼       ▼
   │   Resolve   Lead
   │   or Esc.    │
   │   (30min)    ▼
   │      │    Resolve
   ▼      ▼       │
CTO/CEO  CTO   (8h)
(5min)  (30min)
```

**Timeframes**:
- **P1**: Resposta < 5min, Resolução < 15min, Escalação imediata se não resolver
- **P2**: Resposta < 15min, Resolução < 1h, Escalação após 30min
- **P3**: Resposta < 1h, Resolução < 4h, Escalação após 2h
- **P4**: Resposta < 1 dia, Resolução < 3 dias, Escalação após 24h

---

## Post-Incident

### Após resolver alerta P1/P2:

1. **Atualizar ticket**: Documentar causa raiz e resolução
2. **Post-mortem**: Criar documento de troubleshooting interno (formato: YYYY-MM-DD-description.md)
3. **Notificar stakeholders**: Update em #incident-response
4. **Runbook update**: Se procedimento faltou, adicionar neste runbook
5. **Preventive measures**: Identificar e implementar melhorias

### Template de Post-Mortem

Seguir template interno de troubleshooting com as seções:
- Sintoma (logs, métricas, alertas)
- Timeline (eventos cronológicos)
- Diagnóstico (comandos executados, resultados)
- Causa Raiz (análise técnica profunda)
- Solução (passo a passo)
- Prevenção (alertas, monitoramento, melhorias)

---

## Checklist de On-Call Handoff

Ao passar o turno, documentar em #oncall-handoff:

- [ ] Alertas ativos (se houver)
- [ ] Incidentes em andamento
- [ ] Ações pendentes (follow-ups)
- [ ] Mudanças recentes (deploys, config changes)
- [ ] Tenant-specific issues
- [ ] Scheduled maintenance (se houver)

---

## Related Documentation

- [Database Metrics Guide](../observability/database-metrics-guide.md)
- [Database Pool Troubleshooting](database-pool-troubleshooting.md)
- [NATS Troubleshooting](nats-troubleshooting.md)
- [Structured Logging Guide](../observability/structured-logging-guide.md)

---

*Última atualização: 2026-03-13*
