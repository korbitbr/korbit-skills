---
name: korbit-refunds
description: >
  Devoluções e contestações na Korbit: criação de refunds totais/parciais com
  reserva transacional, modos de aprovação (instantâneo, revisão, política de
  risco), casos com prazo de resposta do merchant (aprovar/contestar/responder)
  e disputas/chargebacks. Use ao implementar pós-venda e gestão de contestações.
---

# Korbit — Refunds e Disputas

Pré-requisito: [[korbit-api-basics]]. Exige escopo `refunds:write` para criar.

## Criar refund

```typescript
const refund = await api.post('/v1/refunds', {
  paymentIntentId,      // pagamento de origem
  amountMinor: 14990,   // ≤ saldo restante do pagamento (parcial ou total)
  reason: 'produto_nao_entregue',
}, { idempotencyKey });
```

- Refunds acumulados **nunca excedem** o valor pago (validado transacionalmente).
- O valor é **reservado antes** da execução; falha reverte a reserva integralmente.
- Modo de aprovação depende do pagamento/valor/histórico: `INSTANT` (executa já), `OPERATIONS` (revisão Korbit), `RISK_POLICY` — a resposta indica o caminho.

## Casos de revisão (refund cases)

Quando o refund abre um caso, existe **prazo de resposta** (`review_deadline_at`, também no evento `refund.requested.v1`):

```typescript
await api.post(`/v1/refund-cases/${caseId}/merchant-approve`);  // concorda com a devolução
await api.post(`/v1/refund-cases/${caseId}/merchant-contest`);  // contesta com argumentação
await api.post(`/v1/refund-cases/${caseId}/merchant-responses`); // info/evidências adicionais
```

**Sem resposta no prazo, aplica-se a política padrão da Korbit** (pode virar refund automático) — monitore `refund.requested.v1` e `refund.updated.v1`.

## Disputas e chargebacks

Contestação aberta pelo comprador no banco (chargeback, MED) chega como `dispute.opened.v1` e segue fluxo próprio com upload de evidências e prazos regulatórios. O impacto financeiro reserva o valor da mesma forma que um refund — acompanhe o saldo reservado no breakdown.

## Checklist de implementação

1. Endpoint de webhook ativo para `refund.*` e `dispute.*` (ver [[korbit-webhooks]]).
2. Alerta no seu sistema para `review_deadline_at` próximo.
3. Conciliação: estados por `GET /v1/refund-cases` / `GET /v1/refund-cases/{caseId}`, nunca apenas por notificação.
