# Database Metrics Guide

> **Relacionado:** [Structured Logging Guide](./structured-logging-guide.md) | [API Standards](../standards/api-standards.md)

---

Guia completo sobre métricas de pool de conexões de banco de dados no Turbo Notify.

## 📊 Índice

- [Visão Geral](#visão-geral)
- [Métricas Disponíveis](#métricas-disponíveis)
- [Métricas Tenant-Aware](#métricas-tenant-aware)
- [Interpretação das Métricas](#interpretação-das-métricas)
- [Alertas Recomendados](#alertas-recomendados)
- [Queries Prometheus](#queries-prometheus)
- [Dashboards](#dashboards)

---

## Visão Geral

O sistema expõe métricas de pool de conexões em **dois formatos**:

1. **Prometheus** (via `/metrics`): Para monitoramento time-series
2. **Health Endpoint** (via `/health`): Para health checks de Kubernetes/Docker

### Arquitetura de Métricas

```
┌──────────────┐
│ Application  │
│   (FastAPI)  │
└───────┬──────┘
        │
        ├─── Prometheus Metrics (/metrics)
        │    • db_pool_connections{state="active|idle", tenant_id="..."}
        │    • db_connection_acquisition_seconds
        │    • db_query_duration_seconds{operation="..."}
        │    • db_errors_total{error_type="...", tenant_id="..."}
        │
        └─── Health Endpoint (/health)
             • pool_active
             • pool_idle
             • pool_max
             • status
```

---

## Métricas Disponíveis

### 1. `db_pool_connections`

**Tipo:** Gauge
**Labels:** `state` (active, idle, max, min), `service`, `tenant_id` (opcional)
**Descrição:** Número de conexões no pool por estado

**Exemplo de uso:**
```promql
# Conexões ativas (public-api)
db_pool_connections{service="public-api", state="active"}

# Conexões idle disponíveis
db_pool_connections{state="idle"}

# Tamanho máximo do pool
db_pool_connections{state="max"}
```

**Interpretação:**
- `active`: Conexões em uso no momento
- `idle`: Conexões disponíveis para uso
- `max`: Limite superior configurado (pool_size)
- `min`: Limite inferior mantido (pool_min_size)

**Valores esperados (public-api):**
```
active: 0-12 (max=12)
idle: 0-12
max: 12
min: 2
```

---

### 2. `db_connection_acquisition_seconds`

**Tipo:** Histogram
**Buckets:** 0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5, 10, 30
**Descrição:** Latência de aquisição de conexão do pool

**Métricas derivadas:**
```promql
# P50 - Mediana (50% das aquisições abaixo deste valor)
histogram_quantile(0.50, rate(db_connection_acquisition_seconds_bucket[5m]))

# P95 - 95% das aquisições abaixo deste valor
histogram_quantile(0.95, rate(db_connection_acquisition_seconds_bucket[5m]))

# P99 - 99% das aquisições abaixo deste valor
histogram_quantile(0.99, rate(db_connection_acquisition_seconds_bucket[5m]))
```

**Interpretação:**
- **P50 < 10ms**: Excelente (pool saudável, conexões imediatas)
- **P95 < 100ms**: Bom (pool com carga normal)
- **P99 < 1s**: Aceitável (pool ocasionalmente saturado)
- **P99 > 5s**: PROBLEMA (pool saturado, investigar)

**SLO Recomendado:**
```
P95 < 100ms (95% das aquisições em < 100ms)
P99 < 1s    (99% das aquisições em < 1s)
```

---

### 3. `db_query_duration_seconds`

**Tipo:** Histogram
**Labels:** `operation` (session_create, session_query, message_insert, message_query, webhook_query)
**Descrição:** Latência de execução de queries específicas

**Operações monitoradas:**
- `session_create`: INSERT INTO sessions
- `session_query`: SELECT FROM sessions WHERE tenant_id = ?
- `message_insert`: INSERT INTO messages
- `message_query`: SELECT FROM messages WHERE tenant_id = ?
- `webhook_query`: SELECT FROM webhooks WHERE tenant_id = ?

**Query P95 por operação:**
```promql
histogram_quantile(
  0.95,
  rate(db_query_duration_seconds_bucket{operation="session_query"}[5m])
)
```

**Thresholds recomendados:**
- Session queries: P95 < 50ms
- Message queries: P95 < 100ms
- Webhook queries: P95 < 50ms

---

### 4. `db_errors_total`

**Tipo:** Counter
**Labels:** `error_type` (pool_timeout, protocol_violation, connection_error, unknown), `tenant_id` (opcional)
**Descrição:** Total de erros de database por tipo

**Tipos de erro:**

#### `pool_timeout`
- **Causa:** Pool saturado, timeout ao aguardar conexão disponível
- **Ação:** Aumentar pool_size ou escalar horizontalmente
- **Turbo-specific:** Verificar qual tenant está saturando o pool

#### `protocol_violation`
- **Causa:** Mismatch de timeouts (app max_idle > PgBouncer client_idle_timeout)
- **Ação:** Ajustar hierarquia de timeouts

#### `connection_error`
- **Causa:** Rede, banco indisponível, DNS failure
- **Ação:** Verificar conectividade com PostgreSQL/PgBouncer

#### `unknown`
- **Causa:** Erro inesperado não mapeado
- **Ação:** Investigar logs para root cause

**Query de taxa de erro:**
```promql
# Taxa de erros nos últimos 5 minutos
rate(db_errors_total[5m])

# Taxa de protocol violations por tenant
rate(db_errors_total{error_type="protocol_violation", tenant_id="tenant_123"}[5m])
```

---

## Métricas Tenant-Aware

### 1. Pool Utilization by Tenant

**Query:**
```promql
# Top 5 tenants utilizando pool
topk(5,
  sum by (tenant_id) (
    rate(db_connection_acquisition_seconds_count{tenant_id!=""}[5m])
  )
)
```

**Interpretação:**
- Identifica tenants "ruidosos" (heavy users)
- Detecta tenant saturando pool compartilhado
- Auxilia decisão de tenant-specific pool

---

### 2. Session Query Latency by Tenant

**Query:**
```promql
# P99 de session queries por tenant
histogram_quantile(
  0.99,
  sum by (tenant_id, le) (
    rate(db_query_duration_seconds_bucket{operation="session_query"}[5m])
  )
)
```

**Threshold:** P99 < 100ms por tenant

**Ação se P99 > 500ms:**
- Verificar índices em sessions table
- Analisar queries do tenant (N+1, missing WHERE tenant_id)
- Considerar particionamento por tenant

---

### 3. Message Insertion Rate by Tenant

**Query:**
```promql
# Taxa de inserção de mensagens por tenant
sum by (tenant_id) (
  rate(db_query_duration_seconds_count{operation="message_insert"}[5m])
)
```

**Threshold:** < 100 msg/s por tenant (free tier)

**Ação se > threshold:**
- Rate limiting via rate-sync
- Upgrade tenant para tier superior

---

### 4. Error Rate by Tenant

**Query:**
```promql
# Taxa de erros de database por tenant
sum by (tenant_id, error_type) (
  rate(db_errors_total{tenant_id!=""}[5m])
)
```

**Threshold:** < 1% error rate por tenant

**Ação se > 5% error rate:**
- Investigar logs do tenant
- Verificar quota exceeded
- Contact tenant (possível ataque)

---

## Interpretação das Métricas

### Cenário 1: Pool Saudável

```promql
db_pool_connections{service="public-api", state="active"} = 3
db_pool_connections{service="public-api", state="idle"} = 9
db_pool_connections{service="public-api", state="max"} = 12

histogram_quantile(0.95, db_connection_acquisition_seconds_bucket) = 0.008  # 8ms

rate(db_errors_total[5m]) = 0
```

**Diagnóstico:** ✅ Pool operando normalmente
- Utilização: 25% (3/12)
- P95 < 10ms (conexões instantâneas)
- Zero erros

---

### Cenário 2: Pool Saturando (Tenant-Specific)

```promql
db_pool_connections{service="public-api", state="active"} = 10
db_pool_connections{service="public-api", state="idle"} = 2
db_pool_connections{service="public-api", state="max"} = 12

# Identificar tenant saturando
topk(1, sum by (tenant_id) (rate(db_connection_acquisition_seconds_count[5m])))
# tenant_123: 80% das aquisições

histogram_quantile(0.95, db_connection_acquisition_seconds_bucket) = 0.250  # 250ms
histogram_quantile(0.99, db_connection_acquisition_seconds_bucket) = 2.500  # 2.5s

rate(db_errors_total{error_type="pool_timeout"}[5m]) = 0.02  # 1.2/min
```

**Diagnóstico:** ⚠️ Pool próximo da saturação, tenant_123 saturando
- Utilização: 83% (10/12)
- P95 = 250ms (aguardando conexão disponível)
- P99 = 2.5s (requests lentos)
- 1.2 pool timeouts/min
- **tenant_123 responsável por 80% das aquisições**

**Ações:**
1. **Immediate:** Rate limit tenant_123 (via rate-sync)
2. **Short-term:** Aumentar `pool_size` de 12 → 18
3. **Medium-term:** Considerar tenant-specific pool para tenant_123
4. **Long-term:** Migrar tenant_123 para dedicated infrastructure

---

### Cenário 3: Protocol Violation

```promql
db_pool_connections{service="public-api", state="active"} = 3
db_pool_connections{service="public-api", state="idle"} = 9

rate(db_errors_total{error_type="protocol_violation"}[5m]) = 0.1  # 6/min
```

**Diagnóstico:** ⚠️ Mismatch de timeouts
- Pool com capacidade sobrando
- Protocol violations frequentes (6/min)

**Ações:**
1. Verificar configuração:
   - App `max_idle`: deve ser < PgBouncer `client_idle_timeout`
2. Ajustar timeouts conforme [Runbook](../runbooks/database-pool-troubleshooting.md#protocol-violation)

---

### Cenário 4: Pool Esgotado (Multi-Tenant)

```promql
db_pool_connections{service="public-api", state="active"} = 12
db_pool_connections{service="public-api", state="idle"} = 0

# Top tenants causando problema
topk(3, sum by (tenant_id) (rate(db_connection_acquisition_seconds_count[5m])))
# tenant_123: 40%
# tenant_456: 30%
# tenant_789: 20%

histogram_quantile(0.50, db_connection_acquisition_seconds_bucket) = 10.0  # 10s
histogram_quantile(0.99, db_connection_acquisition_seconds_bucket) = 25.0  # 25s (timeout)

rate(db_errors_total{error_type="pool_timeout"}[5m]) = 1.0  # 60/min
```

**Diagnóstico:** 🚨 CRÍTICO - Pool completamente esgotado
- Utilização: 100% (12/12)
- P50 = 10s (metade dos requests aguardam 10s)
- P99 = 25s (hitting timeout)
- 60 pool timeouts/min (API inacessível)
- **3 tenants consumindo 90% do pool**

**Ações IMEDIATAS:**
1. Rate limit tenant_123, tenant_456, tenant_789
2. Escalar horizontalmente (dobrar réplicas) + aumentar pool_size
3. Investigar se há incident (DDoS, query travada, tenant abuse)

---

## Alertas Recomendados

### Alerta 1: Pool Saturation (Warning)

```yaml
alert: DatabasePoolSaturation
expr: |
  (
    db_pool_connections{state="active"}
    / db_pool_connections{state="max"}
  ) > 0.8
for: 5m
labels:
  severity: warning
annotations:
  summary: "Pool {{ $labels.service }} saturando ({{ $value | humanizePercentage }})"
  description: |
    Pool de {{ $labels.service }} está utilizando mais de 80% da capacidade há 5 minutos.

    Ação: Considerar aumentar pool_size ou escalar horizontalmente.
```

---

### Alerta 2: Pool Exhaustion (Critical)

```yaml
alert: DatabasePoolExhaustion
expr: |
  db_pool_connections{state="active"} == db_pool_connections{state="max"}
for: 1m
labels:
  severity: critical
annotations:
  summary: "Pool {{ $labels.service }} ESGOTADO (100% ativo)"
  description: |
    Pool de {{ $labels.service }} está 100% utilizado há 1 minuto.
    Requests provavelmente estão falhando com timeout.

    Ação IMEDIATA: Escalar horizontalmente ou aumentar pool_size.
```

---

### Alerta 3: Tenant Saturating Pool (Warning)

```yaml
alert: TenantSaturatingPool
expr: |
  (
    sum by (tenant_id) (
      rate(db_connection_acquisition_seconds_count{tenant_id!=""}[5m])
    )
    / ignoring(tenant_id) group_left
    sum(rate(db_connection_acquisition_seconds_count[5m]))
  ) > 0.6
for: 5m
labels:
  severity: warning
annotations:
  summary: "Tenant {{ $labels.tenant_id }} saturando pool ({{ $value | humanizePercentage }})"
  description: |
    Tenant {{ $labels.tenant_id }} está responsável por mais de 60% das aquisições de conexão.

    Ação: Considerar rate limiting ou tenant-specific pool.
```

---

### Alerta 4: High Protocol Violations (Warning)

```yaml
alert: DatabaseProtocolViolationHigh
expr: |
  rate(db_errors_total{error_type="protocol_violation"}[5m]) > 0.1
for: 5m
labels:
  severity: warning
annotations:
  summary: "Alta taxa de protocol violations ({{ $value }}/s)"
  description: |
    {{ $labels.service }} está gerando mais de 0.1 protocol violations/s (6/min).

    Causa provável: Mismatch de timeouts (app max_idle > PgBouncer client_idle_timeout)

    Ação: Verificar e ajustar hierarquia de timeouts.
```

---

### Alerta 5: Slow Session Queries (Warning)

```yaml
alert: SlowSessionQueries
expr: |
  histogram_quantile(
    0.95,
    rate(db_query_duration_seconds_bucket{operation="session_query"}[5m])
  ) > 0.1
for: 5m
labels:
  severity: warning
annotations:
  summary: "Session queries lentas (P95: {{ $value | humanizeDuration }})"
  description: |
    Queries de sessão estão demorando mais de 100ms (P95).

    Causa provável: Missing índices, N+1 queries, ou tenant-specific issue.

    Ação: Verificar índices em sessions table, analisar slow query log.
```

---

### Alerta 6: Message Insertion Backlog (Warning)

```yaml
alert: MessageInsertionBacklog
expr: |
  histogram_quantile(
    0.95,
    rate(db_query_duration_seconds_bucket{operation="message_insert"}[5m])
  ) > 0.5
for: 5m
labels:
  severity: warning
annotations:
  summary: "Latência alta em inserção de mensagens (P95: {{ $value | humanizeDuration }})"
  description: |
    Inserção de mensagens está demorando mais de 500ms (P95).

    Causa provável: Lock contention, índices em messages table, ou pool saturation.

    Ação: Investigar pg_stat_activity para locks, verificar pool utilization.
```

---

## Queries Prometheus

### Monitoramento de Pool

```promql
# Utilização do pool (%)
(
  db_pool_connections{state="active"}
  / db_pool_connections{state="max"}
) * 100

# Taxa de conexões adquiridas por segundo
rate(db_connection_acquisition_seconds_count[5m])

# Latência média de aquisição (últimos 5min)
rate(db_connection_acquisition_seconds_sum[5m])
/ rate(db_connection_acquisition_seconds_count[5m])
```

---

### Taxa de Erros

```promql
# Taxa total de erros (todos os tipos)
sum(rate(db_errors_total[5m]))

# Taxa de erros por tipo
sum by (error_type) (rate(db_errors_total[5m]))

# Porcentagem de requests com erro
(
  rate(db_errors_total[5m])
  / rate(db_connection_acquisition_seconds_count[5m])
) * 100
```

---

### Tenant-Aware Queries

```promql
# Top 5 tenants por pool utilization
topk(5,
  sum by (tenant_id) (
    rate(db_connection_acquisition_seconds_count{tenant_id!=""}[5m])
  )
)

# Taxa de erros por tenant (top 10)
topk(10,
  sum by (tenant_id) (
    rate(db_errors_total{tenant_id!=""}[5m])
  )
)

# P99 de aquisição de conexão por tenant
histogram_quantile(
  0.99,
  sum by (tenant_id, le) (
    rate(db_connection_acquisition_seconds_bucket{tenant_id!=""}[5m])
  )
)
```

---

### Comparação Multi-Service

```promql
# Utilização de pool por serviço
sum by (service) (db_pool_connections{state="active"})

# P95 de latência por serviço
histogram_quantile(
  0.95,
  sum by (service, le) (rate(db_connection_acquisition_seconds_bucket[5m]))
)

# Taxa de erro por serviço
sum by (service) (rate(db_errors_total[5m]))
```

---

## Dashboards

### Dashboard Recomendado: Database Pool Overview

**Painéis essenciais:**

1. **Pool Connections (Time Series)**
   - Ativo vs Idle vs Max (por serviço)
   - Ajuda visualizar saturação ao longo do tempo

2. **Connection Acquisition Latency (P50, P95, P99)**
   - Histogram com percentis
   - Detecta degradação de performance

3. **Error Rate by Type (Stacked Area)**
   - pool_timeout, protocol_violation, connection_error
   - Visualiza padrões de erro

4. **Pool Utilization Gauge (%)**
   - Gauge de 0-100% por serviço
   - Verde (< 50%), Amarelo (50-80%), Vermelho (> 80%)

5. **Top Tenants by Pool Usage (Bar Chart)**
   - Top 10 tenants por aquisição de conexão
   - Identifica tenant causando contenção

6. **Query Latency by Operation (Heatmap)**
   - session_query, message_insert, webhook_query
   - P50, P95, P99 por operação

7. **Active Connections by Service (Stacked Area)**
   - Comparação entre public-api, orchestrator, session-worker
   - Identifica serviço causando contenção

---

### Dashboard Tenant-Specific

**Variável:** `tenant_id` (multi-select)

**Painéis:**
1. Pool utilization by selected tenants
2. Query latency (P95) by selected tenants
3. Error rate by selected tenants
4. Message insertion rate by selected tenants

---

## Health Endpoint

### Estrutura do Response

```json
{
  "status": "healthy",
  "components": {
    "database": {
      "initialized": true,
      "pool_max": 12,
      "pool_active": 3,
      "pool_idle": 9,
      "status": "healthy"
    }
  }
}
```

### Uso em Kubernetes

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 10
  periodSeconds: 30

readinessProbe:
  httpGet:
    path: /health
    port: 8000
  initialDelaySeconds: 5
  periodSeconds: 10
```

**Comportamento:**
- `status: "unhealthy"` → Kubernetes remove pod do load balancer
- Permite drain automático de pods com pool corrompido

---

## Referências

- [Runbook de Troubleshooting](../runbooks/database-pool-troubleshooting.md)
- [Structured Logging Guide](./structured-logging-guide.md)
- [Prometheus Best Practices](https://prometheus.io/docs/practices/)
- [Grafana Dashboards](https://grafana.com/docs/grafana/latest/dashboards/)

---

*Última atualização: 2026-03-13*
