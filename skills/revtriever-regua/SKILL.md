---
name: revtriever-regua
description: Ler, montar e publicar a régua de recuperação da Revtriever pelas ferramentas MCP (flow_context, flow_manual, flow_apply, flow_edit, flow_step_conversion, flow_stuck_cases) — passos, ramos, mensagens, emissão de Pix/boleto e roteamento por gateway. Use ao montar ou ajustar a recuperação de cobranças recusadas, entender por que um passo não roda, ou descobrir onde a régua perde pagador.
---

<!-- Gerado por `npm run skill:regua` no revtriever-api. Não edite à mão: o catálogo de passos é a fonte. -->

# Régua de recuperação — Revtriever

A régua é o que acontece **depois** que uma cobrança falha ou vence: retentar o cartão, mandar mensagem, gerar Pix, emitir segunda via, desistir. O motor cobra; a régua recupera.

Tudo aqui é MCP, no mesmo servidor do motor (`https://motor.revtriever.com/mcp`, mesma API key `rk_…`). **Não existe REST público de régua** — se você está procurando `GET /v1/flows`, ele não existe de propósito.

| ferramenta | o que faz |
| --- | --- |
| `flow_context` | tudo numa chamada: manual, a régua como documento, modelos, variáveis e gateways |
| `flow_manual` | só o manual dos passos, quando você já tem o documento |
| `flow_apply` | publica um documento como **versão nova** — a única escrita |
| `flow_edit` | edição incremental do rascunho, passo a passo |
| `flow_step_conversion` | quantos casos alcançam cada passo e quanto cada um recupera |
| `flow_stuck_cases` | passo que passou da hora e não rodou, com o valor preso |

## As três regras que quebram régua se ignoradas

1. **Passo de emissão não envia nada.** `pix` gera o código, `boleto` emite a segunda via e `payment_options` prepara a página onde o pagador escolhe. Quem fala com o pagador é um **passo de mensagem depois**, citando a variável do que foi preparado. Emissão sem mensagem em seguida = artefato que ninguém recebe.
2. **Você monta um documento; nada existe até `flow_apply`.** Monte a régua inteira em memória a partir do que `flow_context` devolveu e mande de uma vez. Publicar cria uma versão nova — **se a régua já estiver ligada, ela passa a valer para os pagadores na hora**, e a resposta diz isso em `liveNow`. Ligar uma régua desligada continua sendo ato humano no painel.
3. **A régua é podada pela capacidade do gateway.** Um passo de retentativa de cartão numa conexão Efí é recusado, não silenciosamente ignorado. O `gatewayCan` do `flow_context` é a fonte — consulte antes de montar.

## Anatomia

```
trigger            Gatilho
switch             Condição
case               Ramo
time               Espera
email_message      E-mail
whatsapp_message   WhatsApp
pix                Gerar Pix
boleto             Emitir boleto
payment_options    Formas de pagamento
card_retry         Retentar cartão principal
card_alternative   Cobrar em cartão secundário
card_sweep         Cobrar em múltiplos cartões
human_task         Tarefa humana
finish             Encerrar
```

| passo | nome | o que faz | campos |
| --- | --- | --- | --- |
| `trigger` | Gatilho | A raiz da régua: o caso nasce aqui quando uma cobrança falha ou vence. Passa direto, sem esperar nada. | sem campos próprios |
| `switch` | Condição | Pergunta um fato do caso e segue por um ramo só. Os ramos são os passos case logo abaixo dela. | field, operator |
| `case` | Ramo | Uma resposta possível da condição acima. Não faz nada sozinho: segura o caminho que vem abaixo dele. | value |
| `time` | Espera | Estaciona o caso e o acorda depois do prazo. É o que dá ritmo à régua. | delay, timeUnit |
| `email_message` | E-mail | Manda um e-mail ao pagador com o modelo escolhido. | templateKey, attachPix (opcional) |
| `whatsapp_message` | WhatsApp | Manda uma mensagem de WhatsApp ao pagador com um modelo já aprovado na Meta. | templateKey (opcional) |
| `pix` | Gerar Pix | Pede ao gateway um Pix para a dívida e guarda o código na cobrança. Não fala com o pagador. | gatewayConnectionId (opcional) |
| `boleto` | Emitir boleto | Emite a segunda via do boleto da dívida e guarda o link. Não fala com o pagador. | gatewayConnectionId (opcional) |
| `payment_options` | Formas de pagamento | Prepara a página onde o pagador escolhe entre cartão, Pix e boleto. Não fala com o pagador. | gatewayConnectionId (opcional), offersCard (opcional), offersPix (opcional), offersBoleto (opcional) |
| `card_retry` | Retentar cartão principal | Cobra de novo o cartão principal, o que falhou, respeitando o motivo da recusa e o limite de tentativas da bandeira. | sem campos próprios |
| `card_alternative` | Cobrar em cartão secundário | Cobra uma vez em um cartão secundário que o gateway guarda para aquele pagador. Caso particular de card_sweep com teto de um cartão; o editor não o oferece mais. | sem campos próprios |
| `card_sweep` | Cobrar em múltiplos cartões | Cobra, um por vez, cada outro cartão que o gateway guarda para aquele pagador, esperando entre uma tentativa e a seguinte. Para quando o pagamento entra, quando os cartões acabam ou quando o teto é atingido. | intervalMinutes (opcional), maxCards (opcional) |
| `human_task` | Tarefa humana | Abre uma tarefa na fila do time para alguém falar com o pagador, e espera até o prazo. | deadlineHours (opcional), templateKey (opcional), instructions (opcional) |
| `finish` | Encerrar | Fecha o caso com um desfecho: recovered, exhausted ou canceled. | outcome |

A condição (`switch`) pergunta sobre um destes fatos, e cada `case` abaixo dela é uma resposta (um deles pode ser `isDefault`):

| fato | o que observa | como o ramo responde |
| --- | --- | --- |
| `decline_reason` | A categoria da recusa do emissor | texto, em stringValue |
| `decline_action` | O que fazer diante daquela recusa, segundo a regra de elegibilidade | texto, em stringValue |
| `reversible` | Se o motivo da recusa permite tentar de novo | sim ou não, em booleanValue |
| `card_brand` | A bandeira do cartão que falhou | texto, em stringValue |
| `days_overdue` | Há quantos dias a cobrança venceu | número, em numberValue |
| `amount_cents` | O valor da dívida em centavos | número, em numberValue |
| `attempt_count` | Quantas tentativas de cobrança já houve | número, em numberValue |
| `has_alternative_card` | Se o gateway guarda outro cartão daquele pagador | sim ou não, em booleanValue |
| `payment_method` | Como o pagador pagava: card, pix ou boleto | texto, em stringValue; com operator "in", separado por vírgula |

## Como eu leio a régua que existe

```
flow_context {}
```

Volta o manual, as réguas da empresa como documento (`document`) e em árvore de texto (`outline`), o `gatewayCan` de cada uma, os modelos de e-mail e WhatsApp que um passo pode citar em `templateKey` — **com as variáveis que cada modelo entrega ao pagador** — e as conexões de gateway.

As `key` do documento são as que `flow_apply` aceita. Chame `flow_context` **antes** de qualquer edição: chave inventada é recusa garantida.

## Como eu monto e publico

Edite o `document` que veio e mande inteiro:

```
flow_apply {
  "flowId": "01a05585-84bc-75bf-9670-c19b77242cb7",
  "document": { "name": "Fluxo padrão", "root": { "key": "gatilho", "type": "trigger", "name": "cobrança falhou", "children": [ /* … */ ] } }
}
```

```jsonc
{ "applied": true, "reason": null, "problems": [], "warnings": [], "version": 12, "liveNow": true, "steps": 34 }
```

**Recusa é `applied: false` com a lista em `problems`, e nada é gravado:**

```jsonc
{ "applied": false, "reason": "O documento tem problemas que impedem a publicação. Corrija-os e mande de novo.",
  "problems": [{ "key": "espera", "reason": "O caminho que passa por “espera” não termina. Adicione um passo Encerrar abaixo dele…" }] }
```

`warnings` não impede publicar, mas é o que costuma virar régua que não recupera: modelo que não carrega o link do que foi preparado, condição sem ramo padrão, passo que o gateway não executa.

## Variáveis das mensagens

| variável | o que vira | depende de |
| --- | --- | --- |
| `{{nome}}` | Como o pagador está cadastrado na cobrança | nada |
| `{{valor}}` | Formatado em reais, ex. R$ 389,00 | nada |
| `{{empresa}}` | O nome que o pagador reconhece, vindo da identidade de envio | nada |
| `{{vencimento}}` | Dia e mês do vencimento da cobrança, ex. 15/08 | nada |
| `{{codigo_pix}}` | O copia-e-cola da cobrança, quando já gerado — no e-mail ele vem no bloco de Pix, não no texto | passo `pix` antes |
| `{{link_pagamento}}` | O link rastreado da cobrança, onde o pagador escolhe entre cartão e Pix — no WhatsApp ele já vai no botão | nada |
| `{{link_pix}}` | Abre a página de pagamento direto no Pix, com o QR e o copia-e-cola | passo `pix` antes |
| `{{link_boleto}}` | Abre a segunda via emitida pela régua; compensa em até 3 dias úteis | passo `boleto` antes |
| `{{link_formas_pagamento}}` | Abre a nossa página com as formas que o passo de formas de pagamento liberou — cartão, Pix e boleto | passo `payment_options` antes |
| `{{link_trocar_cartao}}` | Leva à página do gateway onde o pagador cadastra outro cartão; sem ela, abre a nossa página no cartão | nada |

## O que cada gateway deixa a régua fazer

A verdade da conexão está em `gatewayCan` no `flow_context` — esta tabela é o mapa, não o território:

| gateway | retentar cartão | outro cartão | Pix pela régua | cancelar a cobrança de origem |
| --- | --- | --- | --- | --- |
| Asaas | ✓ | ✓ | ✓ | ✓ |
| Vindi | ✓ | ✓ | ✓ | ✓ |
| pagar.me | ✓ | ✓ | ✓ | ✓ |
| Mercado Pago | ✓ | ✓ | ✓ | ✓ |
| Stripe | ✓ | ✓ | ✗ | ✓ |
| Malga | ✓ | ✓ | ✓ | ✗ |
| Efí | ✗ | ✗ | ✓ | ✓ |
| Nuvemshop | ✗ | ✗ | ✗ | ✗ |

Efí e Nuvemshop não recobram cartão — na Nuvemshop quem recobra é o comprador, na loja, porque a Payment Provider API não tem recorrência. Régua nesses gateways vive de mensagem, Pix e boleto.

## Como eu mudo por onde a emissão sai

Dois níveis, e o do passo ganha:

```
routing_rules {}                                            // regra da empresa, por método
routing_rule_set { "method": "pix", "connectionId": "…" }   // null remove a regra
```

No documento, `gatewayConnectionId` em `pix`, `boleto` e `payment_options` manda aquele passo emitir por outra conexão; `null` volta a seguir a regra da empresa. Serve para cobrar num gateway e receber por outro — o Pix sai onde é barato, a cobrança de origem é cancelada onde dá (coluna acima). Os ids das conexões vêm em `connections`, no `flow_context`.

Roteamento cruzado só fecha quando o **destino** emite o método e a **origem** sabe cancelar a cobrança antiga — senão o pagador fica com duas cobranças vivas para a mesma dívida.

## Como eu descubro onde a régua perde gente

```
flow_step_conversion { "sinceDays": 90 }
flow_stuck_cases { "graceMinutes": 60 }
```

`flow_step_conversion` diz quantos casos alcançaram cada passo e quanto cada um recuperou — queda grande entre dois passos vizinhos é onde vale mexer. `flow_stuck_cases` mostra passo que já passou da hora e não rodou, com o valor preso.

## Pegadinhas confirmadas

- **`trigger`** — Existe sempre, é único e fica sozinho no topo. Não se cria e não se remove.
- **`switch`** — Se nenhum ramo casar, o caso para ali e não anda mais. Um ramo com isDefault true é o que impede isso.
- **`case`** — Só existe logo abaixo de uma condição.
- **`email_message`** — Com attachPix true e nenhum Pix vivo na cobrança, o passo é pulado e o pagador não recebe nada — só marque isso depois de um passo pix.
- **`whatsapp_message`** — Sem número conectado ou sem modelo aprovado, o passo existe e não sai — a lista de modelos vem vazia nesse caso.
- **`pix`** — Se a cobrança já tem Pix vivo, o passo conclui sem emitir outro.
- **`boleto`** — Se já existe boleto vivo para a cobrança, o passo conclui sem emitir outro.
- **`payment_options`** — Sem nenhuma forma possível para aquela cobrança o passo falha, e o caso para ali.
- **`card_retry`** — Cobrança que não era de cartão pula o passo, e o motivo da recusa pode segurar a retentativa para mais tarde.
- **`card_alternative`** — Sem outro cartão guardado o passo não tem o que cobrar — a condição has_alternative_card é quem separa isso antes.
- **`card_sweep`** — O cartão que falhou nunca entra na varredura, e uma recusa de perda, roubo ou fraude encerra o passo na hora: insistir nos cartões irmãos do mesmo pagador é o padrão que o adquirente lê como teste de cartão. Cada passo tenta cada cartão uma vez, e um cartão cuja recusa a regra classifica como irreversível sai do caso de vez: nenhum outro passo volta nele.
- **`human_task`** — Ninguém agindo até o prazo, a tarefa fecha sozinha e o caso segue em frente.
- **`finish`** — Todo caminho da régua precisa terminar em um.
- **`flow_apply` é tudo-ou-nada.** Um problema em qualquer ponto do documento descarta a publicação inteira; leia `problems`.
- **Editar a régua exige plano com régua editável.** Sem ele, `flow_apply` devolve `billing.editable_flows_plan_required` — a leitura continua liberada.
- **Ligar a régua não está aqui.** Publicar cria a versão; ligar uma régua desligada é ato humano no painel.
