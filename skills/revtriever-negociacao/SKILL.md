---
name: revtriever-negociacao
description: Combinar, testar e publicar a política de negociação da Revtriever pelas ferramentas MCP (negotiation_context, negotiation_manual, negotiation_policy, negotiation_edit, negotiation_simulate, negotiation_publish, negotiation_settings, negotiation_dashboard) — formas, desconto máximo, parcelas, desistência em loja, repetição por pagador e excedente. Use ao configurar o assistente que negocia cobrança recusada pelo WhatsApp, entender por que uma operação foi recusada, ou ler quanto a negociação recuperou.
---

<!-- Gerado por `npm run skill:negociacao` no revtriever-api. Não edite à mão: o pacote de negociação é a fonte. -->

# Negociação assistida — Revtriever

A negociação é o passo da régua em que um assistente conversa com o pagador pelo WhatsApp, abre com o desconto máximo que a empresa autorizou, emite o Pix ou o boleto do acordo, parcela no cartão pela página da Revtriever e transborda para a equipe quando não é caso dele. O que ele pode ceder é a **política de negociação**, e é ela que se configura aqui.

Tudo é MCP, no mesmo servidor do motor (`https://motor.revtriever.com/mcp`, mesma API key `rk_…`). As ferramentas só aparecem quando a empresa contratou o produto **negociação** e a chave tem o escopo `negotiation`; sem o produto elas não existem, e a escrita sem o escopo responde `billing.api_key_scope_missing`.

| ferramenta | o que faz |
| --- | --- |
| `negotiation_context` | tudo numa chamada: manual, a política de cada conexão, o que ela consegue, o que é da empresa e o medidor do ciclo |
| `negotiation_manual` | só o manual: perguntas, operações, regras e como o desconto acontece em cada gateway |
| `negotiation_policy` | a política de uma conexão como está no rascunho |
| `negotiation_edit` | operações tipadas sobre o rascunho, em lote e tudo-ou-nada |
| `negotiation_simulate` | um turno do assistente contra o rascunho, fazendo o papel do pagador, sem efeitos |
| `negotiation_publish` | publica o rascunho — a única escrita que chega aos pagadores |
| `negotiation_settings` | repetição por pagador e política de excedente, que são da empresa |
| `negotiation_dashboard` | o que a negociação liquidou no período, líquido da concessão |

## As regras que quebram política se ignoradas

1. A política é de UMA conexão de gateway, não da empresa: cada conexão tem a sua, porque o desconto acontece de um jeito em cada gateway e nem toda conexão emite as mesmas formas.
2. O assistente abre a conversa já com o desconto máximo da política. Não existe escada, degrau nem "guardar para depois": o que a política permite é o que o pagador recebe na primeira mensagem.
3. A política não decide dias de atraso, promessa de pagamento para outra data nem quando transbordar para a equipe. Isso é da régua e do próprio assistente.
4. Forma que a conexão não emite é recusada na hora, não ignorada. O que a conexão pode está em connection.can.
5. Bônus do Pix só existe com Pix entre as formas; parcelamento só com cartão; desistência do pedido só em conexão de loja.
6. Faixas (de desconto e de parcelas) vêm em ordem crescente de valor e a última tem upToCents nulo.
7. Desconto acima de 40% publica, mas com aviso: revise antes.
8. Nada muda para os pagadores até publicar. Editar mexe no rascunho; publicar sobe a versão e passa a valer para as conversas que abrirem daí em diante.
9. A repetição por pagador e a política de excedente são da empresa, não da conexão: valem para todas as políticas.

## As perguntas, na ordem

A política responde a estas perguntas. `review.pending` na resposta de `negotiation_policy` é a lista do que ainda falta, nesta ordem; `publishable` só vira `true` com tudo respondido e sem problema.

| pergunta | chave | quando existe | o que decide | sugestão padrão |
| --- | --- | --- | --- | --- |
| Formas de pagamento | `methods` | fora de loja | Quais formas o assistente pode oferecer, dentro do que a conexão tem. Tem empresa que prefere não abrir boleto. | Todas as formas que a conexão tem. |
| Desconto máximo | `discount` | sempre | Quanto a empresa aceita ceder para receber agora: teto único ou por faixa de valor da cobrança, com bônus opcional para quem paga no Pix. O assistente abre a conversa já com esse desconto. | 15%, no máximo R$ 400 por cobrança. |
| Parcelamento no cartão | `installments` | com cartão entre as formas | Faixas por valor da cobrança com o máximo de parcelas, parcela mínima e até quantas sem juros. Só existe quando cartão está entre as formas. | Até R$ 300 à vista; até R$ 1.500 em 3x sem juros; acima disso em 6x. |
| Desistência do pedido | `storeCancel` | só em loja | Em loja o comprador pode não querer mais. Se ele desiste pela conversa, o assistente cancela o pedido na hora ou passa para a equipe. | Pode desistir; o assistente cancela e encerra o caso. |
| Como o assistente se apresenta | `identity` | sempre | O nome que o assistente usa e o horário em que a equipe responde quando assume. Ele sempre se declara virtual na primeira mensagem. | "Assistente da <empresa>", seg a sex 9h às 18h. |

## Anatomia do documento

```json
{
  "version": 1,
  "methods": {
    "pix": true,
    "boleto": false,
    "card": true
  },
  "discount": {
    "mode": "flat",
    "maxPercent": 15,
    "maxCents": 40000,
    "pixBonusPercent": 5
  },
  "installments": [
    {
      "upToCents": 30000,
      "maxInstallments": 1,
      "minInstallmentCents": null,
      "interestFreeUpTo": null
    },
    {
      "upToCents": null,
      "maxInstallments": 6,
      "minInstallmentCents": 10000,
      "interestFreeUpTo": 3
    }
  ],
  "storeCancel": null,
  "identity": {
    "assistantName": "Bia, assistente do Clube do Vinho",
    "teamHours": "seg a sex, 9h às 18h"
  }
}
```

Nulo é "ainda não combinado". `installments: []` é só à vista. `storeCancel` só existe em conexão de loja (Nuvemshop); fora dela fica nulo e não conta como pendência.

## As operações

Cada uma é validada contra `connection.can` antes de gravar; a leva inteira é recusada se uma não couber, e nada é gravado.

**`set_methods`** — Formas que o assistente pode oferecer. Forma que a conexão não tem é recusada; tirar o Pix apaga o bônus do Pix, tirar o cartão zera o parcelamento.

```json
{"op":"set_methods","methods":{"pix":true,"boleto":false,"card":true}}
```

**`set_discount`** — Desconto máximo: teto único (flat, com teto opcional em centavos) ou faixas por valor da cobrança em ordem crescente (bands), mais o bônus opcional de quem paga no Pix.

```json
{"op":"set_discount","discount":{"mode":"flat","maxPercent":15,"maxCents":40000,"pixBonusPercent":5}}
```

**`set_installments`** — Parcelamento no cartão por faixa de valor: máximo de parcelas (até 24), parcela mínima e até quantas sem juros. Lista vazia é só à vista. Só cabe com cartão entre as formas.

```json
{"op":"set_installments","installments":[{"upToCents":30000,"maxInstallments":1,"minInstallmentCents":null,"interestFreeUpTo":null},{"upToCents":null,"maxInstallments":6,"minInstallmentCents":10000,"interestFreeUpTo":3}]}
```

**`set_store_cancel`** — Só em loja: se o comprador desiste pela conversa, o assistente cancela o pedido na hora (true) ou passa para a equipe (false).

```json
{"op":"set_store_cancel","storeCancel":true}
```

**`set_identity`** — Como o assistente se apresenta e o horário em que a equipe responde quando assume. Ele sempre se declara virtual.

```json
{"op":"set_identity","identity":{"assistantName":"Bia, assistente do Clube do Vinho","teamHours":"seg a sex, 9h às 18h"}}
```

**`clear`** — Volta uma pergunta para "não combinado". A política deixa de ser publicável até responder de novo.

```json
{"op":"clear","key":"discount"}
```

## Como o desconto acontece em cada gateway

A verdade da conexão está em `connection.can` (`pix`, `boleto`, `card`, `kind`) no `negotiation_context`. Esta tabela é o mapa:

| gateway | como o desconto acontece |
| --- | --- |
| Vindi | Na Vindi o desconto vira uma fatura nova com os mesmos produtos e o valor menor; a antiga é cancelada. |
| Stripe | Na Stripe o desconto entra como nota de crédito na própria fatura; o link de pagamento continua o mesmo. |
| Asaas | No Asaas o desconto reduz a própria cobrança, que continua com o mesmo link. |
| Mercado Pago | No Mercado Pago o desconto reduz o próprio pagamento pendente. |
| pagar.me | No pagar.me o desconto vira um pedido novo com os mesmos itens e o valor menor. |
| Malga | Na Malga o desconto vira uma cobrança nova para o mesmo cliente. |
| Efí | Na Efí o Pix é revisado no lugar, com o mesmo código; o boleto sai novo. |
| Nuvemshop | Na Nuvemshop o desconto vira um pedido rascunho com o checkout da própria loja; o pedido original é cancelado quando o rascunho paga. |

Cartão não passa pelo gateway na conversa: o acordo em cartão fecha na página da Revtriever, com valor e parcelas travados, só nos gateways que tokenizam no navegador (`can.card`).

## Como eu leio o que existe

```
negotiation_context {}
```

Volta `manual`, `policies` (uma por conexão que emite cobrança: `connection`, `document`, `review` com `problems`, `warnings` e `pending`, `sections` em prosa, `publishable`, `version`, `publishedAt`), `repetition`, `overagePolicy` e `quota`. Os `connectionId` que as outras ferramentas aceitam saem daqui.

## Como eu combino e publico

Edite o rascunho com operações, na ordem das perguntas:

```
negotiation_edit {
  "connectionId": "01a05585-84bc-75bf-9670-c19b77242cb7",
  "operations": [
    { "op": "set_methods", "methods": { "pix": true, "boleto": false, "card": true } },
    { "op": "set_discount", "discount": { "mode": "flat", "maxPercent": 15, "maxCents": 40000, "pixBonusPercent": 5 } }
  ]
}
```

A resposta é a política depois da leva. **Recusa é erro com o motivo em português, e nada é gravado:**

```jsonc
{ "code": "negotiation.policy_invalid", "status": 422, "meta": { "reason": "Esta conexão não emite boleto." } }
```

Teste antes de publicar, fazendo o papel do pagador:

```
negotiation_simulate { "connectionId": "…", "message": "consigo 30%?", "history": [] }
```

Volta `reply` (a fala do assistente), `message` (como sairia no WhatsApp, com Pix ou botão), `outcome` (`waiting`, `handoff` com `reason`, `settled`), `offer` (o máximo em percentual, valores e texto) e `trace` (cada ferramenta aceita ou recusada). Nada é emitido nem enviado, mas cada chamada consome uma simulação do limite mensal (`negotiation.simulation_limit` quando acaba). `scenario: "gateway_failure"` mostra a emissão falhando.

Publique:

```
negotiation_publish { "connectionId": "…" }
```

Com pergunta em aberto responde `negotiation.policy_incomplete` listando o que falta em `meta.pending`. Publicada, `version` sobe e as conversas que o passo abrir daí em diante seguem a nova. Publicar **não** liga a régua nem coloca o passo nela: isso é da régua (skill `revtriever-regua`, passo `negotiation`) e continua sendo ato humano no painel.

## O que é da empresa, não da conexão

```
negotiation_settings { "repetition": { "count": 1, "everyMonths": 12 }, "overagePolicy": "pause" }
```

- **Repetição**: quantas concessões um mesmo pagador pode ganhar em quantos meses; nulos é sem limite.
- **Excedente**: ao bater as conversas incluídas no ciclo, `allow` libera cobrando por conversa na fatura do ciclo e `pause` deixa o passo parado (o caso segue para a tarefa humana).
- `quota` mostra `allowed`, `block` (`locked` sem o produto, `limit_reached` com o mês batido e política de pausar), `used` e `limit`.

Omita o que não quer mexer; a resposta traz as duas com o medidor.

## O que a negociação recuperou

```
negotiation_dashboard { "from": "2026-09-01", "to": "2026-09-30" }
```

`settled` (quantos acordos liquidaram, valor cheio, recuperado líquido, concessão em reais e em pontos-base), `withoutHuman` (os que fecharam sem gente) e `byDiscountBand`. Recuperado é o que entrou, líquido da concessão: um acordo de R$ 300 fechado por R$ 255 conta R$ 255.
