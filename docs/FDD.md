# FDD: Sistema de Webhooks de Notificação de Pedidos

## Contexto e Motivação Técnica

A aplicação `order-management-api` é um monolito Node.js/Express/TypeScript com banco MySQL via Prisma. Possui módulos de autenticação, usuários, clientes, produtos e pedidos, com máquina de estados controlada para o ciclo de vida do pedido (`PENDING → PAID → PROCESSING → SHIPPED → DELIVERED`, com `CANCELLED` atingível a partir de `PENDING`, `PAID` ou `PROCESSING`).

A aplicação não possui nenhum mecanismo de notificação externa. O método `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) executa, dentro de uma transação Prisma, a atualização de `orders.status`, inserção em `order_status_history`, e ajuste de `stock_quantity`. A feature de webhooks adiciona um quarto passo nessa transação: a inserção de um evento na tabela `webhook_outbox`.

O desafio técnico central é garantir atomicidade entre a mudança de status e a publicação do evento, sem introduzir latência perceptível na API e sem exigir nova infraestrutura.

## Objetivos Técnicos

1. Publicar eventos de mudança de status de pedido com atomicidade transacional (outbox no MySQL)
2. Entregar notificações a endpoints externos com latência inferior a 10 segundos
3. Garantir entrega at-least-once com idempotência via `X-Event-Id`
4. Resiliência: retry com backoff exponencial (5 tentativas, janela de ~15h) e DLQ
5. Segurança: assinatura HMAC-SHA256 por endpoint, TLS obrigatório, rotação de secret
6. Reuso máximo dos padrões existentes: AppError, Pino, error middleware, estrutura modular

## Escopo e Exclusões

### Incluso
- CRUD de configuração de webhooks (POST, GET, PATCH, DELETE)
- Inserção de eventos na outbox dentro da transação do `changeStatus`
- Worker em processo separado com polling de 2s
- Retry com backoff exponencial (1m/5m/30m/2h/12h)
- DLQ em tabela separada com endpoint admin de replay
- Histórico de entregas (`GET /webhooks/:id/deliveries`)
- Rotação de secret com grace period de 24h

### Fora de escopo
- Notificação por email ao cliente em caso de falha (fase futura)
- Rate limiting de envio (observar e decidir depois)
- Dashboard visual para gestão de webhooks (projeto do time de frontend)
- Arquivamento automático de registros antigos da outbox (previsto para 30 dias, mas fora desta fase)
- Múltiplos workers em paralelo (single-worker nesta fase)

## Fluxos Detalhados

### 1. Criação do evento na outbox

**Trigger:** `PATCH /api/v1/orders/:id/status` é chamado com sucesso.

**Local:** `src/modules/orders/order.service.ts`, método `changeStatus`.

Dentro da transação `prisma.$transaction(async (tx) => { ... })`, após as operações existentes:

```typescript
// Dentro da transação existente em changeStatus()
await tx.order.update({ where: { id }, data: { status: to } });
await tx.orderStatusHistory.create({ /* ... */ });

// NOVO: publicar evento na outbox
await publishWebhookEvent(tx, {
  orderId: id,
  orderNumber: order.orderNumber,
  customerId: order.customerId,
  fromStatus: from,
  toStatus: to,
  totalCents: order.totalCents,
});
```

A função `publishWebhookEvent`:
1. Consulta `webhook_endpoints` ativos para o `customer_id` que tenham `toStatus` na lista de eventos filtrados (ou lista vazia = todos os status)
2. Se nenhum endpoint elegível, retorna sem inserir (economiza linha na outbox)
3. Para cada endpoint elegível, insere uma linha em `webhook_outbox` com:
   - `event_id`: UUID v4
   - `endpoint_id`: FK para `webhook_endpoints`
   - `payload`: JSON renderizado com snapshot do estado do pedido
   - `status`: `PENDING`
   - `attempt_count`: 0
   - `next_retry_at`: NOW()
4. Todas as inserções usam o `tx` (TransactionClient), garantindo atomicidade

**Payload do evento (snapshot armazenado na outbox):**

```json
{
  "event_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-12T14:30:00.000Z",
  "data": {
    "order_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
    "order_number": "ORD-000042",
    "customer_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "total_cents": 14990
  }
}
```

### 2. Processamento pelo worker

**Processo:** `src/worker.ts` (entry point separada, `npm run worker`)

**Ciclo de polling (a cada 2 segundos):**

```
loop:
  1. BEGIN TRANSACTION READ COMMITTED
  2. SELECT * FROM webhook_outbox
     WHERE status = 'PENDING'
       AND next_retry_at <= NOW()
     ORDER BY created_at ASC
     LIMIT 20
     FOR UPDATE SKIP LOCKED
  3. Para cada evento:
     a. Monta request HTTP POST com headers e payload
     b. Envia com timeout de 10s
     c. Se sucesso (2xx):
        - Atualiza status para DELIVERED
        - Insere em webhook_deliveries (success)
     d. Se falha (não-2xx, timeout, erro de rede):
        - Se attempt_count < 5:
          - Incrementa attempt_count
          - Atualiza next_retry_at com backoff
          - Mantém status PENDING
        - Se attempt_count >= 5:
          - Move para webhook_dead_letter
          - Remove da outbox (ou marca como DEAD)
  4. COMMIT
  5. Sleep 2s
```

**Headers enviados ao cliente:**

| Header | Valor | Descrição |
|--------|-------|-----------|
| `Content-Type` | `application/json` | Tipo do corpo |
| `X-Event-Id` | UUID v4 | ID único do evento para dedup |
| `X-Signature` | `sha256=<hash>` | HMAC-SHA256 do payload |
| `X-Timestamp` | ISO 8601 | Timestamp do envio |
| `X-Webhook-Id` | UUID | ID do endpoint de webhook |
| `User-Agent` | `OrderMgmt-Webhook/1.0` | Identificação do emissor |

### 3. Retry com backoff

**Política de backoff (a partir da primeira falha):**

| Tentativa | Intervalo após falha anterior | Tempo acumulado |
|-----------|-------------------------------|-----------------|
| 1ª (envio inicial) | Imediato | 0 |
| 2ª | 1 minuto | ~1 min |
| 3ª | 5 minutos | ~6 min |
| 4ª | 30 minutos | ~36 min |
| 5ª | 2 horas | ~2h 36min |
| 6ª (última) | 12 horas | ~14h 36min |

O campo `next_retry_at` é calculado como `NOW() + backoff_interval`. O worker ignora eventos cujo `next_retry_at > NOW()`.

### 4. DLQ (Dead Letter Queue)

Após a 6ª falha (5ª tentativa de retry), o worker:

1. Insere em `webhook_dead_letter`:
   - `event_id` original
   - `endpoint_id`
   - `payload` original
   - `last_error`: mensagem de erro da última tentativa
   - `last_http_status`: HTTP status code da última resposta (se houve)
   - `failed_at`: timestamp
2. Atualiza o evento na outbox para `status = DEAD` (ou remove, a depender da implementação)

**Reprocessamento (endpoint admin):**

```
POST /api/v1/admin/webhooks/dead-letter/:id/replay
Authorization: Bearer <jwt_admin>
```

Reinsere o payload na `webhook_outbox` com `status = PENDING` e `attempt_count = 0`. Requer role `ADMIN` e registra em log quem executou.

## Contratos Públicos

### 1. Criar configuração de webhook

```
POST /api/v1/customers/:customerId/webhooks
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request:**
```json
{
  "url": "https://api.cliente.com/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true
}
```

**Response 201:**
```json
{
  "id": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
  "customer_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "url": "https://api.cliente.com/webhooks/orders",
  "secret": "whsec_xK9mP2vL7nQ4wR8tY3uA6bC0dE1fG5hI",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
  "active": true,
  "created_at": "2026-07-12T14:30:00.000Z",
  "updated_at": "2026-07-12T14:30:00.000Z"
}
```

**Validações:**
- `url`: obrigatório, deve ser HTTPS (schema `https://`)
- `events`: array opcional de `OrderStatus`. Se vazio ou omitido, notifica todos os status
- `active`: opcional, default `true`
- `customerId` no path: UUID do cliente

**Erros:**
| Status | Código | Condição |
|--------|--------|----------|
| 400 | `WEBHOOK_INVALID_URL` | URL não é HTTPS |
| 400 | `VALIDATION_ERROR` | Schema inválido (Zod) |
| 404 | `NOT_FOUND` | Customer não existe |

### 2. Listar webhooks de um cliente

```
GET /api/v1/customers/:customerId/webhooks?page=1&pageSize=20
Authorization: Bearer <jwt>
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
      "customer_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
      "url": "https://api.cliente.com/webhooks/orders",
      "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED"],
      "active": true,
      "created_at": "2026-07-12T14:30:00.000Z",
      "updated_at": "2026-07-12T14:30:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

**Nota:** O campo `secret` nunca é retornado em listagens ou GET. Só é exposto no momento da criação e na rotação.

### 3. Editar configuração de webhook

```
PATCH /api/v1/webhooks/:id
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request:**
```json
{
  "url": "https://api.cliente.com/webhooks/orders-v2",
  "events": ["SHIPPED", "DELIVERED"],
  "active": false
}
```

Todos os campos são opcionais — apenas os campos enviados são alterados (partial update).

**Response 200:**
```json
{
  "id": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
  "customer_id": "c3d4e5f6-a7b8-9012-cdef-123456789012",
  "url": "https://api.cliente.com/webhooks/orders-v2",
  "events": ["SHIPPED", "DELIVERED"],
  "active": false,
  "created_at": "2026-07-12T14:30:00.000Z",
  "updated_at": "2026-07-13T09:15:00.000Z"
}
```

**Validações:**
- `url`: se informado, deve ser HTTPS (schema `https://`)
- `events`: se informado, array de `OrderStatus` válidos
- `active`: se informado, booleano

**Erros:**
| Status | Código | Condição |
|--------|--------|----------|
| 400 | `WEBHOOK_INVALID_URL` | URL informada não é HTTPS |
| 400 | `WEBHOOK_EVENTS_INVALID` | Um ou mais eventos não são `OrderStatus` válidos |
| 400 | `VALIDATION_ERROR` | Schema inválido (Zod) |
| 404 | `WEBHOOK_NOT_FOUND` | Webhook não encontrado |
| 403 | `WEBHOOK_CUSTOMER_MISMATCH` | Webhook não pertence ao cliente autenticado |

### 4. Remover configuração de webhook

```
DELETE /api/v1/webhooks/:id
Authorization: Bearer <jwt>
```

**Response 204:** No Content.

**Erros:**
| Status | Código | Condição |
|--------|--------|----------|
| 404 | `WEBHOOK_NOT_FOUND` | Webhook não encontrado |
| 403 | `WEBHOOK_CUSTOMER_MISMATCH` | Webhook não pertence ao cliente autenticado |

**Nota:** A remoção é lógica (soft delete) ou física fica a critério da implementação. Eventos já pendentes na outbox para esse endpoint continuarão sendo processados normalmente até esgotarem. Nenhum novo evento será gerado para um webhook removido.

### 5. Histórico de entregas

```
GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20
Authorization: Bearer <jwt>
```

**Response 200:**
```json
{
  "data": [
    {
      "id": "f6a7b8c9-d0e1-2345-f678-90abcdef0123",
      "webhook_id": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
      "event_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "delivery_status": "SUCCESS",
      "http_status": 200,
      "response_body": "{\"received\":true}",
      "duration_ms": 342,
      "attempt_number": 1,
      "created_at": "2026-07-12T14:30:02.000Z"
    },
    {
      "id": "a7b8c9d0-e1f2-3456-7890-abcdef012345",
      "webhook_id": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
      "event_id": "b2c3d4e5-f6a7-8901-bcde-f12345678901",
      "delivery_status": "FAILED",
      "http_status": 503,
      "response_body": null,
      "error_message": "Connection timeout after 10000ms",
      "duration_ms": 10002,
      "attempt_number": 3,
      "created_at": "2026-07-12T14:36:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 47,
    "totalPages": 3
  }
}
```

### 6. Rotação de secret

```
POST /api/v1/webhooks/:id/rotate-secret
Authorization: Bearer <jwt>
```

**Response 200:**
```json
{
  "id": "e5f6a7b8-c9d0-1234-ef56-7890abcdef01",
  "secret": "whsec_mJ3nK8pQ2rS7tV9wX4yZ5aB6cD0eF1gH",
  "previous_secret_valid_until": "2026-07-13T14:30:00.000Z",
  "message": "Previous secret remains valid for 24 hours. Update your systems before the grace period expires."
}
```

### 7. Replay de DLQ (admin)

```
POST /api/v1/admin/webhooks/dead-letter/:id/replay
Authorization: Bearer <jwt_admin>
```

**Response 200:**
```json
{
  "event_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "status": "replayed",
  "message": "Event reinserted into outbox for reprocessing"
}
```

| Status | Código | Condição |
|--------|--------|----------|
| 403 | `FORBIDDEN` | Role não é ADMIN |
| 404 | `WEBHOOK_DEAD_LETTER_NOT_FOUND` | ID não encontrado na DLQ |

## Matriz de Erros — prefixo `WEBHOOK_`

| Código | HTTP Status | Mensagem | Condição |
|--------|-------------|----------|----------|
| `WEBHOOK_NOT_FOUND` | 404 | Webhook endpoint not found | Endpoint não encontrado por ID |
| `WEBHOOK_INVALID_URL` | 400 | Webhook URL must use HTTPS | URL não começa com `https://` |
| `WEBHOOK_URL_UNREACHABLE` | 422 | Webhook URL validation failed | Tentativa de validação de URL falhou |
| `WEBHOOK_EVENTS_INVALID` | 400 | One or more event types are invalid | Status listado não é um `OrderStatus` válido |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Secret is required for webhook delivery | Configuração sem secret (não deveria ocorrer) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 413 | Webhook payload exceeds 64KB limit | Payload serializado > 65536 bytes |
| `WEBHOOK_DELIVERY_FAILED` | — (interno) | Webhook delivery failed after N attempts | Interno, dispara movimentação para DLQ |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Dead letter entry not found | ID não existe na tabela `webhook_dead_letter` |
| `WEBHOOK_ALREADY_REPLAYED` | 409 | Dead letter entry has already been replayed | Evento já foi reprocessado |
| `WEBHOOK_CUSTOMER_MISMATCH` | 403 | Webhook does not belong to this customer | Tentativa de acessar webhook de outro cliente |

## Estratégias de Resiliência

### Timeouts
- **Timeout de conexão HTTP:** 10 segundos por chamada do worker
- **Timeout de leitura:** 10 segundos (parte do mesmo timeout)
- Se o cliente não responder em 10s, considera falha e agenda retry

### Retries com backoff exponencial
- 6 tentativas totais (1 inicial + 5 retries)
- Intervalos: 1m, 5m, 30m, 2h, 12h
- Implementado via campo `next_retry_at` na tabela; worker consulta `WHERE next_retry_at <= NOW()`

### Circuit breaker (futuro)
- Não implementado nesta fase. Se um endpoint falhar consistentemente, o backoff naturalmente espaça as tentativas. Circuit breaker dedicado pode ser adicionado se o volume de eventos para endpoints problemáticos se tornar um gargalo

### Fallback
- Após esgotar retries: movimentação para DLQ (não há fallback automático para outro canal)
- DLQ preserva payload, erro e timestamp para diagnóstico

### Graceful shutdown do worker
- Worker escuta SIGINT/SIGTERM
- Completa o batch atual antes de encerrar (usa flag `shuttingDown`)
- Loga quantos eventos ficaram pendentes no batch interrompido

## Observabilidade

### Métricas
| Métrica | Tipo | Descrição |
|---------|------|-----------|
| `webhook_outbox_lag_seconds` | Gauge | Tempo do evento pendente mais antigo |
| `webhook_outbox_pending_count` | Gauge | Total de eventos com status PENDING |
| `webhook_delivery_duration_ms` | Histogram | Duração das chamadas HTTP (p50, p95, p99) |
| `webhook_delivery_total` | Counter | Total de entregas, com labels `status` (success/failed) e `attempt` |
| `webhook_dead_letter_total` | Counter | Eventos movidos para DLQ |
| `webhook_worker_cycle_duration_ms` | Histogram | Duração de cada ciclo de polling |

### Logs
- **Criação de evento na outbox**: `logger.info({ eventId, orderId, endpointCount }, 'webhook_event_published')`
- **Entrega com sucesso**: `logger.info({ eventId, endpointId, httpStatus, durationMs }, 'webhook_delivery_success')`
- **Falha de entrega**: `logger.warn({ eventId, endpointId, attemptCount, error }, 'webhook_delivery_failed')`
- **Entrada na DLQ**: `logger.error({ eventId, endpointId, attempts, lastError }, 'webhook_moved_to_dead_letter')`
- **Replay de DLQ**: `logger.warn({ eventId, replayedBy }, 'webhook_dead_letter_replayed')` — incluir `userId` do admin
- **Worker health**: `logger.info({ processedCount, failedCount, cycleDurationMs }, 'worker_cycle_completed')`

### Tracing
- O `X-Event-Id` gerado na outbox é usado como identificador de tracing ponta a ponta
- O `X-Request-Id` do middleware `request-logger.middleware.ts` já existe no fluxo da API
- Logs do worker incluem `eventId` para correlação com o evento original

## Dependências e Compatibilidade

### Dependências de módulos existentes
- `OrderService.changeStatus` — o ponto de integração principal (ver seção Integração)
- `PrismaClient` — worker instancia seu próprio, compartilhando `DATABASE_URL`
- Middleware de autenticação (`authenticate`, `requireRole`) — reutilizado nos endpoints de webhook
- Error middleware — captura `AppError`, `ZodError`, `Prisma` errors sem alterações

### Novas dependências de pacote
Nenhuma. A feature usa apenas dependências já existentes no `package.json`:
- `express` para rotas
- `@prisma/client` para banco
- `zod` para validação
- `pino` para logging
- `jsonwebtoken` + middleware existente para autenticação
- `uuid` para geração de `event_id`

### Migração de banco
Migração aditiva — novas tabelas sem alterar existentes:

- `webhook_endpoints`: configuração de webhooks por cliente
- `webhook_outbox`: fila de eventos a serem enviados
- `webhook_deliveries`: histórico de tentativas de entrega
- `webhook_dead_letter`: eventos que esgotaram retries

### Compatibilidade
- Endpoints existentes não mudam assinatura ou comportamento
- `OrderService.changeStatus` ganha um passo adicional na transação, mas o contrato público (`PATCH /orders/:id/status`) não muda
- Worker é processo separado — não afeta o runtime da API

## Critérios de Aceite Técnicos

1. `PATCH /orders/:id/status` com transição válida insere evento na `webhook_outbox` dentro da mesma transação
2. Se a transação do `changeStatus` der rollback, nenhum evento aparece na outbox
3. Worker entrega evento com sucesso em endpoint HTTPS mock em menos de 10s (medido do `changeStatus` ao recebimento)
4. Worker realiza retry com os intervalos configurados (1m, 5m, 30m, 2h, 12h)
5. Após 6 falhas, evento é movido para `webhook_dead_letter` com payload e erro preservados
6. Endpoint `POST /admin/webhooks/dead-letter/:id/replay` reinsere evento na outbox (apenas role ADMIN)
7. Criação de webhook com URL `http://...` retorna 400 `WEBHOOK_INVALID_URL`
8. Secret retornada na criação não aparece em GET nem em listagem
9. `X-Event-Id` é único por evento e consistente entre retries do mesmo evento
10. Rotação de secret: secret antiga continua válida por 24h, depois é rejeitada
11. Logs incluem `eventId` em todos os níveis (info, warn, error) para correlação
12. Cobertura de testes cobre: inserção na outbox (com e sem endpoints elegíveis), fluxo completo do worker (sucesso, falha, retry, DLQ), validação de URL HTTPS, rotação de secret, autorização ADMIN no replay

## Integração com o Sistema Existente

### 1. `src/modules/orders/order.service.ts` — método `changeStatus`

**Como se integra:** Uma chamada a `publishWebhookEvent(tx, order, fromStatus, toStatus)` é adicionada dentro da transação `prisma.$transaction`, após as operações existentes de `update order`, `create history` e ajuste de estoque. A função recebe o `tx` (TransactionClient) — mesma conexão transacional — para garantir atomicidade. Se a outbox falhar, a transação inteira dá rollback.

### 2. `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — hierarquia de erros

**Como se integra:** Os erros do módulo de webhooks estendem as classes existentes. Por exemplo, `WebhookNotFoundError` estende `NotFoundError`, `WebhookInvalidUrlError` estende `BadRequestError`. O middleware de erro centralizado (`src/middlewares/error.middleware.ts`) já trata `AppError`, `ZodError` e `Prisma` errors — nenhum código novo de error handling é necessário. O padrão de código de erro com prefixo (`WEBHOOK_*`) segue a mesma convenção de `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION` e `INACTIVE_PRODUCT`.

### 3. `src/middlewares/auth.middleware.ts` — `authenticate` e `requireRole`

**Como se integra:** As rotas de CRUD de webhook usam `authenticate` (rota autenticada normal, qualquer role). O endpoint de replay de DLQ (`POST /admin/webhooks/dead-letter/:id/replay`) usa `requireRole('ADMIN')` para restringir acesso. Ambos os middlewares são importados diretamente, sem modificação. O padrão é idêntico ao usado em `src/modules/orders/order.routes.ts` (linha 14: `router.use(authenticate)`).

### 4. `src/shared/logger/index.ts` — logger Pino

**Como se integra:** Tanto os endpoints HTTP quanto o worker importam o mesmo `logger` de `src/shared/logger/index.js`. O logger já está configurado com redação de dados sensíveis (`authorization`, `password`, `token`), nível de log via `env.LOG_LEVEL`, e formatação pretty em desenvolvimento. Os logs do worker usam o mesmo `base: { service: 'order-management-api' }`, garantindo que todas as entradas sejam correlacionáveis no mesmo sistema de agregação.

### 5. `src/middlewares/validate.middleware.ts` — validação Zod

**Como se integra:** Os schemas de validação do módulo de webhooks (`webhook.schemas.ts`) seguem o padrão Zod do projeto. O middleware `validate` é usado nas rotas da mesma forma que nos módulos existentes. Exemplo: `validate({ body: createWebhookSchema })` para POST, `validate({ params: webhookIdParamSchema })` para GET por ID. A validação de URL HTTPS é feita via `.url().refine(url => url.startsWith('https://'), ...)` no schema Zod.

### 6. `src/app.ts` — composição de dependências

**Como se integra:** O módulo de webhooks é registrado em `buildControllers()` da mesma forma que os módulos existentes: instanciação de Repository, Service e Controller, seguida de injeção no router. O `buildApiRouter()` em `src/routes/index.ts` ganha uma nova entrada: `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))`. O worker (`src/worker.ts`) não passa por `buildApp` — ele é um entry point separado que instancia seu próprio `PrismaClient` e inicia o loop de polling.

### 7. `src/server.ts` — bootstrap da aplicação

**Como se integra:** O `server.ts` continua sendo o entry point da API HTTP e não é alterado. O worker terá seu próprio entry point `src/worker.ts` com estrutura análoga (`import { prisma } from './config/database.js'`, `import { logger } from './shared/logger/index.js'`, bootstrap com graceful shutdown). O `package.json` ganha um novo script: `"worker": "tsx --env-file=.env src/worker.ts"`.

### 8. `prisma/schema.prisma` — schema do banco

**Como se integra:** Novos models são adicionados ao schema Prisma sem alterar models existentes:

- `WebhookEndpoint`: id, customerId (FK), url, secretHash, events (JSON), active, createdAt, updatedAt
- `WebhookOutbox`: id, eventId (UUID único), endpointId (FK), payload (JSON), status (enum: PENDING/PROCESSING/DELIVERED/DEAD), attemptCount, lastError, nextRetryAt, createdAt
- `WebhookDelivery`: id, outboxId (FK), endpointId (FK), eventId, deliveryStatus, httpStatus, responseBody, errorMessage, durationMs, attemptNumber, createdAt
- `WebhookDeadLetter`: id, eventId, endpointId (FK), payload (JSON), lastError, lastHttpStatus, failedAt, replayedAt, replayedBy

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Worker cair sem alerta | Média | Alto | Health check com métrica `webhook_outbox_lag_seconds`. Alerta se lag > 60s |
| Eventos duplicados em cenário de crash do worker durante batch | Média | Baixo (at-least-once é esperado) | `X-Event-Id` permite dedup do lado do cliente. Batch usa `SKIP LOCKED` para evitar duplicação entre workers |
| Aumento da latência do `changeStatus` pela inserção adicional na outbox | Baixa | Médio | Inserção é local (mesmo banco), sem I/O de rede adicional. Medir impacto em teste de carga |
| Cliente cadastrar URL maliciosa (SSRF) | Baixa | Alto | Validação de schema: apenas HTTPS. Idealmente, validação de domínio contra lista de domínios conhecidos do cliente (fase futura) |
| Colisão de eventos com mesmo `event_id` | Muito baixa | Baixo | UUID v4 tem probabilidade de colisão desprezível |
