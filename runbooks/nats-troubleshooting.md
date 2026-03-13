# NATS JetStream Troubleshooting Runbook

> **Relacionado:** [NATS Events](../architecture/nats-events.md) | [Session Lifecycle](../architecture/session-lifecycle.md)

## Visão Geral

Este runbook documenta procedimentos de troubleshooting para o NATS JetStream no Turbo Notify. O NATS é utilizado para mensageria entre os componentes da arquitetura event-driven (Control Plane API, Session Workers, Orchestrator).

---

## 1. Comandos Úteis NATS CLI

### 1.1 Verificar Conexão

```bash
# Verificar se NATS está acessível
nats server ping

# Ver informações do servidor
nats server info
```

### 1.2 Listar Streams

```bash
# Listar todos os streams
nats stream list

# Ver detalhes de streams específicos
nats stream info turbo_sessions_create
nats stream info turbo_sessions_events
nats stream info turbo_messages_send
nats stream info turbo_messages_events
nats stream info turbo_webhooks_deliver
nats stream info turbo_dead_letter
```

### 1.3 Listar Consumers

```bash
# Listar consumers de um stream
nats consumer list turbo_sessions_create
nats consumer list turbo_sessions_events
nats consumer list turbo_messages_send

# Ver detalhes de um consumer
nats consumer info turbo_sessions_create public-api-sessions-consumer
nats consumer info turbo_sessions_events orchestrator-events-consumer
nats consumer info turbo_messages_send session-worker-messages-consumer
```

### 1.4 Ver Mensagens

```bash
# Ver mensagens em um stream (NÃO consome, apenas visualiza)
nats stream view turbo_sessions_create --last 10
nats stream view turbo_messages_send --last 10
nats stream view turbo_dead_letter --last 10

# Ver mensagem específica por sequência
nats stream get turbo_sessions_create 1
```

---

## 2. Diagnóstico de Problemas Comuns

### 2.1 Consumer Lag Alto

**Sintoma**: Mensagens acumulando no stream, consumers não processando.

**Diagnóstico**:
```bash
# Verificar pending messages por consumer
nats consumer info turbo_sessions_create public-api-sessions-consumer
nats consumer info turbo_messages_send session-worker-messages-consumer

# Campos importantes:
# - Num Pending: mensagens aguardando processamento
# - Num Ack Pending: mensagens em processamento (aguardando ACK)
# - Num Redelivered: mensagens reentregues (indicador de falhas)
```

**Causas comuns**:
1. Session workers offline ou crashando
2. Processamento lento (WhatsApp Web automation delay)
3. Erros de banco de dados (pool saturado)
4. Rate limiting de WhatsApp Web

**Ações**:
```bash
# Verificar status dos workers
kubectl get pods -l app=session-worker
# ou via docker
docker ps | grep session-worker

# Verificar logs dos workers
kubectl logs -l app=session-worker --tail=100
# ou via docker
docker logs session-worker --tail 100

# Verificar logs do Control Plane API
docker logs turbo-public-api --tail 100

# Verificar metricas no Grafana
# Dashboard: NATS > Consumers > Mensagens Pendentes por Consumer
```

### 2.2 Mensagens na Dead Letter Queue

**Sintoma**: Alerta de mensagens na DLQ (threshold: >0).

**Diagnóstico**:
```bash
# Ver mensagens na DLQ
nats stream view turbo_dead_letter --last 20

# Contar mensagens na DLQ
nats stream info turbo_dead_letter | grep "Messages:"
```

**Visualização no Grafana**:
- Dashboard: NATS > Dead Letter Queue (DLQ)
- Painéis: Mensagens na DLQ, Tamanho, Timeline, Taxa de entrada
- URL: https://grafana.turbo-notify.com/d/nats/nats

**Formato da mensagem DLQ**:
```json
{
  "original_stream": "turbo_sessions_events",
  "original_subject": "turbo.sessions.connected",
  "original_seq": 123,
  "consumer": "session-worker-events-consumer",
  "deliveries": 20,
  "dead_at": 1710345600.123,
  "original_data": "{ ... mensagem original ... }",
  "is_binary": false,
  "tenant_id": "tenant_abc123",
  "session_id": "session_xyz789"
}
```

**Como mensagens chegam na DLQ**:
1. Worker/Consumer tenta processar mensagem
2. Após `max_deliver` tentativas (20 para session-worker, 5 para outros), NATS emite Advisory Message
3. DeadLetterHandler no Orchestrator captura e copia para stream `turbo_dead_letter`
4. Alerta Grafana dispara imediatamente (threshold > 0)

**Ações**:
1. Verificar Dashboard Grafana > NATS > Dead Letter Queue para detalhes
2. Analisar `original_data` para entender o erro
3. Verificar logs do consumer correspondente
4. Verificar se é problema de tenant específico (`tenant_id` no payload)
5. Corrigir o problema raiz
6. Reprocessar via script (se implementado)

**Reprocessamento manual (exemplo)**:
```bash
# Ver mensagem específica
nats stream get turbo_dead_letter 123

# Republicar manualmente (se necessário)
# Extrair original_data e republicar no subject correto
nats pub turbo.sessions.create.requested '{"tenant_id":"...","session_id":"..."}'
```

### 2.3 Session Worker Não Consome Mensagens

**Sintoma**: Stream `turbo_sessions_create` ou `turbo_messages_send` acumula mensagens, mas workers estão rodando.

**Diagnóstico**:
```bash
# Verificar se workers estão conectados ao NATS
nats consumer info turbo_sessions_create session-worker-sessions-consumer

# Verificar logs do worker
kubectl logs -l app=session-worker --tail=200 | grep -i nats
# ou docker
docker logs session-worker --tail 200 | grep -i nats

# Verificar conexões NATS
nats server report connections | grep session-worker
```

**Causas comuns**:
1. Worker não consegue conectar ao NATS (NATS_URL incorreta)
2. Consumer não criado (inicialização falhou)
3. Worker está ocupado com sessões existentes (sem capacidade)
4. Erro de autenticação/autorização NATS
5. Crash loop (worker morre antes de processar)

**Ações**:
```bash
# Verificar variáveis de ambiente do worker
kubectl describe pod <worker-pod-name> | grep NATS_URL
# ou docker
docker inspect session-worker | grep NATS_URL

# Verificar health do worker
curl http://session-worker:8080/health

# Verificar capacidade do worker (sessões ativas)
# (depende da implementação de metrics)

# Reiniciar worker
kubectl rollout restart deployment/session-worker
# ou docker
docker restart session-worker
```

### 2.4 Message Delivery Lag

**Sintoma**: Mensagens enviadas pela API demoram muito para serem processadas pelos workers.

**Diagnóstico**:
```bash
# Verificar lag no consumer de mensagens
nats consumer info turbo_messages_send session-worker-messages-consumer

# Ver latência de mensagens (via logs estruturados)
kubectl logs -l app=session-worker --tail=100 | \
  grep "message.send" | jq '.duration_seconds'

# Verificar se é problema de worker ou WhatsApp
# - Se pending alto: workers não estão processando rápido
# - Se pending baixo mas delivery demorado: WhatsApp Web delay
```

**Causas comuns**:
1. Worker lento (browser automation overhead)
2. WhatsApp Web rate limiting
3. Sessões desconectadas (worker tentando reconectar)
4. Pool de conexões DB saturado no worker

**Ações**:
```bash
# Escalar workers horizontalmente
kubectl scale deployment session-worker --replicas=3

# Verificar status das sessões
psql $DATABASE_URL -c "SELECT status, count(*) FROM sessions GROUP BY status;"

# Verificar logs de WhatsApp errors
kubectl logs -l app=session-worker --tail=200 | grep -i "whatsapp\|rate limit\|timeout"
```

### 2.5 Slow Consumer Detectado

**Sintoma**: NATS reporta slow consumer no monitoramento.

**Diagnóstico**:
```bash
# Ver metricas do servidor
nats server report connections

# Verificar qual consumer está lento
# Procurar por "slow consumer" nos logs do NATS
docker logs nats --tail 200 | grep -i slow
```

**Causas comuns**:
1. Processamento muito lento (browser automation)
2. Concurrency muito baixa
3. Recursos insuficientes (CPU/memoria)

**Ações**:
- Aumentar `WORKER_CONCURRENCY` no worker afetado
- Verificar recursos do container/pod
- Considerar escalar horizontalmente (mais replicas)

### 2.6 Conexão NATS Perdida

**Sintoma**: Workers não conseguem conectar ao NATS.

**Diagnóstico**:
```bash
# Verificar se NATS está rodando
kubectl get pods -l app=nats
# ou docker
docker ps | grep nats

# Testar conectividade
nc -zv nats-service 4222
# ou IP direto
nc -zv <nats-ip> 4222

# Verificar logs do NATS
kubectl logs -l app=nats --tail=100
# ou docker
docker logs nats --tail 100
```

**Ações**:
```bash
# Reiniciar NATS
kubectl rollout restart statefulset/nats
# ou docker
docker restart nats

# Reiniciar workers (reconectarão automaticamente)
kubectl rollout restart deployment/session-worker
kubectl rollout restart deployment/turbo-public-api
kubectl rollout restart deployment/turbo-orchestrator
```

---

## 3. Operações de Manutenção

### 3.1 Purgar Mensagens Antigas

```bash
# Purgar mensagens mais antigas que X horas de um stream
nats stream purge turbo_sessions_create --keep 1000

# Purgar stream inteiro (CUIDADO!)
nats stream purge turbo_sessions_create --force
```

### 3.2 Recriar Consumer

Se um consumer estiver corrompido ou com configuração errada:

```bash
# Deletar consumer existente
nats consumer delete turbo_sessions_create public-api-sessions-consumer

# Recriar via script de inicializacao
./scripts/nats/init_consumers.sh
```

### 3.3 Forcar Redelivery

Para forcar reprocessamento de mensagens pendentes:

```bash
# NAK todas as mensagens pendentes de um consumer
# Isso forca redelivery imediata
nats consumer next turbo_messages_send session-worker-messages-consumer --count 100 --nak
```

### 3.4 Verificar Duplicatas

```bash
# Ver configuração de deduplicação do stream
nats stream info turbo_sessions_create | grep -i dup

# Verificar se mensagem foi rejeitada como duplicata
docker logs nats --tail 500 | grep -i duplicate
```

---

## 4. Subjects e Streams do Turbo Notify

### 4.1 Sessions

| Subject | Descrição | Stream | Publisher | Consumer |
|---------|-----------|--------|-----------|----------|
| `turbo.sessions.create.requested` | Solicita criação de sessão | turbo_sessions_create | public-api | session-worker |
| `turbo.sessions.connected` | Sessão conectou com sucesso | turbo_sessions_events | session-worker | public-api, orchestrator |
| `turbo.sessions.disconnected` | Sessão desconectou | turbo_sessions_events | session-worker | public-api, orchestrator |
| `turbo.sessions.qr.generated` | QR code gerado | turbo_sessions_events | session-worker | public-api |
| `turbo.sessions.terminate.requested` | Solicita término de sessão | turbo_sessions_create | public-api | session-worker |
| `turbo.sessions.terminated` | Sessão terminada | turbo_sessions_events | session-worker | public-api, orchestrator |

### 4.2 Messages

| Subject | Descrição | Stream | Publisher | Consumer |
|---------|-----------|--------|-----------|----------|
| `turbo.messages.send.requested` | Solicita envio de mensagem | turbo_messages_send | public-api | session-worker |
| `turbo.messages.send.completed` | Mensagem enviada com sucesso | turbo_messages_events | session-worker | public-api |
| `turbo.messages.send.failed` | Falha no envio | turbo_messages_events | session-worker | public-api |
| `turbo.messages.received` | Mensagem recebida (incoming) | turbo_messages_events | session-worker | public-api |

### 4.3 Webhooks

| Subject | Descrição | Stream | Publisher | Consumer |
|---------|-----------|--------|-----------|----------|
| `turbo.webhooks.deliver.requested` | Solicita delivery de webhook | turbo_webhooks_deliver | public-api | webhook-worker* |
| `turbo.webhooks.delivered` | Webhook entregue | turbo_webhooks_events | webhook-worker* | public-api |
| `turbo.webhooks.failed` | Falha no delivery | turbo_webhooks_events | webhook-worker* | public-api |

*Módulo futuro

### 4.4 Dead Letter Queue

| Subject | Descrição | Stream | Publisher | Consumer |
|---------|-----------|--------|-----------|----------|
| `turbo.dlq.*` | Mensagens que falharam após max retries | turbo_dead_letter | orchestrator | manual (ops) |

---

## 5. Métricas Prometheus Disponíveis

### 5.1 Métricas do Servidor NATS

| Métrica | Descrição |
|---------|-----------|
| `gnatsd_varz_connections` | Conexões ativas |
| `gnatsd_varz_subscriptions` | Subscriptions ativas |
| `gnatsd_varz_slow_consumers` | Número de slow consumers |
| `gnatsd_varz_in_msgs` / `out_msgs` | Mensagens in/out (total) |
| `gnatsd_varz_jetstream_stats_api_errors` | Erros na API JetStream |
| `gnatsd_varz_jetstream_stats_storage` | Storage usado |

### 5.2 Métricas JetStream

| Métrica | Descrição |
|---------|-----------|
| `jetstream_stream_total_messages{stream_name}` | Mensagens por stream |
| `jetstream_consumer_num_ack_pending{consumer_name}` | Mensagens pendentes por consumer |
| `jetstream_consumer_delivered_consumer_seq` | Mensagens entregues |
| `jetstream_server_total_streams` | Total de streams |
| `jetstream_server_total_consumers` | Total de consumers |

### 5.3 Métricas Customizadas (Aplicação)

| Métrica | Descrição |
|---------|-----------|
| `nats_messages_processed_total{handler, subject, status, tenant_id}` | Mensagens processadas por tenant |
| `nats_message_processing_duration_seconds{handler, subject}` | Latência de processamento |
| `nats_dlq_messages_total{stream}` | Mensagens na DLQ |
| `session_worker_sessions_active{worker_id}` | Sessões ativas por worker |
| `message_send_duration_seconds{tenant_id, status}` | Latência de envio de mensagem end-to-end |

**Queries PromQL úteis**:

```promql
# Consumer lag por stream
jetstream_consumer_num_ack_pending{stream_name=~"turbo_.*"}

# Taxa de mensagens processadas por tenant
rate(nats_messages_processed_total{status="success"}[5m]) by (tenant_id)

# P95 de latência de processamento
histogram_quantile(
  0.95,
  sum(rate(nats_message_processing_duration_seconds_bucket[5m])) by (le, handler)
)

# Mensagens na DLQ (alerta se > 0)
jetstream_stream_total_messages{stream_name="turbo_dead_letter"}

# Slow consumers (alerta se > 0)
gnatsd_varz_slow_consumers
```

---

## 6. Alertas Configurados

| Alerta | Threshold | Descrição |
|--------|-----------|-----------|
| `NatsConsumerLag` | > 100 por 5min | Consumer lag alto |
| `NatsDLQAccumulating` | > 0 msgs | Mensagens na DLQ (alerta imediato) |
| `NatsSlowConsumer` | > 0 por 1min | Slow consumer detectado |
| `NatsJetStreamErrors` | > 10 em 15min | Erros na API JetStream |
| `NatsStorageHigh` | > 80% | Storage alto |
| `NatsNoConsumers` | < 1 por 2min | Nenhum consumer ativo |
| `SessionWorkerOffline` | 0 workers | Workers offline |
| `MessageDeliveryLag` | P95 > 30s | Latência de delivery alta |

**Exemplo de alerta Prometheus**:

```yaml
groups:
  - name: nats_alerts
    rules:
      - alert: NatsConsumerLag
        expr: jetstream_consumer_num_ack_pending > 100
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "NATS consumer {{ $labels.consumer_name }} has high lag"
          description: "Consumer {{ $labels.consumer_name }} has {{ $value }} pending messages"

      - alert: NatsDLQAccumulating
        expr: jetstream_stream_total_messages{stream_name="turbo_dead_letter"} > 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Messages accumulating in Dead Letter Queue"
          description: "DLQ has {{ $value }} messages - immediate investigation required"

      - alert: SessionWorkerOffline
        expr: up{job="session-worker"} == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "Session workers are offline"
          description: "No session workers available - sessions cannot be created"
```

---

## 7. Contatos e Escalação

1. **Nível 1**: Verificar alertas no Grafana, executar comandos básicos
2. **Nível 2**: Analisar logs, recriar consumers, reiniciar serviços
3. **Nível 3**: Escalar para desenvolvedor responsável

**Equipe de Plataforma:** #plataforma-alerts (Slack)
**Oncall:** @oncall-plataforma

---

## 8. Checklist de Verificação Rápida

- [ ] NATS está rodando? (`kubectl get pods -l app=nats` ou `docker ps | grep nats`)
- [ ] Session workers estão rodando? (`kubectl get pods -l app=session-worker`)
- [ ] Control Plane API está rodando? (`kubectl get pods -l app=public-api`)
- [ ] Conexões ativas? (`nats server ping`)
- [ ] Streams existem? (`nats stream list`)
- [ ] Consumers existem? (`nats consumer list <stream>`)
- [ ] Mensagens pendentes? (`nats consumer info <stream> <consumer>`)
- [ ] DLQ vazia? (`nats stream info turbo_dead_letter`)
- [ ] Sem slow consumers? (Grafana: NATS > Slow Consumers)
- [ ] Workers conectados? (`nats server report connections | grep worker`)

---

## 9. Troubleshooting Específico por Componente

### 9.1 Public API (Control Plane)

**Problema**: API não consegue publicar eventos

```bash
# Verificar logs
kubectl logs -l app=public-api --tail=100 | grep -i nats

# Verificar conexão
nats server report connections | grep public-api

# Ação: Reiniciar API
kubectl rollout restart deployment/turbo-public-api
```

### 9.2 Session Worker

**Problema**: Worker não consome mensagens de criação de sessão

```bash
# Verificar consumer
nats consumer info turbo_sessions_create session-worker-sessions-consumer

# Verificar logs
kubectl logs -l app=session-worker --tail=200

# Verificar capacidade (sessões ativas vs max)
# (implementar metrics endpoint)

# Ação: Escalar workers
kubectl scale deployment session-worker --replicas=3
```

**Problema**: Worker não publica eventos de estado de sessão

```bash
# Verificar logs de publish
kubectl logs -l app=session-worker --tail=100 | grep "turbo.sessions"

# Verificar se stream existe
nats stream info turbo_sessions_events

# Ação: Verificar permissões NATS, reiniciar worker
```

### 9.3 Orchestrator

**Problema**: Orchestrator não processa eventos de sessão

```bash
# Verificar consumer
nats consumer info turbo_sessions_events orchestrator-events-consumer

# Verificar logs
kubectl logs -l app=orchestrator --tail=100

# Ação: Reiniciar orchestrator
kubectl rollout restart deployment/turbo-orchestrator
```

---

## 10. Exemplos de Troubleshooting por Cenário

### Cenário 1: Sessão Não Conecta

**Sintomas**:
- API retorna session criada
- Session fica stuck em `PENDING` ou `CONNECTING`
- Worker não processa

**Diagnóstico**:
```bash
# 1. Verificar se mensagem foi publicada
nats stream view turbo_sessions_create --last 10

# 2. Verificar consumer
nats consumer info turbo_sessions_create session-worker-sessions-consumer

# 3. Verificar logs do worker
kubectl logs -l app=session-worker --tail=200 | grep <session_id>
```

**Causas possíveis**:
- Worker offline
- Consumer lag alto
- Worker sem capacidade (max sessions atingido)
- Erro no processamento (exception no worker)

### Cenário 2: Mensagem Não Enviada

**Sintomas**:
- API retorna sucesso
- Mensagem fica stuck em `PENDING`
- Nunca chega status `SENT` ou `FAILED`

**Diagnóstico**:
```bash
# 1. Verificar se mensagem foi publicada
nats stream view turbo_messages_send --last 10 | grep <message_id>

# 2. Verificar consumer
nats consumer info turbo_messages_send session-worker-messages-consumer

# 3. Verificar se sessão está ativa
psql $DATABASE_URL -c "SELECT status FROM sessions WHERE id='<session_id>';"

# 4. Verificar logs do worker
kubectl logs -l app=session-worker --tail=200 | grep <message_id>
```

**Causas possíveis**:
- Sessão desconectada
- Worker não processando (lag)
- WhatsApp Web rate limiting
- Erro de validação (número inválido, etc)

### Cenário 3: DLQ Acumulando

**Sintomas**:
- Alerta `NatsDLQAccumulating` dispara
- Mensagens específicas sempre falham

**Diagnóstico**:
```bash
# 1. Ver mensagens na DLQ
nats stream view turbo_dead_letter --last 20

# 2. Analisar padrão de falhas
# - Mesmo tenant?
# - Mesmo subject?
# - Mesmo tipo de payload?

# 3. Verificar logs do consumer que falhou
# (extrair consumer name do DLQ payload)
```

**Ações**:
- Se erro de código: fix + deploy
- Se erro de dados: corrigir dados, republicar
- Se erro de configuração: ajustar config, republicar

---

*Última atualização: 2026-03-13*
