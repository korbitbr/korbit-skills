---
name: korbit-payments
description: >
  Como criar e acompanhar cobranças na API Korbit: payment intents por PIX
  (QR/copia-e-cola com expiração) e cartão (sempre via checkout hospedado com
  3DS, nunca capturando dados de cartão na sua aplicação), estados, conciliação
  por externalReference e webhooks. Use ao implementar cobranças únicas.
---

# Korbit — Pagamentos

Pré-requisito: [[korbit-api-basics]] (centavos, idempotência, erros).

## Payment intent

A unidade de cobrança. Estados: `REQUIRES_ACTION` → `SUCCEEDED` | `FAILED` | `CANCELED`.

## PIX: criação direta por API

```typescript
const res = await api.post('/v1/payment-intents', {
  amount: 14990,            // centavos
  currency: 'BRL',
  paymentMethod: 'PIX',
  description: 'Pedido #1042',
  externalReference: 'pedido-1042',   // amarra ao SEU pedido — base da conciliação
  expiresInSeconds: 900,
});
// res.status === 'REQUIRES_ACTION'
// res.pix.copyAndPaste → exiba QR + copia-e-cola ao comprador
```

Regras PIX:

- TTL padrão de 900 s; código expirado **não pode ser confirmado** — crie um novo intent (nova idempotency key).
- Valor divergente não é confirmado; nunca "arredonde" o QR.
- A confirmação chega por webhook `payment.succeeded.v1` — nunca trate "tela fechada" como pago.

## Cartão: SEMPRE via checkout hospedado

**Nunca implemente captura de dados de cartão na sua aplicação.** O fluxo de cartão da Korbit acontece na página de checkout hospedada (link público ou sessão criada por API — ver [[korbit-checkout]]): o comprador preenche o cartão em iframe seguro, o 3DS do emissor é conduzido lá, e sua aplicação só recebe o resultado.

- `cardAction` no payment intent (`REQUIRES_ACTION`) indica desafio 3DS pendente — na página hospedada é automático.
- Parcelamento é escolha do comprador, limitado ao `maxInstallments` da oferta; não altera o total.

## Consulta e conciliação

```typescript
const intent = await api.get(`/v1/payment-intents/${id}`);
```

1. Guarde `externalReference` ↔ `id` no seu banco.
2. Atualize estados pelos webhooks (`payment.created.v1`, `payment.requires_action.v1`, `payment.succeeded.v1`, `payment.failed.v1`) — ver [[korbit-webhooks]].
3. Antes de liberar produto/serviço, confirme `status === 'SUCCEEDED'` por API (o payload do webhook é aviso, não estado autoritativo).
