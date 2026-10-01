# korbit-skills

Skills da Korbit para agentes de IA: pacotes de contexto (formato padrão `SKILL.md`) que ensinam o seu assistente de código a integrar a API Korbit **do jeito certo** — valores em centavos, idempotência obrigatória, verificação de webhooks, ambientes separados. Sem chave de API: as skills são conhecimento, não execução.

## Skills disponíveis

| Skill | Ensina |
| --- | --- |
| `korbit-api-basics` | Autenticação, ambientes, centavos, idempotência, erros RFC 7807, rate limits — **base para todas as outras** |
| `korbit-payments` | Payment intents PIX e cartão/3DS via checkout hospedado, conciliação |
| `korbit-checkout` | Catálogo (produto → preço → oferta → publicar), links de pagamento, order bumps, uploads |
| `korbit-subscriptions` | Recorrência, faturas por ciclo, cancelamento, portal do cliente |
| `korbit-webhooks` | Assinatura Svix com código pronto, retries, deduplicação, catálogo de eventos |
| `korbit-payouts` | Saldo, liberações, beneficiários PIX, payouts, antecipações |
| `korbit-refunds` | Refunds totais/parciais, casos com prazo, contestações |

As skills se referenciam entre si — instale todas (a `korbit-api-basics` é pré-requisito conceitual das demais).

## Instalação

### Claude Code

```bash
git clone https://github.com/korbit/korbit-skills.git
mkdir -p .claude/skills
cp -r korbit-skills/skills/* .claude/skills/
```

### Cursor / outros editores

Clone o repositório e adicione o caminho `skills/` ao contexto de regras do projeto (`.cursorrules`, regras de workspace ou equivalente), ou referencie os arquivos `SKILL.md` diretamente.

### MCP (execução, não contexto)

Se o que você quer é **executar operações na sua conta** via agente, use o [MCP server da Korbit](https://docs.korbit.com.br/ia/mcp) (`npx -y @korbit/mcp`) — as skills complementam o MCP ensinando o agente a escrever a integração corretamente.

## Quando usar skills x MCP

| | Skills | MCP |
| --- | --- | --- |
| Objetivo | Gerar código de integração correto | Executar operações reais na conta |
| Chave de API | Não precisa | Sim |
| Onde roda | Contexto do editor | Runtime MCP |

## Licença

MIT
