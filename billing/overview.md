# Sistema de Billing - Visão Geral

## Introdução

O sistema de billing do Turbo Notify gerencia subscriptions, planos, cobranças e pagamentos para o SaaS de infraestrutura WhatsApp Web.

## Arquitetura

```
dashboard-web → billing-api (HTTP)     # Telas de billing
              ↓
         PostgreSQL

dashboard-api → shared-billing         # Verificação de plano
public-api → shared-billing            # Verificação de plano
              ↓
         PostgreSQL (acesso direto)
```

**Componentes**:
- **billing-api**: Microserviço BFF para telas de billing (FastAPI + Python 3.13)
- **shared-billing**: Biblioteca compartilhada para verificação de planos (repositórios PostgreSQL diretos)

---

## Planos Disponíveis

### 1. FREE Plan

**Preço**: BRL 0/mês (Gratuito)

**Limites**:
- Messages per month: 50
- Messages per minute: 5
- Recipients per day: 2
- Advanced features: ❌ Não disponível
- Webhooks: ❌ Não disponível
- Extra numbers: 0 (não disponível)

**CTA**: Ativação direta (signup)

### 2. SOLO Plan

**Preço**: BRL 150/mês

**Limites**:
- Messages per month: 200
- Messages per minute: 20
- Recipients per day: 10
- Advanced features: ✅ Disponível
- Webhooks: ✅ Disponível
- Extra numbers: 0 (não disponível)

**CTA**: Ativação direta (signup)

### 3. BUSINESS Plan (Destaque)

**Preço**: BRL 350/mês

**Limites**:
- Messages per month: Unlimited
- Messages per minute: 200
- Recipients per day: Unlimited
- Advanced features: ✅ Disponível
- Webhooks: ✅ Disponível
- Extra numbers: 300 (BRL 150 cada)

**CTA**: Ativação direta (signup)

**Destaque**: Plano recomendado para empresas

### 4. CUSTOM Plan

**Preço**: Custom (negociável)

**Limites**:
- Messages per month: Unlimited
- Messages per minute: 1,000 (high throughput)
- Recipients per day: Unlimited
- Advanced features: ✅ Disponível
- Webhooks: ✅ Disponível
- Extra numbers: Unlimited

**CTA**: Contato com vendas

**Uso**: Empresas de grande porte com necessidades específicas

---

## Features e Limites

### Tabela de Features

| Feature Key | FREE | SOLO | BUSINESS | CUSTOM |
|-------------|------|------|----------|--------|
| `messages_per_month` | 50 | 200 | Unlimited | Unlimited |
| `messages_per_minute` | 5 | 20 | 200 | 1,000 |
| `recipients_per_day` | 2 | 10 | Unlimited | Unlimited |
| `advanced_features` | No | Yes | Yes | Yes |
| `webhook` | No | Yes | Yes | Yes |
| `extra_numbers` | 0 | 0 | 300 | Unlimited |

### Validity Periods

- `messages_per_month`: Renovado mensalmente
- `messages_per_minute`: Limite por minuto (rate limiting)
- `recipients_per_day`: Renovado diariamente (00:00 UTC)
- `extra_numbers`: Quantidade máxima simultânea

---

## Conceitos Principais

### Subscription

Uma **subscription** representa a assinatura de um tenant a um plano.

**Propriedades**:
- `id`: Identificador único (sub_xxxxx)
- `tenant_id`: Tenant associado
- `plan_id`: Referência ao plano (tn_pricing.plan)
- `plan_tier`: FREE, SOLO, BUSINESS, CUSTOM
- `status`: Estado atual (ACTIVE, PAST_DUE, CANCELED, TRIALING)
- `current_period_start`: Início do ciclo de faturamento
- `current_period_end`: Fim do ciclo de faturamento
- `payment_method_id`: Método de pagamento padrão

**Ciclo de vida**: Ver [subscription-lifecycle.md](./subscription-lifecycle.md)

### Invoice

Uma **invoice** é uma fatura gerada ao final de um billing cycle.

**Propriedades**:
- `id`: Identificador único
- `tenant_id`: Tenant associado
- `billing_cycle_id`: Ciclo de faturamento
- `subscription_id`: Subscription associada
- `amount_cents`: Valor total em centavos
- `status`: draft, open, paid, void, uncollectible
- `due_date`: Data de vencimento
- `paid_at`: Quando foi paga

**Composição**:
- Base amount (plano base)
- Extra numbers charges (números extras ativos)
- Usage charges (uso excedente)

### Payment

Um **payment** representa uma tentativa de pagamento de uma invoice.

**Propriedades**:
- `id`: Identificador único
- `invoice_id`: Invoice sendo paga
- `amount_cents`: Valor
- `status`: pending, processing, succeeded, failed
- `payment_method_id`: Método de pagamento usado
- `stripe_payment_intent_id`: Referência Stripe

**Fluxo**:
1. Invoice gerada → Payment criado (status=pending)
2. Payment Intent criado no Stripe
3. Webhook recebido → Payment atualizado (succeeded/failed)
4. Se succeeded → Invoice marcada como paid
5. Se failed → Subscription marcada como PAST_DUE

### Payment Method

Um **payment method** é um método de pagamento cadastrado (cartão de crédito, boleto).

**Propriedades**:
- `id`: Identificador único
- `tenant_id`: Tenant associado
- `type`: card, boleto
- `card_last4`: Últimos 4 dígitos (para cards)
- `card_brand`: Visa, Mastercard, etc.
- `is_default`: Método padrão

---

## Multimercado

O sistema suporta múltiplas moedas e países via tabela `pricing_rules`.

### Pricing Rules

**Tabela**: `pricing_rules`

Contém pricing específico por:
- `plan_id`: Qual plano
- `country_code`: País (BR, US, etc.)
- `currency_code`: Moeda (BRL, USD, etc.)
- `base_amount_cents`: Preço base do plano
- `extra_number_amount_cents`: Preço de cada número extra
- `recurrence`: monthly, annual

**Seleção automática**: Ao criar subscription, sistema seleciona pricing_rule baseado em country_code e currency_code do tenant.

**Exemplo**:

| Plan | Country | Currency | Base Amount | Extra Number |
|------|---------|----------|-------------|--------------|
| SOLO | BR | BRL | 15,000 centavos (R$ 150) | - |
| SOLO | US | USD | 2,900 centavos ($29) | - |
| BUSINESS | BR | BRL | 35,000 centavos (R$ 350) | 15,000 centavos (R$ 150) |

Ver mais: [pricing-rules.md](./pricing-rules.md)

---

## Estados de Subscription

### ACTIVE

- **Descrição**: Subscription ativa e paga
- **Pode usar features**: ✅ Sim
- **Ações permitidas**: Cancel, Upgrade, Downgrade

### TRIALING

- **Descrição**: Período de trial gratuito
- **Pode usar features**: ✅ Sim (trial)
- **Ações permitidas**: Cancel, Upgrade
- **Duração**: Configurável (padrão: 14 dias)

### PAST_DUE

- **Descrição**: Pagamento atrasado
- **Pode usar features**: ❌ Não (bloqueado)
- **Ações permitidas**: Atualizar método de pagamento, Reativar
- **Auto-transição**: Após 3 falhas de pagamento → CANCELED

### CANCELED

- **Descrição**: Subscription cancelada pelo tenant
- **Pode usar features**: ❌ Não
- **Ações permitidas**: Reativar (nova subscription)
- **Comportamento**: Permanece ativa até `current_period_end`

Ver diagrama completo: [subscription-lifecycle.md](./subscription-lifecycle.md)

---

## Gateway de Pagamento

O sistema abstrai o gateway de pagamento via interface `PaymentGateway`.

**Implementações**:
- `MockPaymentGateway`: Mock para desenvolvimento e testes
- `StripePaymentGateway`: Integração com Stripe (produção)

**Troca de gateway**: Variável de ambiente `PAYMENT_GATEWAY_TYPE=stripe|mock`

**Operações abstraídas**:
- create_customer
- create_subscription
- cancel_subscription
- create_payment_intent
- check_payment_status

Ver mais: [payment-gateway-abstraction.md](./payment-gateway-abstraction.md)

---

## Eventos NATS

### Publicados por billing-api

| Subject | Quando | Payload |
|---------|--------|---------|
| `turbo.billing.subscription.created` | Subscription criada | {tenant_id, subscription_id, plan_tier, status} |
| `turbo.billing.subscription.canceled` | Subscription cancelada | {tenant_id, subscription_id, canceled_at} |
| `turbo.billing.usage.recorded` | Uso registrado (ex: extra number ativado) | {tenant_id, subscription_id, metric_key, quantity} |
| `turbo.billing.payment.succeeded` | Pagamento bem-sucedido | {tenant_id, payment_id, invoice_id, amount_cents} |
| `turbo.billing.payment.failed` | Falha no pagamento | {tenant_id, payment_id, invoice_id, failure_code} |

### Consumidos por billing-api

| Subject | Ação |
|---------|------|
| `turbo.extra_numbers.activated` | Gerar cobrança pro-rata para número extra |
| `turbo.extra_numbers.removed` | Registrar remoção (sem estorno) |

---

## Próximos Passos

Para implementar billing no seu projeto:

1. **Escolher plano**: [Landing page com pricing](https://turbo-notify.com/pricing)
2. **Criar subscription**: POST `/subscriptions` (billing-api)
3. **Adicionar payment method**: POST `/payment-methods`
4. **Usar features**: Verificações via shared-billing nos outros módulos
5. **Receber faturas**: Geradas automaticamente no fim do ciclo
6. **Pagar**: Via Stripe Payment Intent

---

## Relacionado

- [Extra Numbers Billing](./extra-numbers-billing.md)
- [Subscription Lifecycle](./subscription-lifecycle.md)
- [Pricing Rules](./pricing-rules.md)
- [Payment Gateway Abstraction](./payment-gateway-abstraction.md)
- [Architecture: Billing API Integration](../architecture/billing-api-integration.md)
