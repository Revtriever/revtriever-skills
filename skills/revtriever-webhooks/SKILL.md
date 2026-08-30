---
name: revtriever-webhooks
description: Consumir webhooks e eventos do Motor Revtriever — envelope, catálogo de tipos (invoice.issued, invoice.paid, pricing.failing…), deduplicação, replay por cursor e verificação de assinatura. Use ao implementar o receptor de webhooks, reagir a eventos de cobrança, ou reconciliar eventos perdidos.
---

# Webhooks e eventos — Revtriever

Um endpoint só (`PUT /v1/integration/alert-webhook`, URL https; `null` desliga) recebe **tudo**: transições de cobrança e ciclo de vida de recursos. Guia: https://docs.revtriever.com/guias/webhooks-e-eventos/

## As três regras do consumo correto

1. **At-least-once, sem ordem garantida**: deduplique pelo `id` do envelope e não assuma sequência.
2. **`data` é o retrato atual do recurso** no momento do evento — não um diff.
3. **`GET /v1/events` é a fonte de verdade**: todo evento é persistido e re-consultável por cursor, inclusive o que o webhook perdeu. Reconcilie por lá, não acumule estado só do push.

## Envelope

```json
{
  "id": "0199c5…",
  "type": "invoice.issued",
  "occurredAt": "2026-09-28T11:04:00.000Z",
  "data": { "invoiceId": "…", "subscriptionId": "…", "totalCents": 119952, "dueDate": "2026-10-01", "cycleDate": "2026-09-28", "gatewayInvoiceId": "…" }
}
```

Em `invoice.issued`/`invoice.overdue`: `cycleDate` = âncora do ciclo (o `chargeDay`), `dueDate` = vencimento real (`cycleDate + expirationDays`). Eventos `pricing.*` carregam `customerExternalRef` (o **seu** id do cliente). Toda entrega sai assinada com o secret de integração — verificação HMAC idêntica à do endpoint de preço (skill `revtriever-pricing-endpoint`).

## Catálogo

**Transições de cobrança** (as que disparam ação sua):
`invoice.issued` · `invoice.paid` · `invoice.overdue` (venceu ou recusada) · `invoice.held` (retida — intervenção necessária) · `pricing.failing` (seu endpoint de preço falhando) · `pricing.recovered` · `subscription.canceled`

**Ciclo de vida de recursos** (inclui mudanças via dashboard e MCP):
`invoice.created` · `product.created|updated` (arquivar = `updated`) · `plan.created|updated` (inclui nova versão) · `customer.created` · `subscription.created|updated` (inclui avanço de ciclo) · `api_key.created|revoked`

## Replay por cursor

```bash
curl "$MOTOR/events?limit=50" -H "$AUTH"                  # do começo, mais antigo primeiro
curl "$MOTOR/events?cursor=<id-do-último>" -H "$AUTH"     # continua de onde parou
curl "$MOTOR/events?type=invoice.paid&from=2026-09-01T00:00:00Z" -H "$AUTH"
```

Resposta `{ data[], nextCursor }` — `nextCursor: null` = acabou. O `id` de cada evento é o cursor. Cada evento traz `delivery` (`pending`/`delivered`/`failed`/`skipped` + `attempts` + `lastError`) — `skipped` = você não tinha webhook cadastrado na hora.

## Teste

`POST /v1/integration/alert-webhook/test` entrega um evento de exemplo **assinado** e devolve o resultado cru (`delivered`, `responseStatus`, `latencyMs`, `error`). `409` = nenhum webhook cadastrado.
