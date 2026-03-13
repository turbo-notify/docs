# Structured Logging Guide

> **Relacionado:** [Database Metrics Guide](./database-metrics-guide.md) | [API Standards](../standards/api-standards.md)

---

Guia completo para usar logging estruturado (JSON) na plataforma Turbo Notify.

## 📋 Índice

- [Visão Geral](#visão-geral)
- [Configuração](#configuração)
- [Uso Básico](#uso-básico)
- [Formato de Logs](#formato-de-logs)
- [Turbo Notify Specific Patterns](#turbo-notify-specific-patterns)
- [Integração com Loki](#integração-com-loki)
- [Boas Práticas](#boas-práticas)
- [Troubleshooting](#troubleshooting)

---

## Visão Geral

### Por Que Logging Estruturado?

**Logs tradicionais (texto não estruturado):**
```
2026-03-13 15:30:45 [WARNING] Pool timeout ao adquirir conexão (attempt 1/3, duration=25.001s)
```

**Problemas:**
- Difícil de parsear (regex frágil)
- Campos não padronizados
- Queries complexas no Loki/Elasticsearch
- Difícil correlacionar logs relacionados

---

**Logs estruturados (JSON):**
```json
{
  "timestamp": "2026-03-13T15:30:45.123Z",
  "level": "WARNING",
  "logger": "shared_core.infrastructure.db.client",
  "message": "Pool timeout ao adquirir conexão",
  "service": "public-api",
  "component": "database",
  "operation": "acquire_connection",
  "tenant_id": "tenant_123",
  "attempt": 1,
  "max_retries": 3,
  "duration_seconds": 25.001,
  "pool_size": 8,
  "timeout": 25
}
```

**Vantagens:**
- ✅ Fácil de parsear (JSON nativo)
- ✅ Campos padronizados e tipados
- ✅ Queries simples no Loki (`{component="database"} | json | duration_seconds > 10`)
- ✅ Correlação por campos (trace_id, request_id, tenant_id)
- ✅ Agregações e métricas derivadas de logs
- ✅ **Tenant isolation**: Filtrar logs por tenant_id

---

## Configuração

### 1. Instalar Dependência

A biblioteca `python-json-logger` é usada para formatar logs em JSON.

**Adicionar ao pyproject.toml:**
```toml
[tool.poetry.dependencies]
python-json-logger = "^2.0.0"
```

**Instalar:**
```bash
poetry add python-json-logger
```

---

### 2. Configurar no Startup

**FastAPI (public-api, landing-api, dashboard-api):**
```python
# main.py
from shared_core.infrastructure.logging import configure_structured_logging
from contextlib import asynccontextmanager
from fastapi import FastAPI

@asynccontextmanager
async def lifespan(app: FastAPI):
    # Configurar structured logging no startup
    configure_structured_logging(
        level="INFO",
        service_name="public-api",
        use_json=True,
    )

    yield

app = FastAPI(lifespan=lifespan)
```

---

**Session Worker (future):**
```python
# worker.py
from shared_core.infrastructure.logging import configure_structured_logging

async def main():
    # Configurar logging antes de qualquer operação
    configure_structured_logging(
        level="INFO",
        service_name="session-worker",
        use_json=True,
    )

    # Iniciar worker...
    await run_worker()

if __name__ == "__main__":
    asyncio.run(main())
```

---

### 3. Variável de Ambiente

**Controlar formato via env var (opcional):**
```bash
# .env
LOG_FORMAT=json  # ou "text" para fallback
LOG_LEVEL=INFO
```

**Aplicar:**
```python
import os
from shared_core.infrastructure.logging import configure_structured_logging

configure_structured_logging(
    level=os.getenv("LOG_LEVEL", "INFO"),
    service_name="public-api",
    use_json=os.getenv("LOG_FORMAT", "json") == "json",
)
```

---

## Uso Básico

### Logging Simples

```python
import logging

logger = logging.getLogger(__name__)

# Log simples
logger.info("Session created successfully")

# Log com campos estruturados
logger.warning(
    "Session connection timeout",
    extra={
        "tenant_id": "tenant_123",
        "session_id": "sess_abc123",
        "attempt": 1,
        "max_retries": 3,
        "duration_seconds": 25.001,
    }
)
```

**Output JSON:**
```json
{
  "timestamp": "2026-03-13T15:30:45.123Z",
  "level": "WARNING",
  "logger": "application.use_cases.sessions.create_session",
  "message": "Session connection timeout",
  "service": "public-api",
  "component": "sessions",
  "tenant_id": "tenant_123",
  "session_id": "sess_abc123",
  "attempt": 1,
  "max_retries": 3,
  "duration_seconds": 25.001
}
```

---

### Logger com Contexto Padrão

Para evitar repetir campos em todo log:

```python
from shared_core.infrastructure.logging import get_logger

# Logger com contexto padrão
logger = get_logger(
    __name__,
    component="sessions",
    operation="create_session",
)

# Todos os logs terão component="sessions" e operation="create_session"
logger.warning("Timeout", extra={"attempt": 1, "duration": 25.0})
```

**Output JSON:**
```json
{
  "timestamp": "2026-03-13T15:30:45.123Z",
  "level": "WARNING",
  "message": "Timeout",
  "component": "sessions",
  "operation": "create_session",
  "attempt": 1,
  "duration": 25.0
}
```

---

### Campos Padrão

Todos os logs incluem automaticamente:

| Campo | Tipo | Descrição | Exemplo |
|-------|------|-----------|---------|
| `timestamp` | string | ISO8601 UTC | `"2026-03-13T15:30:45.123Z"` |
| `level` | string | Nível de log | `"INFO"`, `"WARNING"`, `"ERROR"` |
| `logger` | string | Nome do logger | `"application.use_cases.sessions.create_session"` |
| `message` | string | Mensagem principal | `"Session created"` |
| `service` | string | Nome do serviço | `"public-api"`, `"session-worker"` |
| `component` | string | Componente afetado | `"sessions"`, `"messages"`, `"webhooks"` |

---

## Formato de Logs

### Estrutura Completa

```json
{
  // === Campos Obrigatórios (sempre presentes) ===
  "timestamp": "2026-03-13T15:30:45.123456Z",
  "level": "WARNING",
  "logger": "application.use_cases.sessions.create_session",
  "message": "Session connection timeout",
  "service": "public-api",
  "component": "sessions",

  // === Campos Contextuais (via extra={}) ===
  "operation": "create_session",
  "tenant_id": "tenant_123",
  "session_id": "sess_abc123",
  "attempt": 1,
  "max_retries": 3,
  "duration_seconds": 25.001,

  // === Campos de Trace (opcional, OpenTelemetry) ===
  "trace_id": "abc123...",
  "span_id": "def456...",
  "parent_span_id": "ghi789...",

  // === Campos de Request (HTTP APIs) ===
  "request_id": "req-uuid-...",
  "method": "POST",
  "path": "/v1/sessions/create",
  "status_code": 500,
  "client_ip": "203.0.113.45"
}
```

---

### Convenções de Nomenclatura

**Campos numéricos:**
- Use sufixo `_seconds` para duração: `duration_seconds`, `timeout_seconds`
- Use sufixo `_bytes` para tamanho: `response_size_bytes`, `message_size_bytes`
- Use sufixo `_count` para contagens: `retry_count`, `error_count`, `message_count`

**Campos booleanos:**
- Use prefixo `is_`: `is_retry`, `is_timeout`, `is_connected`
- Ou sufixo `_enabled`: `cache_enabled`, `webhooks_enabled`

**IDs e identificadores:**
- Use sufixo `_id`: `request_id`, `trace_id`, `tenant_id`, `session_id`, `message_id`, `webhook_id`

---

## Turbo Notify Specific Patterns

### Session State Logging

**SEMPRE incluir `tenant_id` e `session_id` em logs relacionados a sessões:**

```python
logger.info(
    "Session state transition",
    extra={
        "tenant_id": tenant_id,
        "session_id": session_id,
        "from_state": "PENDING",
        "to_state": "CONNECTING",
        "worker_id": "worker-1",
        "duration_seconds": 0.5,
    }
)
```

**Estados de sessão válidos:**
- `PENDING` → `CONNECTING` → `ACTIVE` → `DISCONNECTED` → `TERMINATED`

**Exemplo de query Loki:**
```logql
{service="public-api", component="sessions"}
| json
| to_state="ACTIVE"
| tenant_id="tenant_123"
```

---

### Message Tracking

**Correlacionar message_id em toda a pipeline (API → NATS → Worker → WhatsApp → Webhook):**

```python
# Control Plane API
logger.info(
    "Message send requested",
    extra={
        "tenant_id": tenant_id,
        "session_id": session_id,
        "message_id": message_id,
        "to": "+5511999999999",
        "content_type": "text",
    }
)

# Session Worker
logger.info(
    "Message sent to WhatsApp",
    extra={
        "tenant_id": tenant_id,
        "session_id": session_id,
        "message_id": message_id,
        "whatsapp_message_id": "wamid.abc123",
        "duration_seconds": 1.2,
    }
)

# Webhook Dispatcher
logger.info(
    "Webhook delivered",
    extra={
        "tenant_id": tenant_id,
        "message_id": message_id,
        "webhook_id": "webhook_xyz",
        "target_url": "https://customer.com/webhook",
        "status_code": 200,
        "attempt": 1,
    }
)
```

**Rastrear toda a jornada de uma mensagem:**
```logql
{service=~"public-api|session-worker|webhook-dispatcher"}
| json
| message_id="msg_abc123"
```

---

### Webhook Delivery Logging

**Log retry attempts, status codes, delivery confirmations:**

```python
logger.warning(
    "Webhook delivery failed, will retry",
    extra={
        "tenant_id": tenant_id,
        "webhook_id": webhook_id,
        "target_url": target_url,
        "status_code": 503,
        "attempt": 2,
        "max_retries": 5,
        "next_retry_seconds": 60,
        "error": "Service Unavailable",
    }
)
```

**Query para analisar falhas de webhook:**
```logql
{component="webhooks"}
| json
| status_code >= 500
| tenant_id="tenant_123"
```

---

### Tenant Isolation (CRÍTICO)

**TODA operação deve incluir `tenant_id` nos logs:**

```python
# ✅ CORRETO
logger.info(
    "Query executed",
    extra={
        "tenant_id": tenant_id,
        "query": "SELECT * FROM sessions WHERE tenant_id = ?",
        "duration_seconds": 0.05,
    }
)

# ❌ ERRADO - Missing tenant_id
logger.info(
    "Query executed",
    extra={
        "query": "SELECT * FROM sessions",
        "duration_seconds": 0.05,
    }
)
```

**Por quê?**
- Permite isolar logs por tenant (compliance, debugging)
- Identifica qual tenant está causando problemas (pool saturation, rate limiting)
- Permite SLA por tenant

---

## Integração com Loki

### Query de Logs no Grafana

**Todos os logs de um serviço:**
```logql
{service="public-api"}
```

**Logs de um componente específico:**
```logql
{service="public-api", component="sessions"}
```

**Filtrar por nível:**
```logql
{service="public-api"} | json | level="ERROR"
```

**Logs com duração > 10s:**
```logql
{component="database"} | json | duration_seconds > 10
```

**Logs de um tenant específico:**
```logql
{service="public-api"} | json | tenant_id="tenant_123"
```

**Taxa de erros por serviço (últimos 5min):**
```logql
sum by (service) (
  rate({level="ERROR"}[5m])
)
```

**P95 de duração de aquisição de conexão:**
```logql
quantile_over_time(
  0.95,
  {operation="acquire_connection"} | json | unwrap duration_seconds [5m]
)
```

---

### Agregações e Métricas

Loki permite derivar métricas de logs estruturados:

**Contagem de session timeouts por tenant:**
```logql
sum by (tenant_id) (
  count_over_time({component="sessions"} | json | message=~"(?i)timeout"[5m])
)
```

**Média de tentativas de retry por tenant:**
```logql
avg by (tenant_id) (
  avg_over_time({operation="send_message"} | json | unwrap attempt [5m])
)
```

**Dashboard Panel (Grafana):**
```json
{
  "targets": [
    {
      "expr": "sum by (tenant_id) (count_over_time({level=\"ERROR\"}[5m]))",
      "legendFormat": "{{tenant_id}}"
    }
  ]
}
```

---

## Boas Práticas

### 1. Use `extra={}` para Campos Estruturados

❌ **NÃO FAÇA:**
```python
logger.warning(f"Session timeout (tenant={tenant_id}, attempt={attempt})")
```

✅ **FAÇA:**
```python
logger.warning(
    "Session timeout",
    extra={
        "tenant_id": tenant_id,
        "attempt": attempt,
        "duration_seconds": duration,
    }
)
```

**Motivo:** Campos estruturados são parseáveis e queryáveis no Loki.

---

### 2. Use Nomes Consistentes

❌ **INCONSISTENTE:**
```python
# worker_1.py
logger.info("Processing", extra={"id_session": session_id})

# worker_2.py
logger.info("Processing", extra={"session_id": session_id})
```

✅ **CONSISTENTE:**
```python
# Todos usam session_id
logger.info("Processing session", extra={"session_id": session_id})
```

**Motivo:** Queries do Loki dependem de nomes consistentes.

---

### 3. Evite Valores Dinâmicos em Mensagens

❌ **NÃO FAÇA:**
```python
logger.error(f"Failed to process session {session_id}")
```

✅ **FAÇA:**
```python
logger.error("Failed to process session", extra={"session_id": session_id})
```

**Motivo:** Mensagens genéricas permitem agregação por tipo de erro.

---

### 4. Tipos de Dados Corretos

❌ **NÃO FAÇA:**
```python
logger.info("Query slow", extra={"duration": "25.001s"})  # string
```

✅ **FAÇA:**
```python
logger.info("Query slow", extra={"duration_seconds": 25.001})  # float
```

**Motivo:** Loki pode fazer math operations em números.

---

### 5. Níveis de Log Apropriados

| Nível | Quando Usar | Exemplo |
|-------|-------------|---------|
| `DEBUG` | Detalhes internos (dev) | `"Acquired connection from pool"` |
| `INFO` | Eventos normais de negócio | `"Session created successfully"` |
| `WARNING` | Situações anormais recuperáveis | `"Session timeout, retrying (1/3)"` |
| `ERROR` | Falhas que impedem operação | `"Failed to send message after 3 retries"` |
| `CRITICAL` | Sistema em estado crítico | `"Database pool exhausted, API down"` |

---

### 6. Nunca Use `print()` para Logging

❌ **NÃO FAÇA:**
```python
print("[DEBUG] Processing session:", session_id)
print(f"tenant_id: {tenant_id}")
print(f"state: {state}")
```

✅ **FAÇA:**
```python
logger.debug(
    "Processing session",
    extra={
        "tenant_id": tenant_id,
        "session_id": session_id,
        "state": state,
    }
)
```

**Motivo:**
- `print()` não é estruturado e não aparece no Loki/Grafana
- `print()` não tem níveis (INFO/ERROR)
- `print()` não tem contexto (service, component, tenant_id)
- `print()` polui stdout e não é queryável

---

### 7. Nunca Use `traceback.print_exc()`

❌ **NÃO FAÇA:**
```python
try:
    result = await send_message()
except Exception as exc:
    traceback.print_exc()  # Imprime no stdout
    return JSONResponse(status_code=500, content={"error": "Internal error"})
```

✅ **FAÇA:**
```python
try:
    result = await send_message()
except Exception as exc:
    logger.error(
        "Failed to send message",
        extra={
            "tenant_id": tenant_id,
            "session_id": session_id,
            "message_id": message_id,
            "error_type": exc.__class__.__name__,
            "error": str(exc),
        },
        exc_info=True  # Inclui traceback no log estruturado
    )
    return JSONResponse(status_code=500, content={"error": "Internal error"})
```

**Motivo:**
- `traceback.print_exc()` não é estruturado
- Não é capturado por sistemas de logging centralizados
- `exc_info=True` inclui traceback no JSON de forma estruturada

---

### 8. SEMPRE Incluir `tenant_id`

❌ **NÃO FAÇA:**
```python
logger.info("Message sent", extra={"message_id": message_id})
```

✅ **FAÇA:**
```python
logger.info(
    "Message sent",
    extra={
        "tenant_id": tenant_id,  # OBRIGATÓRIO
        "message_id": message_id,
    }
)
```

**Motivo:**
- Tenant isolation (compliance, debugging)
- Identificar qual tenant está causando problemas
- SLA por tenant

---

## Troubleshooting

### Logs Não Aparecem no JSON

**Sintoma:** Logs saem em texto plano, não JSON.

**Causa:** `python-json-logger` não instalado.

**Solução:**
```bash
poetry add python-json-logger
```

---

### Campos `extra` Não Aparecem

**Sintoma:** Campos passados em `extra={}` não aparecem no JSON.

**Causa:** Logger não configurado corretamente.

**Solução:** Chamar `configure_structured_logging()` no startup.

---

### Logs Duplicados

**Sintoma:** Cada log aparece 2x.

**Causa:** Handlers duplicados (root logger já tinha handler).

**Solução:** `configure_structured_logging()` limpa handlers existentes automaticamente.

---

### Performance em Produção

**Sintoma:** JSON serialization lenta.

**Solução:** Usar nível INFO em produção (DEBUG apenas em dev):
```python
configure_structured_logging(level="INFO")  # não DEBUG
```

---

## Referências

- [python-json-logger](https://github.com/madzak/python-json-logger)
- [Loki LogQL](https://grafana.com/docs/loki/latest/logql/)
- [Structured Logging Best Practices](https://www.honeycomb.io/blog/how-to-structure-logs)
- [Database Pool Troubleshooting](../runbooks/database-pool-troubleshooting.md)
- [NATS Troubleshooting](../runbooks/nats-troubleshooting.md)

---

*Última atualização: 2026-03-13*
