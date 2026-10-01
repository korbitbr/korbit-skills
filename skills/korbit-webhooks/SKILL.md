---
name: korbit-webhooks
description: >
  Webhooks da Korbit: catálogo dos 13 tipos de evento versionados (.v1),
  verificação de assinatura Svix com código pronto (Node e sem SDK), entrega
  at-least-once com agenda de retries, deduplicação por id, estados da
  assinatura e gestão via /v1/webhook-subscriptions. Use ao implementar o
  consumidor de webhooks.
---

# Korbit — Webhooks

Pré-requisito: [[korbit-api-basics]].

## Contrato

- Endpoint **HTTPS público** (endereços privados são recusados no cadastro).
- Entrega **at-least-once**: deduplique pelo `id` do evento.
- Responda **2xx rápido** (< 5 s): valide assinatura → persista → processe em background.
- Estado autoritativo é a API: confirme por API em decisões sensíveis.
- Assinatura e segredo (`whsec_…`, exibido uma única vez no painel) são por ambiente — desenvolva com assinatura `test`.

## Verificação de assinatura (padrão Svix)

Headers: `svix-id`, `svix-timestamp`, `svix-signature`. **Verifique sobre o corpo BRUTO** (antes de qualquer parse/re-serialização).

```typescript
import { Webhook } from 'svix'; // npm i svix
const wh = new Webhook(process.env.KORBIT_WEBHOOK_SECRET);

app.post('/korbit/webhooks', async (req, res) => {
  const raw = await req.text(); // corpo bruto!
  let event;
  try {
    event = wh.verify(raw, {
      'svix-id': req.headers['svix-id'],
      'svix-timestamp': req.headers['svix-timestamp'],
      'svix-signature': req.headers['svix-signature'],
    });
  } catch {
    return res.status(400).end(); // assinatura inválida: NUNCA processe
  }
  await queue.enqueue(event);   // persista e responda rápido
  res.status(202).end();
});
```

Sem SDK: `HMAC-SHA256(secret, "{id}.{timestamp}.{payload}")` em base64, comparado com `timingSafeEqual` contra cada assinatura do header (separadas por espaço), com tolerância de timestamp de 5 minutos (anti-replay).

## Catálogo de eventos (versionados — novos eventos são aditivos)

`payment.created.v1` · `payment.requires_action.v1` · `payment.succeeded.v1` · `payment.failed.v1` · `refund.requested.v1` · `refund.updated.v1` · `refund.succeeded.v1` · `payout.requested.v1` · `payout.updated.v1` · `dispute.opened.v1` · `dispute.updated.v1` · `merchant.kyc.updated.v1` · `merchant.status.updated.v1`

Payload: envelope Svix com `type`, `timestamp` e `data` em `snake_case` (`amount_minor` em centavos; campos variam por tipo — ex.: `review_deadline_at` em eventos de refund).

## Retries

Falha = não-2xx, timeout ou conexão. Agenda com backoff progressivo (~35 h): imediato → 5 s → 5 min → 30 min → 2 h → 5 h → 10 h → 10 h. Assinatura entra em `ERROR` quando tudo esgota (nova entrega para até recuperação — monitore `GET /v1/webhook-subscriptions` e alerte). Eventos com agenda esgotada podem ser reenviados pelo suporte; depois de um incidente, faça **reconciliação por API** do período.

## Gestão

`POST /v1/webhook-subscriptions` (`url` + `eventTypes[]`, uma assinatura por URL) · `GET /v1/webhook-subscriptions` · `POST /v1/webhook-subscriptions/{id}/disable`. Assine só o que consome.
