---
name: revtriever-motor
description: Integrar com a API do Motor de Cobrança Revtriever (motor.revtriever.com) — produtos, planos, clientes, assinaturas, faturas, eventos. Use ao escrever código que chama a API Revtriever, montar cobrança recorrente (pix, boleto, cartão), ou tirar dúvida sobre o contrato, autenticação, idempotência ou erros do motor.
---

# Motor de Cobrança Revtriever — integração

API REST de cobrança por assinatura. Base: `https://motor.revtriever.com/v1` · Auth: `Authorization: Bearer rk_…` (API key). O mesmo backend é acessível por MCP em `https://motor.revtriever.com/mcp` (mesma key, mesmas validações).

**Referência profunda**: https://docs.revtriever.com/llms-full.txt (todo o site, formato para agentes) · OpenAPI: https://motor.revtriever.com/docs-json · Site: https://docs.revtriever.com

## Convenções que quebram integração se ignoradas

1. **Dinheiro é sempre centavos, inteiro** (`24990` = R$ 249,90). Nunca float, nunca string.
2. **`Idempotency-Key` é obrigatório** em `POST /products`, `POST /plans` e `POST /subscriptions` — sem o header, `400`. Repetir a chave devolve a mesma resposta: use uma chave sua determinística (ex.: id do pedido) e re-tente sem medo.
3. **Erros são problem+json**: `{ type, title, status, detail, requestId, meta? }`. Faça branch no `type` (código estável, ex.: `engine.gateway_choice_required`), **nunca** no texto do `detail`. Guarde o `requestId` para suporte.
4. **Listas**: `?page=1&pageSize=25` (máx 100), envelope `{ data, pagination }`. Exceção: `/v1/events` pagina por **cursor** (`?cursor=<id do último evento>`, mais antigo primeiro).
5. **Rate limit**: 300 req/min por conta (REST + MCP somados). `429` com `Retry-After` — respeite o header.
6. **Datas** `YYYY-MM-DD` em `America/Sao_Paulo`; instantes ISO 8601 UTC. `externalRef` = o **seu** id (cliente, produto) — é ele que volta em webhooks. Para achar o id do motor a partir do seu: `GET /v1/customers?externalRef=…`.

## Modelo de dados (o essencial)

`Produto` (catálogo, preço **mensal**: fixo ou variável via endpoint seu) → `Plano` (oferta versionada: `periodMonths` 1/3/6/12, método padrão, `maxInstallments` 1–12 só cartão, `expirationDays` 0–90, régua de descontos por faixa de ciclo — **primeira regra que casa ganha, nada empilha; ciclo = fatura**) → `Cliente` (nasce inline na assinatura, chave = `externalRef`; o motor é a fonte do nome/e-mail/CPF-CNPJ/telefone/endereço e espelha nos gateways) → `Assinatura` (vínculo cliente↔plano, **pinada na versão** do plano da adesão; `chargeDay` 1–28; gateway congelado na criação) → `Fatura` (gerada pelo motor no próprio `chargeDay`, janela 05h–22h BRT; nunca criada por você).

Fluxo da fatura: `draft → awaiting_value → ready → issued → paid | overdue` (+ `canceled`). `dueDate` = data do ciclo; `paymentDueDate` = vencimento real (`dueDate + expirationDays`, ou o que o gateway confirmar). Fatura com item variável sem resposta fica **retida** em `awaiting_value` — monitore `GET /v1/invoices?status=awaiting_value`.

## Fluxo mínimo (a ordem importa)

1. `POST /v1/products` — produto com `pricing: { mode: "fixed", monthlyPriceCents }` (ou `mode: "endpoint"` — ver skill `revtriever-pricing-endpoint`).
2. `POST /v1/plans` — a oferta completa num POST (itens + réguas).
3. `POST /v1/subscriptions` — cliente inline (`customer.externalRef` = seu id; reaproveitado se já existe). Com **mais de um gateway** conectado, `gatewayConnectionId` é obrigatório (`422 engine.gateway_choice_required`; opções em `GET /v1/integration`).
4. `GET /v1/subscriptions/{id}/preview` — dry-run da próxima fatura, **zero efeitos**. Sempre confira aqui antes do primeiro ciclo.

## Pegadinhas confirmadas

- Editar plano é `PUT /v1/plans/{id}` com a **oferta completa** (não é patch) e cria **versão nova** — assinantes atuais ficam pinados. Para mudar um assinante só: `POST /v1/subscriptions/{id}/customize`.
- `pricingMode` do produto é **imutável** (`409` se tentar trocar).
- **Cliente se edita em `PATCH /v1/customers/{id}`** (nome, e-mail, `document`, `phone`, `address`; `null` apaga; `externalRef` é imutável) — **não** repita `POST /v1/subscriptions` para corrigir um dado: isso cria outra assinatura. A edição vale para as próximas faturas/notas e é replicada ao cliente espelhado no gateway que aceita atualização (assíncrono; evento `customer.updated`).
- Pix e boleto são sempre à vista; parcelamento só no cartão.
- `chargeDay` aceito: 1–28. Criou dia 29/30/31 sem informar → vira 28.
- **Duas credenciais distintas**: API key `rk_…` (você → motor) ≠ secret de integração (motor → você, assina webhooks e o pull de preço). Nunca use uma no lugar da outra.
- pagar.me exige `customer.phone` para pix e `customer.address` para boleto — inclua na criação da assinatura se o gateway for pagar.me.

## Erros que você vai encontrar

`400` payload/Idempotency-Key · `401` API key · `404` não existe (ou não é seu) · `409` conflito de estado (modo de preço, plano arquivado, já cancelada) · `422` regra de negócio (método/parcelamento não suportado, desconto em produto variável, escolha de gateway) · `429` rate limit.
