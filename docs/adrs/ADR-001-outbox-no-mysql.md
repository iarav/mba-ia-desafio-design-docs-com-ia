# ADR-001: Padrão Outbox no MySQL para publicação de eventos de webhook

**Status:** Decidido

**Data:** 2026-07-12

**Participantes:** Larissa (Tech Lead), Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Marcos (PM)

## Contexto

A feature de Webhooks de Notificação de Pedidos exige que eventos de mudança de status de pedido sejam notificados a endpoints externos dos clientes. A operação de mudança de status (`OrderService.changeStatus`) já executa múltiplas escritas dentro de uma transação SQL: atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos. A publicação do evento de webhook precisa ser atômica com essa transação — não pode haver cenário em que o status muda e o evento não é publicado, nem evento publicado para uma transação que deu rollback.

## Decisão

Usaremos o **padrão Outbox transacional no MySQL**. Quando o status de um pedido muda, dentro da mesma transação Prisma que atualiza `orders` e `order_status_history`, uma linha será inserida na tabela `webhook_outbox` com o payload completo do evento. Um worker separado fará polling dessa tabela e disparará as chamadas HTTP.

A inserção na outbox usa o `tx` (Prisma TransactionClient) já disponível no escopo da transação do `changeStatus`, sem abrir nova conexão.

## Alternativas consideradas

### Redis Streams / RabbitMQ

Usar um message broker externo para desacoplar a publicação. **Descartada** porque:
- Exigiria subir e operar infraestrutura adicional (Redis Cluster ou RabbitMQ)
- O time é pequeno e não tem capacidade operacional para mais um serviço stateful
- A complexidade adicional não se justifica para o volume esperado de eventos

### Disparo síncrono dentro da transação

Fazer a chamada HTTP diretamente no método `changeStatus`, dentro da transação. **Descartada** porque:
- A transação já é pesada (múltiplas tabelas, lock de linha)
- Um cliente lento ou offline travaria a mudança de status para todos os pedidos
- Rollback de HTTP não é possível — se o HTTP falha, o que fazer com a transação SQL?

## Consequências

### Positivas
- Atomicidade garantida: se a transação commita, o evento está persistido; se dá rollback, o evento desaparece junto
- Zero infraestrutura adicional — reusa o MySQL existente e o Prisma já configurado
- Simples de implementar com a stack atual do time

### Negativas
- A tabela `webhook_outbox` cresce com o tempo; requer estratégia de arquivamento/limpeza (fora do escopo desta fase, prevista para 30 dias)
- Polling introduz latência mínima de 2 segundos entre a mudança de status e o disparo (aceitável contra o requisito de <10s)
- Se o volume de eventos crescer muito, a tabela pode se tornar gargalo; endereçável com particionamento futuro
