---
name: korbit-checkout
description: >
  Como montar o catálogo e vender com a Korbit sem construir checkout: modelo
  produto → preço (versionado) → oferta → publicar → link de pagamento público,
  order bumps, sessão de checkout por API com comprador pré-preenchido e upload
  de imagens. Use ao criar fluxos de venda ou sincronizar catálogo.
---

# Korbit — Catálogo e Checkout

Pré-requisito: [[korbit-api-basics]].

## O modelo (nesta ordem)

```text
POST /v1/products            → produto (a "ficha" do que você vende)
POST /v1/products/{id}/prices → preço versionado (amountMinor em centavos)
POST /v1/products/{id}/offers → oferta: priceId + paymentMethods + maxInstallments
POST /v1/offers/{id}/publish  → publicação (gera o publicCode do link)
```

Regras:

- Preço é **imutável por versão**: mudou o preço, crie versão nova — nunca recrie o produto.
- Só oferta **publicada** vende; arquivar não cancela sessões abertas (completam dentro do TTL de 30 min).
- `maxInstallments` (1–12) limita o parcelamento; não altera o total.

## Link de pagamento

A oferta publicada tem um `publicCode`. O link é `https://korbit.com.br/o/{publicCode}` — pronto para Instagram, WhatsApp, landing page. Consulte os dados com `GET /v1/checkout-links/{publicCode}`.

## Sessão de checkout via API (comprador conhecido)

`POST /v1/checkout-sessions` cria a mesma sessão do link, com comprador pré-preenchido (cliente já cadastrado na sua base). O token da sessão:

- expira em **30 minutos**;
- vai ao browser no **fragmento** da URL (`#session=…`) e é apagado do endereço no carregamento;
- não é credencial: nunca persista no seu backend.

## Upload de imagem (3 passos)

1. `POST /v1/products/{id}/image-uploads` → recebe upload pré-assinado.
2. Envie o arquivo ao storage com body **exatamente** igual ao declarado (JPEG/PNG/WebP, ≤ 2 MiB).
3. `POST /v1/products/{id}/image-uploads/{uploadId}/complete` → a Korbit valida o objeto **armazenado** (tipo, tamanho, checksum); divergência rejeita.

## Order bumps

Ofertas complementares são configuradas na oferta/checkout e somadas ao total do pedido no submit — o comprador seleciona, o valor é sempre calculado server-side a partir do snapshot da sessão. O payload de submit **não tem campo de valor**.
