---
name: revtriever-pricing-endpoint
description: Implementar o endpoint de preço variável do Motor Revtriever — o webhook que o motor chama para perguntar quanto cobrar de um produto de preço variável (uso, consumo). Use ao criar produto com pricing mode "endpoint", implementar/depurar o endpoint de preço, ou verificar a assinatura HMAC X-Revtriever-Signature.
---

# Endpoint de preço variável — Revtriever

Produto com `pricing.mode: "endpoint"`: na geração da fatura, o motor faz `POST` na sua URL e você responde o valor. Guia completo: https://docs.revtriever.com/guias/endpoint-de-preco/

## O contrato

**Request do motor** (header `X-Revtriever-Signature: t=<unix>,v1=<hmac>`):

```json
{
  "version": 1,
  "mode": "live",
  "pricingRequestId": "prq_…",
  "product": { "externalRef": "seu-id-do-produto" },
  "subscription": { "externalRef": "…" },
  "customer": { "externalRef": "seu-id-do-cliente" },
  "period": { "start": "2026-08-01", "end": "2026-08-31" },
  "cycle": 3,
  "currency": "BRL"
}
```

**Sua resposta** — HTTP 200 em até **10 segundos**:

```json
{ "amount": 18760, "description": "Consumo de agosto" }
```

- `amount` em **centavos, inteiro**. `description` opcional (vira a linha da fatura).
- Para **não cobrar neste ciclo**: `{ "skip": true }`. Isso é diferente de `amount: 0` (que cobra zero e liquida localmente).
- Resposta acima do `capCents` do produto = tratada como **falha** (`over_cap`) — o cap é a proteção contra um bug seu virar cobrança absurda.

## Verificação HMAC — os dois erros clássicos

Assinatura: `v1 = HMAC-SHA256(secret, t + "." + corpo_cru)`. Rejeite `t` com mais de 5 minutos.

1. **Verifique sobre o corpo CRU** (bytes recebidos), nunca sobre o JSON re-serializado — a ordem das chaves muda e a assinatura quebra.
2. **Compare com `timingSafeEqual`**, nunca `===` (e cheque o comprimento antes, senão o `timingSafeEqual` lança).

```js
const crypto = require("node:crypto");
function verify(rawBody, signatureHeader, secret) {
  const { t, v1 } = Object.fromEntries(signatureHeader.split(",").map((p) => p.split("=")));
  if (Math.abs(Date.now() / 1000 - Number(t)) > 300) return false;
  const expected = crypto.createHmac("sha256", secret).update(`${t}.${rawBody}`).digest("hex");
  const a = Buffer.from(expected), b = Buffer.from(v1 ?? "");
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

O secret é **um por conta** (o mesmo dos webhooks) — `GET /v1/integration` mostra os últimos 4; rotação via `POST /v1/integration/secret/rotate` com 24h de graça para o anterior.

## Regras de operação

- O motor pergunta **uma vez por (item, ciclo)** e congela request + response crus na fatura — seu endpoint não precisa ser idempotente.
- A pergunta vem **com a janela do ciclo já fechada** (`period.end` = hora 0 do `chargeDay`) — o valor cobre o período completo.
- Falhou (timeout 10s, HTTP ≠ 200, formato inválido, over cap)? Backoff automático por **12h+**, aviso rápido (~15 min, evento `pricing.failing` com o erro cru), depois fatura **retida** (`awaiting_value`) + `invoice.held`.
- Destravar: corrija e chame `POST /v1/invoices/{id}/retry-pricing`, ou informe valor manual em `POST /v1/invoices/{id}/items/{itemId}/value`.
- **Antes de ir a produção**: `POST /v1/products/{id}/pricing-test` dispara uma chamada real com `mode: "test"` e devolve o resultado cru (`outcome`: `ok`/`skip`/`timeout`/`http_error`/`invalid_response`/`over_cap`, status, latência, corpo). Ninguém descobre integração quebrada na primeira fatura.
- Debug pós-fato: `GET /v1/invoices/{id}` expõe `pricingAttempts[]` com cada tentativa crua.
