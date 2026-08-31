---
name: revtriever-regua
description: Ler e editar a régua de recuperação da Revtriever pelas ferramentas MCP (flow_tree, flow_edit, flow_message_templates, flow_gateways, flow_step_conversion, flow_stuck_cases) — passos, ramos, mensagens, emissão de Pix/boleto e roteamento por gateway. Use ao montar ou ajustar a recuperação de cobranças recusadas, entender por que um passo não roda, ou descobrir onde a régua perde pagador.
---

# Régua de recuperação — Revtriever

A régua é o que acontece **depois** que uma cobrança falha ou vence: retentar o cartão, mandar mensagem, gerar Pix, emitir segunda via, desistir. O motor cobra; a régua recupera.

Tudo aqui é MCP, no mesmo servidor do motor (`https://motor.revtriever.com/mcp`, mesma API key `rk_…`). **Não existe REST público de régua** — se você está procurando `GET /v1/flows`, ele não existe de propósito: a régua se lê e se edita pelas ferramentas abaixo.

| ferramenta                | o que faz                                                                 |
| ------------------------- | ------------------------------------------------------------------------- |
| `flow_tree`               | a régua inteira como está no rascunho, com o que o gateway sabe executar  |
| `flow_message_templates`  | os modelos que um passo de mensagem pode referenciar                      |
| `flow_gateways`           | as conexões ativas e o que cada uma sabe fazer — a origem do `connectionId` |
| `flow_edit`               | adiciona, altera e remove passos do **rascunho**, em lote e tudo-ou-nada  |
| `flow_step_conversion`    | quantos casos alcançam cada passo e quanto cada um recupera               |
| `flow_stuck_cases`        | passo que passou da hora e não rodou, com o valor preso                   |

## As três regras que quebram régua se ignoradas

1. **Passo de emissão não envia nada.** `pix` gera o código, `boleto` emite a segunda via e `payment_options` prepara a página onde o pagador escolhe. Quem fala com o pagador é um **passo de mensagem depois**, citando a variável do que foi preparado. Emissão sem mensagem em seguida = artefato que ninguém recebe.
2. **Você edita o rascunho, não o que está no ar.** `flow_edit` mexe na versão de trabalho; nada muda para o pagador até alguém publicar no painel. Publicar e ligar são atos humanos e **não existem como ferramenta** — não procure.
3. **A régua é podada pela capacidade do gateway.** Um passo de retentativa de cartão numa conexão Efí é recusado na edição, não silenciosamente ignorado. O `gatewayCan` do `flow_tree` é a fonte — consulte antes de montar.

## Anatomia

```
trigger  (a cobrança falhou — existe sempre, é único, não se cria)
└── switch (pergunta) → case (resposta) → case (resposta)
    ├── time            espera N minutes|hours|days
    ├── card_retry      recobra o mesmo cartão
    ├── card_alternative recobra outro cartão que o gateway guarda
    ├── pix | boleto | payment_options   preparam o artefato
    ├── email_message | whatsapp_message  falam com o pagador
    ├── human_task      cai na fila de alguém do time
    └── finish          recovered | exhausted | canceled
```

`switch` pergunta sobre `decline_reason`, `decline_action`, `reversible`, `card_brand`, `days_overdue`, `amount_cents`, `attempt_count` ou `has_alternative_card`; cada `case` é uma resposta (um deles pode ser `isDefault`).

## Como eu leio a régua que existe

```
flow_tree {}
```

```jsonc
{
  "flows": [{
    "id": "01a05585-84bc-75bf-9670-c19b77242cb7",
    "name": "Fluxo padrão",
    "active": true,
    "provider": "asaas",
    "gatewayCan": { "cardRetry": true, "alternativeCard": true, "pix": true, "chargeCancel": true, "boleto": true },
    "tree": { "id": "…", "type": "trigger", "name": "cobrança falhou", "children": [ /* … */ ] }
  }]
}
```

Os `id` daí são os que `flow_edit` aceita. Chame `flow_tree` **antes** de qualquer edição: id inventado é recusa garantida.

## Como eu descubro o `templateKey` de uma mensagem

```
flow_message_templates {}
```

```jsonc
{
  "email": [{ "templateKey": "01a04f…", "name": "Cobrança que dá para pagar agora", "subject": "…" }],
  "whatsapp": [{ "templateKey": "01a050…", "name": "cobranca_em_aberto_com_pix", "displayName": "Cobrança em aberto com Pix" }]
}
```

Só aparece o que a régua consegue enviar: e-mail publicado e WhatsApp já aprovado na Meta. Lista de WhatsApp vazia normalmente significa que a empresa ainda não conectou o número — o passo até existe, mas não sai.

## Como eu adiciono Pix na régua (o exemplo que ensina a regra 1)

Dois passos numa leva só: o que prepara e o que avisa. O `ref` liga um ao outro sem você conhecer o id do primeiro.

```
flow_edit {
  "flowId": "01a05585-84bc-75bf-9670-c19b77242cb7",
  "operations": [
    { "op": "add_step", "parentId": "<id do passo depois do qual entra>", "ref": "gerar",
      "step": { "type": "pix", "name": "gerar código Pix" } },
    { "op": "add_step", "parentId": "ref:gerar",
      "step": { "type": "email_message", "name": "avisar com o Pix", "templateKey": "01a04f…" } }
  ]
}
```

```jsonc
{
  "applied": true,
  "reason": null,
  "created": [{ "ref": "gerar", "id": "01a055a6-…", "type": "pix", "name": "gerar código Pix" }, { "id": "…", "type": "email_message" }],
  "updatedIds": [], "removedIds": [], "pendingProblems": []
}
```

O modelo citado em `templateKey` precisa conter `{{codigo_pix}}` ou `{{link_pix}}` — senão o pagador recebe um e-mail que não diz como pagar.

**Recusa é `applied: false` com o motivo em `reason`, e nada é aplicado** (a leva é tudo-ou-nada):

```jsonc
{ "applied": false, "reason": "O gateway desta régua (efi) não executa card_retry, …", "created": [], "updatedIds": [], "removedIds": [] }
```

## Variáveis das mensagens

| variável                    | o que vira                                     | depende de                        |
| --------------------------- | ---------------------------------------------- | --------------------------------- |
| `{{nome}}` `{{valor}}` `{{empresa}}` `{{vencimento}}` | dados da cobrança            | nada                              |
| `{{codigo_pix}}`            | o copia-e-cola                                 | passo `pix` antes                 |
| `{{link_pix}}`              | página com o QR                                | passo `pix` antes                 |
| `{{link_boleto}}`           | a segunda via                                  | passo `boleto` antes              |
| `{{link_formas_pagamento}}` | página onde o pagador escolhe como pagar       | passo `payment_options` antes     |
| `{{link_pagamento}}`        | checkout da cobrança                           | nada                              |
| `{{link_trocar_cartao}}`    | página de troca de cartão                      | nada                              |

## O que cada gateway deixa a régua fazer

A verdade da conexão está em `gatewayCan` no `flow_tree` — esta tabela é o mapa, não o território:

| gateway      | retentar cartão | outro cartão | Pix pela régua | cancelar a cobrança de origem |
| ------------ | --------------- | ------------ | -------------- | ----------------------------- |
| Asaas        | ✓               | ✓            | ✓              | ✓                             |
| Vindi        | ✓               | ✓            | ✓              | ✓                             |
| pagar.me     | ✓               | ✓            | ✓              | ✓                             |
| Mercado Pago | ✓               | ✓            | ✓              | ✓                             |
| Stripe       | ✓               | ✓            | ✗              | ✓                             |
| Malga        | ✓               | ✓            | ✓              | ✗                             |
| Efí          | ✗               | ✗            | ✓              | ✓                             |
| Nuvemshop    | ✗               | ✗            | ✗              | ✗                             |

Efí e Nuvemshop não recobram cartão — na Nuvemshop quem recobra é o comprador, na loja, porque a Payment Provider API não tem recorrência. Régua nesses gateways vive de mensagem, Pix e boleto.

## Como eu mudo por onde a emissão sai

Dois níveis, e o do passo ganha:

```
routing_rules {}                                   // regra da empresa, por método
routing_rule_set { "method": "pix", "connectionId": "…" }   // null remove a regra
```

```
flow_edit { "flowId": "…", "operations": [
  { "op": "update_step", "stepId": "…", "patch": { "gatewayConnectionId": "…" } }
]}
```

`gatewayConnectionId` em `pix`, `boleto` e `payment_options` manda aquele passo emitir por outra conexão; `null` volta a seguir a regra da empresa. Serve para cobrar num gateway e receber por outro — o Pix sai onde é barato, a cobrança de origem é cancelada onde dá (coluna "cancelar a cobrança de origem" acima).

Os ids vêm de `flow_gateways`:

```jsonc
{ "connections": [{
  "connectionId": "01a02f7d-78a6-774e-addc-9c672a5e829a",
  "provider": "asaas", "environment": "production",
  "can": { "cardRetry": true, "alternativeCard": true, "pix": true, "chargeCancel": true },
  "issuableMethods": ["card", "pix", "boleto"]
}]}
```

Roteamento cruzado só fecha quando o **destino** emite o método e a **origem** sabe cancelar a cobrança antiga — senão o pagador fica com duas cobranças vivas para a mesma dívida.

## Como eu descubro onde a régua perde gente

```
flow_step_conversion { "sinceDays": 90 }
flow_stuck_cases { "graceMinutes": 60 }
```

`flow_step_conversion` diz quantos casos alcançaram cada passo e quanto cada um recuperou — queda grande entre dois passos vizinhos é onde vale mexer. `flow_stuck_cases` mostra passo que já passou da hora e não rodou, com o valor preso: fila travada, template reprovado na Meta e gateway fora do ar aparecem aí antes de virarem prejuízo.

## Pegadinhas confirmadas

- **`flow_edit` é tudo-ou-nada.** Uma operação inválida no meio da leva descarta a leva inteira; leia `reason`.
- **`ref:` só vale dentro da mesma leva** e só para passos criados nela.
- **`add_step` insere no caminho por padrão** (`insert: "chain"`: os filhos atuais do pai descem para baixo do novo passo). Para pendurar como mais um filho, `insert: "append"`.
- **O gatilho não se cria e não se remove** — ele já existe e é único.
- **Editar a régua exige plano com régua editável.** Sem ele, `flow_edit` devolve `billing.editable_flows_plan_required` — a leitura continua liberada.
- **Publicar não está aqui.** Depois de editar, alguém revisa e publica no painel; até lá o pagador não vê diferença nenhuma.
