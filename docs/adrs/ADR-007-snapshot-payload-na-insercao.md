# ADR-007: Snapshot do payload no momento da inserção na outbox

**Status:** Decidido

**Data:** 2026-07-12

**Participantes:** Bruno (Eng. Pedidos), Larissa (Tech Lead), Diego (Eng. Plataforma)

## Contexto

Quando um evento de mudança de status é registrado na `webhook_outbox`, há duas abordagens possíveis para o payload que será enviado ao cliente: armazenar apenas referências (`order_id`, `from_status`, `to_status`) e montar o payload no momento do envio, ou armazenar o payload completo já renderizado no momento da inserção.

A diferença é significativa: entre a inserção na outbox e o envio efetivo (que pode ocorrer horas depois, devido a retries), o pedido pode sofrer novas mudanças de status. Se o payload for montado no envio, ele refletirá o estado atual do pedido, não o estado no momento em que o evento ocorreu.

## Decisão

O **payload completo será renderizado e armazenado como snapshot no momento da inserção** na `webhook_outbox`.

- A coluna `payload` da tabela `webhook_outbox` armazenará o JSON completo que será enviado ao cliente
- O payload contém: `event_id`, `event_type` ("order.status_changed"), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`
- Itens do pedido não são incluídos no payload para mantê-lo enxuto; o cliente pode consultar detalhes via `GET /orders/:id`
- A função `publishWebhookEvent(tx, order, fromStatus, toStatus)` recebe o `tx` da transação ativa e o estado do pedido naquele momento, e gera o payload antes de inserir

## Alternativas consideradas

### Payload montado no momento do envio (lazy rendering)

Armazenar apenas `order_id`, `from_status`, `to_status` na outbox e consultar a tabela `orders` no momento do envio HTTP. **Descartada** porque:
- Se o pedido mudar de status novamente entre a inserção e o envio (ex.: retry após 2h), o evento notificaria o status errado
- Exemplo: evento de PENDING→PAID entra na outbox, mas antes do envio o pedido já foi para PROCESSING. O cliente receberia um evento dizendo que o status atual é PROCESSING, sem saber que houve uma transição anterior
- Viola o propósito do evento como registro imutável do que aconteceu naquele momento

### Payload com dados completos incluindo itens do pedido

**Descartada** porque:
- Inflaria o payload (potencialmente dezenas de itens por pedido)
- O limite de 64KB poderia ser atingido em pedidos grandes
- O cliente pode obter detalhes sob demanda via `GET /orders/:id`
- O evento é uma notificação, não um dump completo do pedido

## Consequências

### Positivas
- Imutabilidade: cada evento reflete exatamente o estado do pedido no momento da transição
- Consistência temporal: mesmo que o pedido mude de status múltiplas vezes, cada evento individual preserva o contexto da sua transição
- Debugging simplificado: o payload na outbox é exatamente o que foi (ou será) enviado; sem surpresas de estado mutável
- Desacoplamento: o worker não precisa consultar `orders` nem `customers` para montar o payload — tudo que precisa está na linha da outbox

### Negativas
- Armazenamento duplicado: cada evento armazena `order_number`, `customer_id`, `total_cents` que já existem na tabela `orders`
- Se o schema do payload evoluir (ex.: adicionar novo campo), eventos antigos na outbox terão o formato antigo — o worker precisa ser backwards-compatible ou limpar a outbox antes de deploy
- Ocupa mais espaço em disco comparado a armazenar só referências; mitigado pelo arquivamento de registros após 30 dias
