---
name: korbit-payouts
description: >
  Dinheiro saindo da Korbit: saldo e breakdown, liberações de recursos
  (funds releases) com calendário, beneficiários PIX (criptografia e chave
  mascarada), solicitação de payouts com reserva transacional e ciclo de
  aprovação, e antecipações (cotação e solicitação). Use ao implementar saques,
  conciliação de disponibilidade ou antecipação de recebíveis.
---

# Korbit — Saldo, Saques e Antecipações

Pré-requisito: [[korbit-api-basics]]. Operações de saque exigem escopo `payouts:write`.

## Saldo e liberações

```typescript
const balance = await api.get('/v1/balance');           // disponível, a liberar, reservado
const detail  = await api.get('/v1/balance/breakdown');
const calendar = await api.get('/v1/funds/releases/calendar'); // projeção de caixa
```

- **Disponível** é a fonte de verdade para saques.
- Liberação pode estar **bloqueada** por refunds/disputas abertos — os motivos aparecem no item.
- Todo valor de pagamento tem data de liberação; exponha o calendário no seu produto em vez de prometer D+0.

## Beneficiários (chaves PIX de destino)

```typescript
await api.post('/v1/payout-beneficiaries', { pixKey: '...', pixKeyType: 'CPF' }); // CPF|CNPJ|EMAIL|PHONE|EVP
await api.get('/v1/payout-beneficiaries');  // chaves sempre MASCARADAS
await api.patch(`/v1/payout-beneficiaries/${id}/primary`);
```

- Chaves são armazenadas criptografadas e **nunca retornadas por inteiro**.
- Confirme o titular antes de cadastrar — beneficiário é destino de dinheiro.
- Desabilitar (`DELETE /v1/payout-beneficiaries/{id}`) impede novos saques.

## Payout (saque)

```typescript
const payout = await api.post('/v1/payout-requests', {
  amountMinor: 150000,
  currency: 'BRL',
  beneficiaryId,             // id do beneficiário
}, { idempotencyKey });
```

- O valor é **reservado transacionalmente** na solicitação — requests concorrentes nunca excedem o disponível.
- Ciclo: `PENDING_REVIEW` → `APPROVED` → `EXECUTED` (com comprovante) → `RECONCILIADO`. **Há verificação humana na Korbit** — não é instantâneo.
- Falha de execução devolve o valor ao disponível com motivo registrado.
- Payout mínimo (ex.: 1000 centavos) e rate limit de 5/min.

## Antecipações

```typescript
await api.get('/v1/advance/receivables');     // recebíveis futuros
await api.get('/v1/advance/eligibility');      // critérios de risco
const quote = await api.post('/v1/advance/quotes', { ... });  // bruto, taxa, líquido, validade
await api.post('/v1/advance/requests', { ...quoteRef });      // passa por revisão Korbit
```

- Cotação expirada → `409 ADVANCE_QUOTE_EXPIRED`.
- **Um recebível, uma garantia**: não pode sustentar duas antecipações (reserva transacional).
- Taxa depende do tempo até a liberação — antecipar perto da data custa menos.
