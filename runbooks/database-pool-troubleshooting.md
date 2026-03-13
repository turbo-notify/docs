# Database Pool Troubleshooting Runbook

> **Relacionado:** [Database Metrics Guide](../observability/database-metrics-guide.md) | [API Standards](../standards/api-standards.md)

---

Guia operacional para diagnosticar e resolver problemas relacionados ao pool de conexões de banco de dados.

## 📋 Índice

- [Sintomas Comuns](#sintomas-comuns)
- [Diagnóstico Rápido](#diagnóstico-rápido)
- [Pool Exhaustion](#pool-exhaustion)
- [Protocol Violation](#protocol-violation)
- [Stale Connections](#stale-connections)
- [Latência Alta](#latência-alta)
- [Tenant-Aware Diagnostics](#tenant-aware-diagnostics)
- [Comandos Úteis](#comandos-úteis)

---

## Sintomas Comuns

### HTTP 500 - Erro Interno do Servidor

**Mensagens típicas nos logs:**
```
"Failed to acquire connection after 3 attempts"
"Pool timeout ao adquirir conexão"
"psycopg_pool.PoolTimeout: timeout in pool connection after 25s"
```

**Possíveis causas:**
1. Pool saturado (todas conexões ativas)
2. Timeouts desconfigurados
3. PgBouncer indisponível ou saturado
4. Tenant específico consumindo muitas conexões

**Próximos passos:** Ver [Pool Exhaustion](#pool-exhaustion)

---

### Protocol Violation - client_idle_timeout

**Mensagens típicas:**
```
"psycopg.errors.ProtocolViolation: client_idle_timeout"
"Protocol violation detectada (attempt 1/3)"
```

**Causa:**
- App mantém conexões idle por mais tempo que PgBouncer permite

**Próximos passos:** Ver [Protocol Violation](#protocol-violation)

---

### Latência Alta em Requests

**Sintomas:**
- Requests demoram 5-30s para responder
- Timeouts ocasionais
- Logs: "Pool timeout ao adquirir conexão (attempt X/3)"

**Causa:**
- Pool saturado forçando espera por conexão disponível
- Tenant específico com queries lentas

**Próximos passos:** Ver [Latência Alta](#latência-alta)

---

## Diagnóstico Rápido

### 1. Verificar Health Endpoint

```bash
curl http://localhost:8000/health | jq '.components.database'
```

**Saída esperada:**
```json
{
  "initialized": true,
  "pool_max": 8,
  "pool_active": 2,
  "pool_idle": 6,
  "status": "healthy"
}
```

**Interpretação:**
- `pool_active`: Conexões em uso no momento
- `pool_idle`: Conexões disponíveis
- `pool_max`: Tamanho máximo do pool

**Sinais de problema:**
- `pool_active` próximo de `pool_max` → Pool saturado
- `pool_idle = 0` → Sem conexões disponíveis
- `status: "unhealthy"` → Pool não consegue conectar ao banco

---

### 2. Verificar Logs Recentes

```bash
# Últimos erros críticos
docker logs turbo-public-api --since 10m 2>&1 | \
  grep -iE '(pool|timeout|protocol|database)' | tail -20

# Contagem de erros por tipo (últimos 30min)
docker logs turbo-public-api --since 30m 2>&1 | \
  grep -c "Pool timeout"

docker logs turbo-public-api --since 30m 2>&1 | \
  grep -c "ProtocolViolation"
```

---

### 3. Verificar PgBouncer

```bash
# Status do PgBouncer
docker exec turbo-pgbouncer sh -c \
  'psql -h localhost -p 6432 -U turbo_notify -d pgbouncer -c "SHOW POOLS;"'
```

**Saída esperada:**
```
 database     | user         | cl_active | cl_waiting | sv_active | sv_idle | sv_used | sv_tested | sv_login | maxwait
--------------+--------------+-----------+------------+-----------+---------+---------+-----------+----------+---------
 turbo_notify | turbo_notify |         8 |          0 |         5 |      25 |       0 |         0 |        0 |       0
```

**Interpretação:**
- `cl_active`: Conexões cliente ativas (soma de todos apps)
- `cl_waiting`: Clientes aguardando conexão (ALERTA se > 0)
- `sv_active`: Conexões server ativas (ao PostgreSQL)
- `sv_idle`: Conexões server idle disponíveis

**Sinais de problema:**
- `cl_waiting > 0` → PgBouncer saturado
- `sv_idle = 0` → Sem conexões server disponíveis

---

## Pool Exhaustion

### Sintoma
```
ERROR: Pool timeout ao adquirir conexão (attempt 1/3, duration=25.001s)
ERROR: Failed to acquire connection after 3 attempts
```

### Diagnóstico

**1. Verificar pool stats:**
```bash
curl http://localhost:8000/health | jq '.components.database.pool_active, .components.database.pool_max'
```

Se `pool_active == pool_max`, pool está **saturado**.

**2. Verificar se há requests lentos:**
```bash
# Logs de queries lentas
docker logs turbo-public-api --since 5m 2>&1 | \
  grep "duration" | awk '{print $NF}' | sort -n | tail -10
```

**3. Verificar PgBouncer:**
```bash
docker exec turbo-pgbouncer sh -c \
  'psql -h localhost -p 6432 -U turbo_notify -d pgbouncer -c "SHOW POOLS;"'
```

Se `cl_waiting > 0`, PgBouncer também está saturado.

**4. Identificar tenant causando saturação:**
```bash
# Ver logs com tenant_id
docker logs turbo-public-api --since 5m 2>&1 | \
  grep "Pool timeout" | grep -o '"tenant_id":"[^"]*"' | sort | uniq -c | sort -rn
```

---

### Resolução

#### Curto Prazo (Imediato)

**Opção A: Aumentar pool_size do app**

```bash
# Editar .env ou config
DATABASE_POOL_SIZE=12  # era 8

# Reiniciar serviço
docker restart turbo-public-api
```

**Opção B: Escalar horizontalmente (adicionar réplicas)**

```bash
# docker-compose.yml
services:
  public-api:
    deploy:
      replicas: 2  # era 1
```

**Opção C: Rate limit por tenant (se saturação por tenant específico)**

```python
# Implementar throttling
if tenant_pool_usage[tenant_id] > threshold:
    raise HTTPException(status_code=429, detail="Rate limit exceeded")
```

---

#### Médio Prazo (Investigação)

**1. Identificar queries lentas:**
```sql
-- Conectar ao PostgreSQL
psql postgresql://turbo_notify:PASSWORD@localhost/turbo_notify

-- Queries ativas há mais de 5s
SELECT pid, now() - query_start as duration, state, query
FROM pg_stat_activity
WHERE state != 'idle'
  AND now() - query_start > interval '5 seconds'
ORDER BY duration DESC;
```

**2. Identificar tenant consumindo mais recursos:**
```sql
-- Ver queries ativas por tenant
SELECT
  CASE
    WHEN query LIKE '%tenant_id%'
    THEN regexp_replace(query, '.*tenant_id[^'']*''([^'']+)''.*', '\1')
    ELSE 'unknown'
  END as tenant_id,
  count(*) as active_queries,
  avg(EXTRACT(EPOCH FROM (now() - query_start))) as avg_duration_seconds
FROM pg_stat_activity
WHERE state != 'idle'
  AND query NOT LIKE '%pg_stat_activity%'
GROUP BY tenant_id
ORDER BY active_queries DESC, avg_duration_seconds DESC;
```

**3. Otimizar queries identificadas:**
- Adicionar índices (sempre incluir `tenant_id` em índices compostos)
- Reescrever queries com JOIN em vez de N+1
- Implementar cache para dados frequentes
- Adicionar paginação (cursor-based se offset > 1000)

---

#### Longo Prazo (Arquitetura)

- Implementar read replicas para separar leitura/escrita
- Migrar queries lentas para workers assíncronos
- Implementar cache distribuído (Redis) para session state
- Particionar tabelas grandes por tenant_id

---

## Protocol Violation

### Sintoma
```
ERROR: psycopg.errors.ProtocolViolation: client_idle_timeout
WARNING: Protocol violation detectada (attempt 1/3)
```

### Causa Raiz
App mantém conexões idle por mais tempo que PgBouncer `client_idle_timeout` permite.

**Hierarquia de timeouts incorreta:**
```
App max_idle: 600s (10min)
PgBouncer client_idle_timeout: 300s (5min)
❌ PROBLEMA: App > PgBouncer
```

---

### Diagnóstico

**1. Verificar configuração atual:**
```bash
# App
grep DATABASE_POOL_MAX_IDLE .env

# PgBouncer
docker exec turbo-pgbouncer cat /etc/pgbouncer/pgbouncer.ini | grep client_idle_timeout
```

**2. Validar hierarquia:**
```
✅ CORRETO:
- App max_idle: 240s (4min)
- PgBouncer client_idle_timeout: 300s (5min)
- Relação: App < PgBouncer (com margem de 25%)

❌ INCORRETO:
- App max_idle: 600s
- PgBouncer client_idle_timeout: 300s
- Relação: App > PgBouncer
```

---

### Resolução

**Opção A: Reduzir app max_idle (RECOMENDADO)**

```bash
# .env
DATABASE_POOL_MAX_IDLE=240  # 4min (< PgBouncer 5min)

# Reiniciar apps
docker restart turbo-public-api turbo-orchestrator
```

**Opção B: Aumentar PgBouncer client_idle_timeout**

```bash
# /etc/pgbouncer/pgbouncer.ini
client_idle_timeout = 900  # 15min

# Reload PgBouncer (sem downtime)
docker exec turbo-pgbouncer sh -c 'kill -HUP 1'
```

**⚠️ IMPORTANTE:** Sempre mantenha `app max_idle < PgBouncer client_idle_timeout`

**Multiplicadores recomendados:**
- `PgBouncer client_idle_timeout = app max_idle × 1.25`
- `PgBouncer server_idle_timeout = app max_idle × 2.5`

---

## Stale Connections

### Sintoma
```
WARNING: Connection validation failed: connection closed
WARNING: Connection validation failed: transaction status UNKNOWN
```

### Causa
Pool entregou conexão que já foi fechada pelo servidor (ex: max_lifetime expirou).

---

### Diagnóstico

**1. Verificar configuração de lifecycle:**
```bash
grep -E 'MAX_IDLE|MAX_LIFETIME' .env
```

**Hierarquia recomendada:**
```
max_idle: 240s (4min)
max_lifetime: 600s (10min)
PgBouncer server_idle_timeout: 600s (10min)
```

**2. Verificar logs de PgBouncer:**
```bash
docker logs turbo-pgbouncer --since 10m 2>&1 | \
  grep -E '(closing|disconnect)'
```

---

### Resolução

**Configuração já implementa validação automática:**
```python
# client.py
async def _check_connection(self, conn):
    """Valida conexão antes de entregar."""
    if conn.closed:
        raise OperationalError("Connection is closed")
    if conn.info.transaction_status == UNKNOWN:
        raise OperationalError("Transaction status is UNKNOWN")
```

**Se problema persistir:**

1. Reduzir `max_lifetime` para forçar reciclagem mais frequente
2. Verificar se PgBouncer está reiniciando inesperadamente
3. Verificar logs do PostgreSQL para server crashes

---

## Latência Alta

### Sintoma
- Requests demoram 10-30s
- P95/P99 de latência muito alto
- Logs: "Pool timeout ao adquirir conexão"

---

### Diagnóstico

**1. Verificar se pool está saturado:**
```bash
# Loop monitorando pool a cada 5s
while true; do
  curl -s http://localhost:8000/health | \
    jq '.components.database | {active, idle, max}'
  sleep 5
done
```

**2. Verificar queries lentas no PostgreSQL:**
```sql
SELECT query, calls, mean_exec_time, max_exec_time
FROM pg_stat_statements
WHERE mean_exec_time > 1000  -- > 1s
ORDER BY mean_exec_time DESC
LIMIT 10;
```

**3. Verificar contenção de lock:**
```sql
SELECT blocked_locks.pid AS blocked_pid,
       blocking_locks.pid AS blocking_pid,
       blocked_activity.query AS blocked_query,
       blocking_activity.query AS blocking_query
FROM pg_locks blocked_locks
JOIN pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
JOIN pg_locks blocking_locks ON blocking_locks.locktype = blocked_locks.locktype
JOIN pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
WHERE NOT blocked_locks.granted
  AND blocking_locks.granted;
```

**4. Verificar se tenant específico está causando latência:**
```sql
-- P99 de latência de session queries por tenant
SELECT
  CASE
    WHEN query LIKE '%sessions%' AND query LIKE '%tenant_id%'
    THEN regexp_replace(query, '.*tenant_id[^'']*''([^'']+)''.*', '\1')
    ELSE 'unknown'
  END as tenant_id,
  count(*) as session_queries,
  percentile_cont(0.99) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (now() - query_start))) as p99_duration
FROM pg_stat_activity
WHERE state != 'idle'
  AND query LIKE '%sessions%'
GROUP BY tenant_id
HAVING count(*) > 5
ORDER BY p99_duration DESC;
```

---

### Resolução

**Se pool saturado:**
- Aumentar `pool_size`
- Escalar horizontalmente (mais réplicas)
- Otimizar queries lentas

**Se queries lentas:**
- Adicionar índices (sempre incluir `tenant_id` em índices compostos)
- Reescrever queries ineficientes
- Implementar cache para session state

**Se contenção de lock:**
- Reduzir escopo de transações
- Usar `SELECT ... FOR UPDATE SKIP LOCKED`
- Particionar tabelas grandes por `tenant_id`

**Se tenant específico saturando:**
- Implementar rate limiting por tenant
- Otimizar queries específicas daquele tenant
- Considerar tenant-specific pool (se enterprise)

---

## Tenant-Aware Diagnostics

### Identificar Tenant Saturando Pool

**1. Ver queries ativas por tenant:**
```sql
SELECT
  CASE
    WHEN query LIKE '%tenant_id%'
    THEN regexp_replace(query, '.*tenant_id[^'']*''([^'']+)''.*', '\1')
    ELSE 'unknown'
  END as tenant_id,
  count(*) as active_connections,
  round(100.0 * count(*) / (SELECT count(*) FROM pg_stat_activity WHERE state != 'idle'), 2) as pct_of_pool
FROM pg_stat_activity
WHERE state != 'idle'
  AND query NOT LIKE '%pg_stat_activity%'
GROUP BY tenant_id
ORDER BY active_connections DESC;
```

**2. Ver latência média por tenant:**
```sql
SELECT
  CASE
    WHEN query LIKE '%tenant_id%'
    THEN regexp_replace(query, '.*tenant_id[^'']*''([^'']+)''.*', '\1')
    ELSE 'unknown'
  END as tenant_id,
  count(*) as queries,
  round(avg(EXTRACT(EPOCH FROM (now() - query_start))), 2) as avg_duration_seconds,
  round(max(EXTRACT(EPOCH FROM (now() - query_start))), 2) as max_duration_seconds
FROM pg_stat_activity
WHERE state != 'idle'
  AND query NOT LIKE '%pg_stat_activity%'
GROUP BY tenant_id
ORDER BY avg_duration_seconds DESC;
```

**3. Logs estruturados com tenant_id:**
```bash
# Ver pool timeouts por tenant (últimos 10min)
docker logs turbo-public-api --since 10m 2>&1 | \
  grep "Pool timeout" | \
  grep -o '"tenant_id":"[^"]*"' | \
  sort | uniq -c | sort -rn | head -10
```

### Session State Queries Performance

**1. Verificar latência de queries de sessão:**
```sql
-- P95 de queries de sessão
SELECT
  CASE
    WHEN query LIKE '%sessions%' AND query LIKE '%status%'
    THEN 'session_status_query'
    WHEN query LIKE '%sessions%' AND query LIKE '%INSERT%'
    THEN 'session_insert'
    WHEN query LIKE '%sessions%' AND query LIKE '%UPDATE%'
    THEN 'session_update'
    ELSE 'other_session_query'
  END as query_type,
  count(*) as count,
  round(percentile_cont(0.95) WITHIN GROUP (ORDER BY EXTRACT(EPOCH FROM (now() - query_start))), 3) as p95_seconds
FROM pg_stat_activity
WHERE state != 'idle'
  AND query LIKE '%sessions%'
GROUP BY query_type
ORDER BY p95_seconds DESC;
```

**2. Índices recomendados para session queries:**
```sql
-- Tenant isolation + session lookup
CREATE INDEX CONCURRENTLY idx_sessions_tenant_id_id
  ON sessions (tenant_id, id);

-- Status filtering
CREATE INDEX CONCURRENTLY idx_sessions_tenant_id_status
  ON sessions (tenant_id, status)
  WHERE status IN ('ACTIVE', 'CONNECTING');

-- Worker assignment
CREATE INDEX CONCURRENTLY idx_sessions_worker_id
  ON sessions (worker_id)
  WHERE worker_id IS NOT NULL;
```

### Message Insertion Backlog

**1. Verificar latência de inserção de mensagens:**
```sql
-- Queries de INSERT em messages por tenant
SELECT
  CASE
    WHEN query LIKE '%tenant_id%'
    THEN regexp_replace(query, '.*tenant_id[^'']*''([^'']+)''.*', '\1')
    ELSE 'unknown'
  END as tenant_id,
  count(*) as insert_queries,
  round(avg(EXTRACT(EPOCH FROM (now() - query_start))), 3) as avg_duration_seconds,
  round(max(EXTRACT(EPOCH FROM (now() - query_start))), 3) as max_duration_seconds
FROM pg_stat_activity
WHERE state != 'idle'
  AND query LIKE '%INSERT INTO messages%'
GROUP BY tenant_id
ORDER BY avg_duration_seconds DESC;
```

**2. Verificar locks em messages table:**
```sql
SELECT
  a.pid,
  a.query_start,
  a.state,
  l.mode,
  l.granted,
  a.query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.relation = 'messages'::regclass
ORDER BY a.query_start;
```

**3. Índices recomendados para message operations:**
```sql
-- Message lookup by tenant + session
CREATE INDEX CONCURRENTLY idx_messages_tenant_session
  ON messages (tenant_id, session_id, created_at DESC);

-- Message status tracking
CREATE INDEX CONCURRENTLY idx_messages_status
  ON messages (status, created_at DESC)
  WHERE status IN ('PENDING', 'SENDING');
```

---

## Comandos Úteis

### Monitoramento Contínuo

```bash
# Loop monitorando saúde a cada 10s
watch -n 10 'curl -s http://localhost:8000/health | jq ".components.database"'

# Stream de logs com filtro
docker logs -f turbo-public-api 2>&1 | \
  grep -E '(pool|timeout|error|warning|tenant_id)'

# Contagem de erros em tempo real
watch -n 5 'docker logs turbo-public-api --since 5m 2>&1 | \
  grep -c "Pool timeout"'
```

---

### PgBouncer

```bash
# Ver pools ativos
docker exec turbo-pgbouncer sh -c \
  'psql -h localhost -p 6432 -U turbo_notify -d pgbouncer -c "SHOW POOLS;"'

# Ver estatísticas
docker exec turbo-pgbouncer sh -c \
  'psql -h localhost -p 6432 -U turbo_notify -d pgbouncer -c "SHOW STATS;"'

# Ver configuração atual
docker exec turbo-pgbouncer sh -c \
  'psql -h localhost -p 6432 -U turbo_notify -d pgbouncer -c "SHOW CONFIG;"'

# Reload configuração (sem downtime)
docker exec turbo-pgbouncer sh -c 'kill -HUP 1'
```

---

### PostgreSQL

```bash
# Conexões ativas por aplicação
psql $DATABASE_URL -c "
  SELECT application_name, count(*)
  FROM pg_stat_activity
  WHERE state != 'idle'
  GROUP BY application_name;"

# Queries lentas (> 1s)
psql $DATABASE_URL -c "
  SELECT pid, now() - query_start as duration, query
  FROM pg_stat_activity
  WHERE state != 'idle'
    AND now() - query_start > interval '1 second'
  ORDER BY duration DESC;"

# Terminar query específica
psql $DATABASE_URL -c "SELECT pg_terminate_backend(PID);"

# Ver queries por tenant (requer query com tenant_id)
psql $DATABASE_URL -c "
  SELECT
    CASE
      WHEN query LIKE '%tenant_id%'
      THEN regexp_replace(query, '.*tenant_id[^'']*''([^'']+)''.*', '\1')
      ELSE 'unknown'
    END as tenant_id,
    count(*) as active_queries
  FROM pg_stat_activity
  WHERE state != 'idle'
    AND query NOT LIKE '%pg_stat_activity%'
  GROUP BY tenant_id
  ORDER BY active_queries DESC;"
```

---

## Escalação

### Severidade 1 (CRÍTICO)
- **Sintoma:** API completamente fora do ar (todas requests HTTP 500)
- **SLA:** Resolução em < 15min
- **Ação:** Aumentar pool_size imediatamente + escalar horizontalmente

### Severidade 2 (ALTO)
- **Sintoma:** Taxa de erro > 5%, latência P95 > 5s
- **SLA:** Resolução em < 1h
- **Ação:** Investigar queries lentas + identificar tenant saturando + otimizar

### Severidade 3 (MÉDIO)
- **Sintoma:** Warnings ocasionais de protocol violation
- **SLA:** Resolução em < 4h
- **Ação:** Ajustar hierarquia de timeouts

### Severidade 4 (BAIXO)
- **Sintoma:** Tenant específico com latência alta (não afeta outros)
- **SLA:** Resolução em < 8h
- **Ação:** Otimizar queries daquele tenant + considerar rate limiting

---

## Contatos

**Equipe de Plataforma:** #plataforma-alerts (Slack)
**Oncall:** @oncall-plataforma

**Documentação relacionada:**
- [Database Metrics Guide](../observability/database-metrics-guide.md)
- [Structured Logging Guide](../observability/structured-logging-guide.md)
- [API Standards](../standards/api-standards.md)

---

*Última atualização: 2026-03-13*
