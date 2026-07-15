# RFC: Sistema de Webhooks de Notificação de Pedidos

**Autor:** Bruno (Eng. Pedidos), com contribuições de Diego (Eng. Plataforma) e Sofia (Eng. Segurança)

**Status:** Proposto (em revisão)

**Data:** 2026-07-12

**Revisores:** Larissa (Tech Lead), Diego (Eng. Plataforma), Sofia (Eng. Segurança), Marcos (PM)

---

## Resumo Executivo (TL;DR)

Propomos um sistema de webhooks outbound para notificar clientes B2B sobre mudanças de status de pedidos em tempo real (latência < 10s). A solução usa padrão **Outbox transacional no MySQL** para garantir atomicidade entre a mudança de status e a publicação do evento, um **worker em processo separado com polling** para disparo das chamadas HTTP, **retry com backoff exponencial + DLQ** para resiliência, e **HMAC-SHA256 com secret por endpoint** para segurança. Nenhuma infraestrutura adicional é necessária — tudo opera sobre o MySQL e a stack Node.js/Prisma existentes. O prazo estimado é de 3 sprints.

---

## Contexto e Problema

Três clientes B2B estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — solicitaram notificações em tempo real quando o status de seus pedidos muda na plataforma. Hoje, a única forma de saber se houve mudança é via polling no `GET /orders`, o que é ineficiente e caro para os clientes. A Atlas Comercial sinalizou que pode migrar para um concorrente se a funcionalidade não for entregue até o fim do trimestre.

O sistema atual (`order-management-api`) não possui nenhum mecanismo de notificação externa, eventos, filas ou webhooks. A aplicação é uma API REST monolítica com Node.js, Express, TypeScript, Prisma e MySQL.

O requisito de latência informado pelos clientes é "abaixo de 10 segundos" entre a mudança de status e o recebimento da notificação.

---

## Proposta Técnica

### Visão geral

A feature será implementada como um novo módulo `src/modules/webhooks/` seguindo a estrutura padrão do projeto (controller, service, repository, routes, schemas). A publicação de eventos usará o padrão Outbox transacional, e o envio será feito por um worker independente.

```
┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│ POST /orders │────▶│ OrderService     │────▶│ webhook_outbox  │
│  /:id/status │     │ .changeStatus()  │     │ (mesma transação)│
└──────────────┘     └──────────────────┘     └────────┬────────┘
                                                       │ polling 2s
                                              ┌────────▼────────┐
                                              │  worker.ts      │
                                              │  (processo sep.)│
                                              └────────┬────────┘
                                                       │ HTTP POST
                                              ┌────────▼────────┐
                                              │ Cliente B2B     │
                                              │ (endpoint HTTPS)│
                                              └─────────────────┘
```

### Componentes principais

1. **Outbox (`webhook_outbox`)**: tabela MySQL onde cada evento é inserido dentro da mesma transação do `changeStatus`. Atomicidade garantida pelo Prisma `$transaction`. Payload renderizado como snapshot no momento da inserção.

2. **Worker (`src/worker.ts`)**: processo Node.js separado da API. Executa polling da `webhook_outbox` a cada 2 segundos, processa eventos pendentes em batch, dispara HTTP POST para os endpoints configurados, aplica política de retry com backoff. Conecta no mesmo banco com instância própria do `PrismaClient`.

3. **DLQ (`webhook_dead_letter`)**: tabela separada para eventos que esgotaram as 5 tentativas de retry. Reprocessamento manual via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay` (restrito a role `ADMIN`).

4. **CRUD de configuração**: endpoints REST para clientes cadastrarem, editarem, listarem e removerem seus webhooks. Cada configuração contém URL (obrigatoriamente HTTPS), secret gerada pelo servidor, lista de status a observar, e estado ativo/inativo.

5. **Segurança**: assinatura HMAC-SHA256 do payload com secret única por endpoint, enviada no header `X-Signature`. Headers adicionais: `X-Event-Id` (UUID para dedup), `X-Timestamp` (anti-replay), `X-Webhook-Id` (identificação do endpoint). Suporte a rotação de secret com grace period de 24h.

### Decisões arquiteturais

Cada decisão listada abaixo tem seu ADR dedicado com contexto completo, alternativas descartadas e consequências:

- [ADR-001](./adrs/ADR-001-outbox-no-mysql.md) — Padrão Outbox no MySQL
- [ADR-002](./adrs/ADR-002-retry-backoff-dlq.md) — Retry com backoff exponencial e DLQ
- [ADR-003](./adrs/ADR-003-hmac-sha256-autenticacao-webhook.md) — Autenticação HMAC-SHA256
- [ADR-004](./adrs/ADR-004-at-least-once-x-event-id.md) — Garantia at-least-once com X-Event-Id
- [ADR-005](./adrs/ADR-005-worker-polling-processo-separado.md) — Worker em polling com processo separado
- [ADR-006](./adrs/ADR-006-reuso-padroes-existentes.md) — Reuso dos padrões do projeto
- [ADR-007](./adrs/ADR-007-snapshot-payload-na-insercao.md) — Snapshot do payload na inserção

---

## Alternativas Consideradas

### Redis Streams / RabbitMQ como message broker

**Alternativa descartada.** Usar um broker de mensagens dedicado para desacoplar a publicação do envio foi discutido como a abordagem "canônica" para esse tipo de problema. O trade-off que levou ao descarte: exige subir e operar infraestrutura adicional (Redis Cluster ou RabbitMQ), e o time atual é pequeno, sem banda operacional para mais um serviço stateful. O padrão Outbox no MySQL resolve o mesmo problema (desacoplamento + atomicidade) sem nova infra.

### Worker dentro do mesmo processo da API

**Alternativa descartada.** Rodar o worker como parte do processo da API HTTP (ex.: `setInterval` dentro do `server.ts`) seria mais simples de implantar. O trade-off: se a API reiniciar por deploy ou crash, o worker morre junto, interrompendo o processamento de eventos. Além disso, competiria por event loop com requests HTTP. Processo separado garante ciclo de vida independente e permite escalar API e worker de forma isolada no futuro.

---

## Questões em Aberto

1. **Rate limiting de envio por cliente.** Se um cliente tiver dezenas de pedidos mudando de status em um intervalo curto, o worker pode bombardeá-lo com chamadas HTTP simultâneas. A decisão foi observar o comportamento em produção e implementar rate limiting apenas se necessário. Não há mecanismo previsto nesta fase.

2. **Notificação proativa de falhas ao cliente.** Se um webhook falhar consistentemente (ex.: 3 falhas seguidas), o cliente não será notificado por email ou outro canal. Email está fora do escopo desta fase. O cliente só descobre o problema se consultar proativamente o endpoint `GET /webhooks/:id/deliveries`. A ser reavaliado após medir o impacto em produção.

3. **Escalabilidade do worker.** Com um único worker, o throughput é limitado a eventos processados sequencialmente a cada 2 segundos. Se o volume de pedidos crescer, múltiplos workers em paralelo serão necessários, o que exige estratégia de particionamento (por `order_id`) ou lock pessimista para evitar duplicação. Isso é reconhecido como limitação, mas não será implementado nesta fase.

---

## Impacto e Riscos

### Impacto em sistemas existentes

- **`OrderService.changeStatus`**: o método ganhará uma chamada adicional dentro da transação — `publishWebhookEvent(tx, order, fromStatus, toStatus)` — que insere na `webhook_outbox`. A transação existente não muda de forma; a nova operação é uma inserção adicional
- **Banco de dados**: duas novas tabelas (`webhook_outbox`, `webhook_dead_letter`) e uma tabela de configuração (`webhook_endpoints`), mais uma tabela de histórico de entregas (`webhook_deliveries`). Migração Prisma aditiva, sem alteração em tabelas existentes
- **Infraestrutura**: novo processo (`npm run worker`) a ser incluído no deploy (PM2, Docker, ou equivalente)

### Riscos identificados

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Worker cair e eventos acumularem sem alerta | Média | Alto — clientes param de receber notificações | Health check do worker com alerta (métrica de lag da outbox). Single-worker é risco, mas aceitável para fase inicial |
| Cliente não implementar validação HMAC e aceitar payloads adulterados | Média | Alto — risco de injeção de dados falsos no sistema do cliente | Documentação no portal do desenvolvedor com exemplos de código. Responsabilidade do cliente, mas a documentação correta reduz o risco |
| Vazamento de secret em log do cliente | Baixa | Médio — um endpoint fica comprometido | Rotação de secret com grace period. Secret é por endpoint, então o impacto é contido. Logs do nosso lado nunca expõem secrets |
| Tabela outbox crescer descontroladamente | Baixa | Médio — degradação de performance do polling | Estratégia de arquivamento após 30 dias (fora do escopo, mas prevista). Índices em `status` e `created_at` garantem polling eficiente mesmo com volume |
| Divergência entre payload snapshot e estado atual do pedido | Muito baixa | Baixo — cliente vê estado antigo no evento | O snapshot é intencional; o evento reflete o estado no momento da transição. Se o cliente precisar do estado atual, usa `GET /orders/:id` |

---

## Decisões Relacionadas

- [ADR-001: Padrão Outbox no MySQL](./adrs/ADR-001-outbox-no-mysql.md)
- [ADR-002: Política de retry com backoff e DLQ](./adrs/ADR-002-retry-backoff-dlq.md)
- [ADR-003: HMAC-SHA256 com secret por endpoint](./adrs/ADR-003-hmac-sha256-autenticacao-webhook.md)
- [ADR-004: At-least-once com X-Event-Id](./adrs/ADR-004-at-least-once-x-event-id.md)
- [ADR-005: Worker em processo separado com polling](./adrs/ADR-005-worker-polling-processo-separado.md)
- [ADR-006: Reuso dos padrões existentes](./adrs/ADR-006-reuso-padroes-existentes.md)
- [ADR-007: Snapshot do payload na inserção](./adrs/ADR-007-snapshot-payload-na-insercao.md)
