# API Versioning Strategy

## Overview

O Turbo Notify utiliza diferentes estratégias de versionamento para APIs internas (BFF) e APIs públicas, adequadas ao caso de uso de cada uma.

---

## BFF APIs (Backend for Frontend) - **SEM** versionamento

### Dashboard API (porta 8001)

**Endpoints**: `/bff/*`

**Estratégia**: Sem versionamento de URL

**Justificativa**:
- ✅ BFF evolui em conjunto com o frontend (dashboard-web)
- ✅ Deploy coordenado (frontend + backend sincronizados)
- ✅ Não há clientes externos para manter compatibilidade
- ✅ Versionamento seria overhead desnecessário
- ✅ Mudanças podem ser feitas livremente

**Exemplos**:
```
POST   /bff/auth/login
GET    /bff/access-keys
POST   /bff/tenants
```

**Breaking Changes**: Permitidos (coordenados com deploy do frontend)

---

## Public APIs - **COM** versionamento `/v1`

### Control Plane API (porta 8000)

**Endpoints**: `/v1/*`

**Estratégia**: Versionamento semântico de URL com prefixo `/v1`

**Justificativa**:
- ✅ Clientes externos podem estar em produção
- ✅ Mudanças precisam ser retrocompatíveis
- ✅ Permite evolução sem quebrar integrações existentes
- ✅ Padrão de mercado (Stripe, Twilio, GitHub, etc.)
- ✅ Contratos de API são públicos e devem ser estáveis

**Exemplos**:
```
POST   /v1/messages
GET    /v1/messages/{message_id}/status
POST   /v1/extra-numbers
GET    /v1/extra-numbers
DELETE /v1/extra-numbers/{alias}
POST   /v1/reactions
POST   /v1/typing-indicator
```

**Breaking Changes**: Requerem nova versão (`/v2`, `/v3`, etc.)

---

## Implementação

### public-api (main.py)

```python
# Public API endpoints with /v1 versioning
app.include_router(messages_router, prefix="/v1")
app.include_router(extra_numbers_router, prefix="/v1")
app.include_router(reactions_router, prefix="/v1")
app.include_router(typing_indicator_router, prefix="/v1")
```

### dashboard-api (main.py)

```python
# BFF endpoints without versioning
app.include_router(auth_router, prefix="/bff")
app.include_router(access_keys_router, prefix="/bff")
app.include_router(tenants_router, prefix="/bff")
```

---

## Quando Criar Nova Versão

### `/v2` é necessário quando:

1. **Mudanças Incompatíveis no Request**:
   - Remover campo obrigatório
   - Alterar tipo de campo (string → int)
   - Alterar validação (tornar campo obrigatório)

2. **Mudanças Incompatíveis no Response**:
   - Remover campo do response
   - Alterar formato de data/timestamp
   - Alterar estrutura de objetos aninhados

3. **Mudanças de Comportamento**:
   - Alterar semântica de endpoint
   - Alterar regras de negócio de forma significativa

### Mudanças Retrocompatíveis (podem permanecer em `/v1`):

- ✅ Adicionar novo campo opcional no request
- ✅ Adicionar novo campo no response
- ✅ Adicionar novo endpoint
- ✅ Adicionar novo enum value (se código cliente for defensivo)
- ✅ Melhorar performance
- ✅ Corrigir bugs

---

## Ciclo de Vida de Versões

### Fase 1: `/v1` - Atual
- Status: **Ativo**
- Manutenção: Completa
- Depreciação: N/A

### Fase 2: `/v2` - Quando necessário
- Status: **Futuro**
- Coexistência: `/v1` e `/v2` rodarão em paralelo
- Transição: 6 meses de suporte a ambas

### Fase 3: `/v1` - Deprecated
- Status: **Deprecated** (após lançamento de `/v2`)
- Manutenção: Apenas correção de bugs críticos
- Sunset: 6 meses após depreciação

---

## Health e Documentação

**Endpoints sem versionamento** (acessíveis diretamente):
```
GET  /health          # Health check
GET  /health/ready    # Readiness check
GET  /health/live     # Liveness check
GET  /docs            # Swagger UI (dev only)
GET  /redoc           # ReDoc UI (dev only)
GET  /openapi.json    # OpenAPI spec
```

---

## Migração de Clientes

Quando `/v2` for lançado:

1. **Comunicação**: 3 meses de antecedência via email + changelog
2. **Depreciação**: Header `X-API-Deprecation` em responses de `/v1`
3. **Sunset**: 6 meses após `/v2` estar estável
4. **Descontinuação**: `/v1` retorna 410 Gone

**Exemplo de Header de Depreciação**:
```
X-API-Deprecation: version=v1 sunset=2027-01-01T00:00:00Z link=https://docs.turbonotify.com/migration/v1-to-v2
```

---

## Testes

### Verificar Versionamento

```bash
# ✅ Endpoints versionados devem funcionar
curl http://localhost:8000/v1/extra-numbers

# ✅ Endpoints sem versão devem retornar 404
curl http://localhost:8000/extra-numbers
# Expected: {"detail":"Not Found"}

# ✅ OpenAPI deve mostrar paths com /v1
curl http://localhost:8000/openapi.json | jq '.paths | keys'
# Expected: ["/health", "/v1/messages", "/v1/extra-numbers", ...]
```

---

## Referências

- [Stripe API Versioning](https://stripe.com/docs/api/versioning)
- [Twilio API Versioning](https://www.twilio.com/docs/usage/api#api-versioning)
- [GitHub API Versioning](https://docs.github.com/en/rest/about-the-rest-api/api-versions)
- [Best Practices for Versioning REST APIs](https://www.freecodecamp.org/news/rest-api-best-practices-rest-endpoint-design-examples/)
