---
name: korbit-api-basics
description: >
  Regras fundamentais da API Korbit que valem para toda integração: autenticação
  por chave de API (kbt_live_/kbt_test_), base URL por ambiente, valores em
  centavos inteiros (BRL), idempotência obrigatória em mutações, erros RFC 7807
  com códigos estáveis e rate limits. Use esta skill sempre que escrever código
  que chama a API Korbit — as demais skills korbit-* assumem estas regras.
---

# Korbit — Fundamentos da API

Toda chamada à API Korbit segue estas regras. Leia antes de escrever qualquer integração.

## Autenticação e ambientes

- Chave no header: `Authorization: Bearer <chave>`. Formato `kbt_{live|test}_{publicId}_{secret}`.
- **O prefixo da chave define o ambiente** — nunca existe parâmetro de ambiente na request:

| Chave | Base URL |
| --- | --- |
| `kbt_live_…` | `https://api.korbit.com.br` |
| `kbt_test_…` | `https://api-test.korbit.com.br` |

- Uma chave `test` é recusada no ambiente live e vice-versa.
- Chave de API é **segredo de servidor**: nunca vai para client-side. O comprador usa link público ou token de sessão efêmero.
- Principais escopos: `payments:write/read`, `commerce:write/read`, `payouts:write/read`, `refunds:write/read`, `webhooks:manage`, `iam:manage`. Escopo insuficiente → `403 INSUFFICIENT_SCOPE`. Menor privilégio, uma chave por integração.

## Valores: sempre centavos inteiros

- Moeda única **BRL**, valores em **inteiros**: `14990` = R$ 149,90.
- Jamais envie `14.99` (float) — o schema rejeita.
- Formatação para exibição: `(14990 / 100).toLocaleString('pt-BR', { style: 'currency', currency: 'BRL' })`.

## Idempotência (obrigatória em mutações de dinheiro)

- Envie `Idempotency-Key: <uuid-v4>` em **toda** mutação financeira (payment intent, payout, refund).
- Mesma chave + mesmo payload → a API repete a **resposta original** (nada é executado duas vezes).
- Mesma chave + payload diferente → `409 IDEMPOTENCY_KEY_REUSED`.
- Original ainda processando → `409 IDEMPOTENCY_IN_PROGRESS` (respeite `Retry-After`).
- Regra de ouro: **nova intenção de negócio = nova chave**; retry do mesmo request = mesma chave. Gere a chave ANTES do fetch e guarde-a até a resposta final.

```typescript
const idempotencyKey = crypto.randomUUID();
const res = await fetch(base + '/v1/payment-intents', {
  method: 'POST',
  headers: { Authorization: `Bearer ${key}`, 'Content-Type': 'application/json', 'Idempotency-Key': idempotencyKey },
  body: JSON.stringify({ amount: 14990, currency: 'BRL', paymentMethod: 'PIX' }),
});
// Em timeout/5xx: refaça com a MESMA chave.
```

## Erros: RFC 7807 com `code` estável

Respostas de erro são `application/problem+json`:

```json
{ "type": "...", "title": "...", "status": 409, "code": "IDEMPOTENCY_KEY_REUSED", "detail": "...", "requestId": "…" }
```

- **Decida a lógica pelo `code`, nunca pelo texto** do `detail`.
- Sempre logue o `requestId` — é o que o suporte usa para investigar.
- Códigos frequentes: `INVALID_API_KEY` (401), `INSUFFICIENT_SCOPE` (403), `MERCHANT_CAPABILITY_DISABLED` (403), `PAYMENT_INTENT_NOT_FOUND` (404), `IDEMPOTENCY_KEY_REQUIRED/REUSED/IN_PROGRESS`, `RATE_LIMITED` (429), `RATE_LIMIT_UNAVAILABLE` (503).

## Rate limits

- Limites por rota: payment intent 60/min, payout 5/min, beneficiário 10/min, chaves de API 10/min.
- Headers `Rate-Limit-Limit/Remaining/Reset`; excesso → `429` com `Retry-After` (em segundos).
- Implemente **backoff exponencial com jitter**; `503 RATE_LIMIT_UNAVAILABLE` é fail-closed (recue e tente depois).

## Fonte autoritativa

O estado de qualquer entidade é o que a **API retorna** — webhooks aceleram, mas não substituem a consulta. Ao tomar decisões sensíveis, confirme o estado atual por API.
