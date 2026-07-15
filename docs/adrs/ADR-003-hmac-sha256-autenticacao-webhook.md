# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint

**Status:** Decidido

**Data:** 2026-07-12

**Participantes:** Sofia (Eng. Segurança), Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Larissa (Tech Lead)

## Contexto

O sistema enviará requisições HTTP com dados de pedidos para endpoints externos controlados pelos clientes. É necessário um mecanismo que permita ao cliente verificar duas coisas: (1) que a requisição realmente partiu do nosso sistema (autenticidade) e (2) que o payload não foi adulterado em trânsito (integridade). Adicionalmente, secrets podem vazar (ex.: log de aplicação do cliente), então o modelo precisa suportar rotação sem interrupção do serviço.

## Decisão

Usaremos **HMAC-SHA256 sobre o corpo da requisição**, com **uma secret única por endpoint de webhook** e **suporte a rotação com grace period de 24 horas**.

- A assinatura é enviada no header `X-Signature: sha256=<hash>`
- Cada configuração de webhook (`webhook_endpoints`) armazena `url`, `secret` (hash), `customer_id` e `active`
- A rotação funciona assim: ao solicitar nova secret (endpoint dedicado), a antiga permanece válida por 24h. Nesse período, o worker envia o header `X-Signature` com a secret atual, mas o endpoint de verificação do cliente pode validar com qualquer uma das duas. Após 24h, a antiga é invalidada
- URLs de webhook são validadas com Zod: se o schema não for `https://`, a criação é rejeitada com erro de validação

## Alternativas consideradas

### Secret global única para todos os clientes

**Descartada** porque:
- Se uma secret vaza (ex.: cliente expôs em log), todos os clientes ficam comprometidos
- Impossibilita rotação individual — trocar a secret global quebraria todos os clientes simultaneamente
- Já houve incidente anterior de vazamento de secret em log de aplicação de cliente

### API Key em header sem assinatura

**Descartada** porque:
- API Key simples (header `Authorization: Bearer <key>`) não garante integridade do payload
- Um atacante que intercepta a requisição pode ler a key, mas não consegue adulterar o payload sem quebrar a assinatura
- HMAC é o padrão de mercado para webhooks (Stripe, GitHub, Shopify)

### JWT interno para autenticação de webhook

**Descartada** porque:
- Adiciona complexidade desnecessária (key management, expiração, refresh)
- HMAC é mais simples e igualmente seguro para o caso de uso
- Clientes não precisam implementar validação de JWT, só HMAC-SHA256

## Consequências

### Positivas
- Isolamento: vazamento de uma secret afeta apenas um endpoint
- Rastreabilidade: cada evento é atribuível a um endpoint específico via `X-Webhook-Id`
- Rotação sem downtime: grace period de 24h permite migração transparente
- Padrão de mercado: qualquer biblioteca HTTP dos clientes tem suporte a HMAC-SHA256

### Negativas
- Complexidade adicional na implementação: lógica de grace period, duas secrets ativas por endpoint durante a janela
- Cliente precisa implementar validação do lado dele (contrapartida inevitável de qualquer scheme de assinatura)
- Geração segura de secret e armazenamento hasheado exigem cuidado na implementação (escopo da revisão de segurança pela Sofia)
