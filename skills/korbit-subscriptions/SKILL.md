---
name: korbit-subscriptions
description: >
  Assinaturas recorrentes na Korbit: criação via checkout com oferta recorrente
  (cartão ou PIX Automático), faturas por ciclo, cancelamento imediato, portal
  do cliente com sessões criadas/revogadas por API. Use ao implementar planos
  recorrentes e gestão de ciclo de vida do assinante.
---

# Korbit — Assinaturas

Pré-requisito: [[korbit-api-basics]], [[korbit-checkout]] (a assinatura nasce de uma oferta recorrente).

## Ciclo de vida

```text
Checkout aprovado → ATIVA → (cobrança por ciclo: fatura paga → ATIVA | falha → retentativas)
ATIVA → cancelamento → CANCELADA
```

## Regras essenciais

- O valor das faturas é o da oferta **no momento da assinatura** — alterar a oferta depois não afeta assinaturas existentes.
- Cancelamento (`POST /v1/subscriptions/{id}/cancel`) é **imediato**: nenhuma fatura nova; as pagas não são afetadas. Requer `Idempotency-Key`.
- PIX recorrente usa PIX Automático — transparente para a sua integração.

## Consultas

```typescript
const subs = await api.get('/v1/subscriptions', { cursor, limit });
const sub = await api.get(`/v1/subscriptions/${id}`);
const invoices = await api.get(`/v1/subscriptions/${id}/invoices`); // faturas por ciclo
```

Use `invoices` como base da conciliação de recorrência. Eventos relevantes chegam via [[korbit-webhooks]] (`refund.*` quando há contestação pós-cancelamento).

## Portal do cliente (autosserviço)

O comprador pode cancelar e trocar de plano sozinho no portal hospedado. O acesso é controlado por **sessões**:

```typescript
// Abrir sessão para o cliente (link único, curta duração)
await api.post('/v1/customer-portal-sessions', { customerId });

// Revogar todas as sessões do cliente (suspeita de acesso indevido, troca de e-mail)
await api.post(`/v1/customers/${id}/portal-sessions/revoke`);
```

**Nunca armazene o token do portal no seu backend como credencial** — é link efêmero de escopo único. Envie ao cliente pelo seu canal de e-mail/chat.
