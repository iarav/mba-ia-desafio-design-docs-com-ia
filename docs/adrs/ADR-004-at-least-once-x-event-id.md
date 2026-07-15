# ADR-004: Garantia at-least-once com X-Event-Id para idempotência

**Status:** Decidido

**Data:** 2026-07-12

**Participantes:** Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Sofia (Eng. Segurança), Marcos (PM)

## Contexto

O sistema precisa definir a semântica de entrega dos webhooks. Em sistemas distribuídos, há três garantias clássicas: at-most-once (pode perder evento), at-least-once (pode entregar duplicado) e exactly-once (entrega única garantida). A escolha impacta tanto a complexidade da implementação quanto o contrato com os clientes. É necessário que o cliente consiga diferenciar eventos duplicados de eventos genuinamente novos.

## Decisão

Adotaremos **at-least-once** com **idempotência via header `X-Event-Id`**.

- Cada evento, ao ser inserido na `webhook_outbox`, recebe um `event_id` UUID v4
- Esse UUID é enviado no header `X-Event-Id` em cada tentativa de entrega
- O cliente é responsável por deduplicar eventos com mesmo `X-Event-Id`
- A garantia é at-least-once: o evento será entregue no mínimo uma vez, mas pode ser entregue mais de uma vez em cenários de retry ou falha de rede

A duplicação pode ocorrer se:
- O worker envia o HTTP, o cliente processa, mas a resposta não chega ao worker (timeout de rede)
- No retry, o mesmo evento é reenviado com o mesmo `X-Event-Id`

## Alternativas consideradas

### Exactly-once

**Descartada** porque:
- Exigiria protocolo de two-phase commit ou idempotency-key storage compartilhado entre nós e o cliente
- Complexidade de implementação muito alta para o benefício marginal
- Nenhum cliente solicitou essa garantia; o requisito é "receber notificação", não "receber exatamente uma vez"

### At-most-once (sem retry)

**Descartada** porque:
- Contraria o propósito da feature — se o cliente estiver offline, perde o evento
- Clientes B2B como Atlas Comercial dependem dessas notificações para integração operacional
- A existência do mecanismo de retry e DLQ pressupõe que entrega é importante

### Dedup via banco de dados do lado do servidor

O servidor manteria registro do que já foi entregue e evitaria reenviar. **Descartada** porque:
- Não resolve o caso de o cliente processar mas o ACK não chegar
- Adiciona estado distribuído sem ganho real de garantia
- O padrão de mercado consolidado (Stripe, GitHub, Shopify) é at-least-once + event_id

## Consequências

### Positivas
- Simplicidade de implementação: worker não precisa coordenar com cliente para confirmar entrega
- Padrão de mercado documentado e compreendido por times de integração
- Clientes com múltiplos endpoints conseguem correlacionar eventos pelo `X-Event-Id`
- O UUID na outbox também serve como identificador para logging e tracing interno

### Negativas
- Responsabilidade de dedup é transferida para o cliente
- Requer documentação clara no portal do desenvolvedor sobre o contrato at-least-once
- Clientes que não implementarem dedup corretamente podem processar o mesmo evento duas vezes
- Em cenário de múltiplos workers no futuro, duplicação se torna mais provável
