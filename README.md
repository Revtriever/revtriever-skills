# Revtriever Skills

Skills de agente para integrar com o [Motor de Cobrança Revtriever](https://docs.revtriever.com) — o seu agente de código (Claude Code, Cursor, Codex…) aprende a usar a API do jeito certo: convenções, endpoint de preço com HMAC, webhooks com dedupe e replay.

## Instalação

### Qualquer agente (formato Agent Skills)

```bash
npx skills add Revtriever/revtriever-skills
```

### Claude Code (plugin: skills + MCP juntos)

```
/plugin marketplace add Revtriever/revtriever-skills
/plugin install revtriever@revtriever-skills
```

O plugin também configura o **servidor MCP do motor** (`motor.revtriever.com/mcp`), que traz o motor, a régua e a negociação. Exporte a sua API key antes de usar:

```bash
export REVTRIEVER_API_KEY=rk_sua_api_key
```

## O que vem

| Skill                         | Quando o agente usa                                                          |
| ----------------------------- | ---------------------------------------------------------------------------- |
| `revtriever-motor`            | Escrever código que chama a API: auth, idempotência, modelo de dados, erros |
| `revtriever-pricing-endpoint` | Implementar o endpoint de preço variável (contrato, HMAC, teste)             |
| `revtriever-webhooks`         | Consumir eventos: envelope, catálogo, dedupe, replay por cursor              |
| `revtriever-regua`            | Ler e editar a régua de recuperação: passos, mensagens, emissão, roteamento   |
| `revtriever-negociacao`       | Combinar, testar e publicar a política de negociação do assistente de cobrança |

## Fontes de verdade

As skills são resumos procedurais — a referência completa vive em:

- **Docs**: https://docs.revtriever.com (também em [`/llms-full.txt`](https://docs.revtriever.com/llms-full.txt) para agentes)
- **OpenAPI vivo**: https://motor.revtriever.com/docs-json

Divergiu? A API é a fonte de verdade — abra uma issue aqui.
