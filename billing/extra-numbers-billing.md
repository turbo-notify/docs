# Cobrança de Números Extras

## Introdução

Números extras são números de telefone adicionais que um tenant pode cadastrar para enviar mensagens WhatsApp além do número principal.

**Disponível em**: Plano BUSINESS e CUSTOM

---

## Modelo de Cobrança

### Cobrança Pro-Rata Diária

Cada número extra é cobrado proporcionalmente aos dias restantes no ciclo de faturamento.

**Fórmula**:
```
amount_prorated = (monthly_rate ÷ 30) × days_remaining
```

**Onde**:
- `monthly_rate`: Taxa mensal do número extra (BRL 150 para BUSINESS)
- `days_remaining`: Dias restantes até o fim do `current_period_end`

### Exemplo de Cálculo

**Cenário**:
- Plano: BUSINESS
- Taxa mensal de número extra: BRL 150
- Billing cycle: 01/jan a 30/jan (30 dias)
- Número extra ativado em: 11/jan
- Dias restantes: 20 (de 11/jan a 30/jan)

**Cálculo**:
```
Valor pro-rata = (BRL 150 ÷ 30) × 20
               = BRL 5,00 × 20
               = BRL 100,00
```

**Resultado**: Tenant é cobrado BRL 100 no momento da ativação.

---

## Pricing por Plano

### BUSINESS Plan

- **Limite de números extras**: 300
- **Taxa mensal por número**: BRL 150
- **Cobrança**: Pro-rata no momento da ativação

### CUSTOM Plan

- **Limite de números extras**: Unlimited
- **Taxa mensal por número**: Negociável (custom pricing)
- **Cobrança**: Pro-rata no momento da ativação

### FREE e SOLO Plans

- **Números extras**: ❌ Não disponível
- **Erro ao tentar adicionar**: `402 not_available_in_plan`

---

## Ciclo de Faturamento

### Pagamento Antecipado

O pagamento de números extras é **antecipado** (prepaid):

1. Tenant ativa número extra no dia 11
2. Sistema calcula pro-rata (BRL 100)
3. **Pagamento ocorre imediatamente** (mesmo dia)
4. Número fica ativo até o fim do billing cycle (dia 30)

### Renovação Mensal

Ao início de cada novo billing cycle:

1. Sistema lista números extras ativos
2. Calcula taxa integral (BRL 150 × quantidade)
3. Adiciona à invoice do mês
4. Tenant paga junto com subscription base

---

## Remoção de Números Extras

### Sem Estorno

**Regra crítica**: Não há estorno em remoção antecipada.

**Cenário**:
- Número ativado: 11/jan (pagou BRL 100 pro-rata)
- Número removido: 15/jan (antes do fim do ciclo)
- Estorno: **BRL 0** (nenhum)
- Número permanece ativo até: 30/jan (fim do ciclo)

### Comportamento

Quando tenant remove um número extra:

1. Número **continua funcionando** até `current_period_end`
2. **Não há estorno** do valor pago
3. No próximo billing cycle, número **não** é incluído na fatura
4. Cobrança encerra automaticamente

### Desativação vs Remoção

**Desativar**:
- Suspende uso imediato do número
- **Continua cobrando** até fim do ciclo
- Útil para: bloquear número temporariamente sem perder

**Remover**:
- Marca número para não renovar
- Continua funcionando até fim do ciclo pago
- **Encerra cobrança** no próximo ciclo

---

## Limites por Plano

### Verificação de Limite

Antes de adicionar número extra, sistema verifica:

```python
current_extra_numbers_count = count_active_extra_numbers(tenant_id)
max_allowed = subscription.plan_tier.max_extra_numbers

if current_extra_numbers_count >= max_allowed:
    raise PlanTierInsufficientError("Limite de números extras atingido")
```

### BUSINESS Plan

- **Max extra numbers**: 300
- **Tentativa de adicionar o 301º**: Erro `402 plan_tier_insufficient`

### CUSTOM Plan

- **Max extra numbers**: `999999` (ilimitado na prática)
- **Negociável**: Custom tiers podem ter limites custom

---

## Tabela de Usage Records

Cada ativação de número extra gera um `UsageRecord`:

**Campos**:
- `id`: Identificador único
- `tenant_id`: Tenant
- `subscription_id`: Subscription associada
- `metric_key`: `extra_number_prorated`
- `quantity`: 1 (um número)
- `period_start`: Data de ativação
- `period_end`: Fim do billing cycle
- `metadata`: `{phone_number_id, alias, number, prorated_amount_cents, days_remaining}`

**Uso**:
- Auditoria de cobranças
- Geração de invoices (agregar todos usage_records do período)
- Relatórios de uso

---

## Fluxo Completo

### 1. Ativação de Número Extra

```
[public-api] POST /extra-numbers
    ↓
Verificar plano (via shared-billing): require_plan("BUSINESS")
Verificar limite: count < max_extra_numbers
Criar ExtraNumber entity
Publicar NATS: turbo.extra_numbers.activated
    ↓
[billing-api] NATS consumer
    ↓
AddExtraNumberChargeUseCase.execute()
    ├─ Buscar subscription ativa
    ├─ Calcular pro-rata (ProrationService)
    ├─ Criar UsageRecord com metadata
    ├─ Salvar em database
    └─ Publicar NATS: turbo.billing.usage.recorded
    ↓
[Response] {usage_record_id, prorated_amount_cents, days_remaining}
```

### 2. Remoção de Número Extra

```
[public-api] DELETE /extra-numbers/{alias}
    ↓
Buscar ExtraNumber entity
Marcar como deleted (soft delete)
Publicar NATS: turbo.extra_numbers.removed
    ↓
[billing-api] NATS consumer
    ↓
Registrar remoção (sem estorno)
Número continua ativo até period_end
    ↓
[Response] 204 No Content
```

### 3. Geração de Invoice Mensal

```
[billing-api] Scheduled job (fim do billing cycle)
    ↓
GenerateInvoiceUseCase.execute()
    ├─ Buscar subscription
    ├─ Calcular base_amount (plano base)
    ├─ Buscar UsageRecords do período (extra_number_prorated)
    ├─ Somar extra_numbers_amount
    ├─ Calcular total = base + extra_numbers
    ├─ Criar Invoice entity
    └─ Publicar NATS: turbo.billing.invoice.generated
    ↓
[Response] Invoice gerada
```

---

## Multimercado

### Pricing Rules por País

A taxa de número extra varia por país:

| Country | Currency | Extra Number Rate |
|---------|----------|-------------------|
| BR | BRL | BRL 150/mês (15,000 centavos) |
| US | USD | $29/mês (2,900 centavos) |
| EU | EUR | €25/mês (2,500 centavos) |

**Seleção automática**: Sistema usa pricing_rule baseado em `country_code` do tenant.

---

## Exemplos Práticos

### Exemplo 1: Ativação no início do ciclo

- **Billing cycle**: 01/mar a 31/mar (31 dias)
- **Ativação**: 01/mar (primeiro dia)
- **Days remaining**: 31
- **Cálculo**: (150 ÷ 30) × 31 = BRL 155
- **Observação**: Cobrado ligeiramente mais que taxa mensal (mês tem 31 dias)

### Exemplo 2: Ativação no meio do ciclo

- **Billing cycle**: 01/mar a 31/mar
- **Ativação**: 16/mar (meio do mês)
- **Days remaining**: 16 (de 16/mar a 31/mar)
- **Cálculo**: (150 ÷ 30) × 16 = BRL 80

### Exemplo 3: Múltiplos números no mesmo dia

- **Números ativados**: 3 números no mesmo dia
- **Days remaining**: 20
- **Cálculo por número**: (150 ÷ 30) × 20 = BRL 100
- **Total**: BRL 100 × 3 = BRL 300

---

## Auditoria e Compliance

### Rastreabilidade

Cada cobrança de número extra é rastreável via:

1. **UsageRecord**: Registro completo com metadata
2. **NATS events**: `turbo.billing.usage.recorded`
3. **Invoice line items**: Detalhamento na fatura

### Metadata Armazenado

```json
{
  "phone_number_id": "extra_abc123",
  "alias": "vendas-sp",
  "number": "+5511999998888",
  "prorated_amount_cents": 10000,
  "days_remaining": 20
}
```

---

## Perguntas Frequentes

### Por que não há estorno?

**Resposta**: Números extras são prepaid. Uma vez pago, o tenant tem direito a usar até o fim do período. Similar a planos mensais de celular.

### Posso pausar um número sem remover?

**Resposta**: Sim, use desativação (deactivate). Número para de funcionar mas continua na sua conta e sendo cobrado.

### O que acontece se eu exceder o limite?

**Resposta**: API retorna erro `402 plan_tier_insufficient`. É necessário fazer upgrade para CUSTOM plan ou remover números inativos.

### Como vejo histórico de cobranças?

**Resposta**: Via dashboard-web → Billing → Invoices, cada invoice lista os números extras cobrados no período.

---

## Relacionado

- [Billing Overview](./overview.md)
- [Subscription Lifecycle](./subscription-lifecycle.md)
- [Pricing Rules](./pricing-rules.md)
- [API: Extra Numbers](../../public-api/docs/api/public/extra-numbers.md)
