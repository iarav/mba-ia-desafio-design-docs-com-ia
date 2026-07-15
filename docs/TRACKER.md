# Tracker de Rastreabilidade

Cada linha mapeia um item registrado nos documentos à sua origem na transcrição (`TRANSCRICAO.md`) ou no código fonte da aplicação.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-FR-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de webhook via POST /customers/:id/webhooks (URL HTTPS + eventos; secret gerada pelo servidor) | TRANSCRICAO | [09:31] Marcos, [09:23] Sofia |
| PRD-FR-02 | `docs/PRD.md` | Requisito Funcional | Listagem de webhooks do cliente via GET /customers/:id/webhooks (paginação) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | `docs/PRD.md` | Requisito Funcional | Edição de webhook via PATCH /webhooks/:id (URL, eventos, ativo/inativo) | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito Funcional | Remoção de webhook via DELETE /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito Funcional | Publicação de evento na outbox atômica com transação de changeStatus | TRANSCRICAO | [09:06] Diego, [09:40] Bruno |
| PRD-FR-06 | `docs/PRD.md` | Requisito Funcional | Worker dispara HTTP POST com payload JSON (event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents) | TRANSCRICAO | [09:43] Diego |
| PRD-FR-07 | `docs/PRD.md` | Requisito Funcional | Headers HTTP: X-Event-Id, X-Signature (HMAC-SHA256), X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego, [09:44] Sofia |
| PRD-FR-08 | `docs/PRD.md` | Requisito Funcional | Retry automático com backoff exponencial 1m/5m/30m/2h/12h (5 retries + 1ª tentativa = 6 no total) | TRANSCRICAO | [09:15] Diego, [09:17] Diego |
| PRD-FR-09 | `docs/PRD.md` | Requisito Funcional | Após 6 tentativas, evento vai para webhook_dead_letter com payload e motivo | TRANSCRICAO | [09:18] Diego |
| PRD-FR-10 | `docs/PRD.md` | Requisito Funcional | Endpoint admin POST /admin/webhooks/dead-letter/:id/replay exige role ADMIN | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| PRD-FR-11 | `docs/PRD.md` | Requisito Funcional | Histórico de entregas consultável via GET /webhooks/:id/deliveries (paginação) | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-12 | `docs/PRD.md` | Requisito Funcional | Rotação de secret via POST /webhooks/:id/rotate-secret com grace period 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-FR-13 | `docs/PRD.md` | Requisito Funcional | URL do webhook deve usar HTTPS (HTTP rejeitado com WEBHOOK_INVALID_URL) | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-01 | `docs/PRD.md` | Requisito Não Funcional | Latência de entrega < 10 segundos (p95) | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| PRD-NFR-02 | `docs/PRD.md` | Requisito Não Funcional | Atomicidade transacional — outbox na mesma transação SQL | TRANSCRICAO | [09:06] Diego, [09:40] Bruno |
| PRD-NFR-03 | `docs/PRD.md` | Requisito Não Funcional | Timeout HTTP de 10 segundos por chamada do worker | TRANSCRICAO | [09:42] Diego |
| PRD-NFR-04 | `docs/PRD.md` | Requisito Não Funcional | Payload máximo de 64KB | TRANSCRICAO | [09:24] Diego |
| PRD-NFR-05 | `docs/PRD.md` | Requisito Não Funcional | TLS obrigatório para endpoints de webhook | TRANSCRICAO | [09:23] Sofia |
| PRD-NFR-06 | `docs/PRD.md` | Requisito Não Funcional | HMAC-SHA256 sobre o corpo do request com secret por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-NFR-07 | `docs/PRD.md` | Requisito Não Funcional | Garantia at-least-once (não exactly-once) | TRANSCRICAO | [09:25] Diego |
| PRD-NFR-08 | `docs/PRD.md` | Requisito Não Funcional | Worker em processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-NFR-09 | `docs/PRD.md` | Requisito Não Funcional | Reuso dos padrões existentes (AppError, Pino, Zod, middleware) | TRANSCRICAO | [09:28] Bruno, [09:29] Larissa |
| PRD-NFR-10 | `docs/PRD.md` | Requisito Não Funcional | Rastreabilidade e auditoria — logs com eventId em todas as operações; replay DLQ registra userId | TRANSCRICAO | [09:36] Sofia (auditoria do replay: "logar quem fez o replay"); [09:25] Diego (X-Event-Id como identificador de tracing) |
| PRD-ESCOPO-01 | `docs/PRD.md` | Exclusão de Escopo | Email de notificação de falha ao cliente — adiado para fase futura | TRANSCRICAO | [09:37] Larissa |
| PRD-ESCOPO-02 | `docs/PRD.md` | Exclusão de Escopo | Rate limiting de envio por cliente — observar em produção antes de implementar | TRANSCRICAO | [09:39] Diego |
| PRD-ESCOPO-03 | `docs/PRD.md` | Exclusão de Escopo | Dashboard visual — projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESCOPO-04 | `docs/PRD.md` | Exclusão de Escopo | Garantia de ordering global de eventos — apenas por order_id com single-worker | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| PRD-ESCOPO-05 | `docs/PRD.md` | Exclusão de Escopo | Exactly-once delivery — semântica é at-least-once; dedup fica a cargo do cliente | TRANSCRICAO | [09:25] Diego, [09:25] Sofia |
| FDD-ESCOPO-01 | `docs/FDD.md` | Exclusão de Escopo | Arquivamento automático de registros antigos da outbox — previsto para 30 dias, fora desta fase | TRANSCRICAO | [09:08] Diego |
| FDD-ESCOPO-02 | `docs/FDD.md` | Exclusão de Escopo | Múltiplos workers em paralelo — single-worker nesta fase | TRANSCRICAO | [09:13] Diego |
| PRD-METRICA-01 | `docs/PRD.md` | Métrica | Latência changeStatus → recebimento cliente p95 < 10s | TRANSCRICAO | [09:02] Marcos |
| PRD-METRICA-02 | `docs/PRD.md` | Métrica | 80% de clientes B2B com webhook ativo em 3 meses | TRANSCRICAO | [09:00] Marcos |
| PRD-RISCO-01 | `docs/PRD.md` | Risco | Cliente não adotar webhooks — mitigação: documentação e onboarding proativo | TRANSCRICAO | [09:00] Marcos, [09:26] Marcos |
| PRD-RISCO-02 | `docs/PRD.md` | Risco | Worker cair e eventos acumularem — mitigação: health check e alerta | TRANSCRICAO | [09:11] Diego |
| PRD-RISCO-03 | `docs/PRD.md` | Risco | Atlas Comercial migrar para concorrente — mitigação: prazo 3 sprints | TRANSCRICAO | [09:00] Marcos |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa Descartada | Redis Streams / RabbitMQ como message broker — descartado por custo de infra adicional | TRANSCRICAO | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa Descartada | Worker dentro do mesmo processo da API — descartado por acoplamento de ciclo de vida | TRANSCRICAO | [09:11] Diego |
| RFC-ABERTO-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting de envio por cliente — observar e decidir depois | TRANSCRICAO | [09:39] Diego |
| RFC-ABERTO-02 | `docs/RFC.md` | Questão em Aberto | Notificação proativa de falha ao cliente — email fora do escopo | TRANSCRICAO | [09:37] Marcos, [09:37] Larissa |
| RFC-ABERTO-03 | `docs/RFC.md` | Questão em Aberto | Escalabilidade do worker com múltiplos workers em paralelo | TRANSCRICAO | [09:13] Diego |
| RFC-RISCO-01 | `docs/RFC.md` | Risco | Worker cair sem alerta — mitigação: health check, métrica de lag da outbox | TRANSCRICAO | [09:11] Diego |
| RFC-RISCO-02 | `docs/RFC.md` | Risco | Vazamento de secret em log do cliente — mitigação: rotação com grace period, secret hasheada | TRANSCRICAO | [09:22] Diego |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | POST /customers/:customerId/webhooks — criar configuração de webhook | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | GET /customers/:customerId/webhooks — listar webhooks do cliente | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | GET /webhooks/:id/deliveries — histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | POST /webhooks/:id/rotate-secret — rotação de secret | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | POST /admin/webhooks/dead-letter/:id/replay — replay de DLQ (admin) | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| FDD-ERRO-01 | `docs/FDD.md` | Código de Erro | WEBHOOK_NOT_FOUND — endpoint não encontrado | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | `docs/FDD.md` | Código de Erro | WEBHOOK_INVALID_URL — URL não é HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | `docs/FDD.md` | Código de Erro | WEBHOOK_PAYLOAD_TOO_LARGE — payload > 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-04 | `docs/FDD.md` | Código de Erro | WEBHOOK_DEAD_LETTER_NOT_FOUND — entrada DLQ não encontrada | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Inserção na outbox via publishWebhookEvent(tx, order, from, to) na transação do changeStatus | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Worker polling a cada 2s, batch de 20, SKIP LOCKED | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Backoff: 1m, 5m, 30m, 2h, 12h via campo next_retry_at | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | DLQ: move para webhook_dead_letter após 5 falhas, replay via endpoint admin | TRANSCRICAO | [09:18] Diego |
| FDD-INTEGRACAO-01 | `docs/FDD.md` | Integração | src/modules/orders/order.service.ts — changeStatus ganha chamada a publishWebhookEvent na transação | CODIGO | `src/modules/orders/order.service.ts:126-179` |
| FDD-INTEGRACAO-02 | `docs/FDD.md` | Integração | src/shared/errors/app-error.ts — erros do módulo estendem AppError com prefixo WEBHOOK_ | CODIGO | `src/shared/errors/app-error.ts:3-16` |
| FDD-INTEGRACAO-03 | `docs/FDD.md` | Integração | src/middlewares/auth.middleware.ts — authenticate e requireRole('ADMIN') reutilizados | CODIGO | `src/middlewares/auth.middleware.ts:27-61` |
| FDD-INTEGRACAO-04 | `docs/FDD.md` | Integração | src/shared/logger/index.ts — logger Pino reutilizado no worker e endpoints | CODIGO | `src/shared/logger/index.ts:13-30` |
| FDD-INTEGRACAO-05 | `docs/FDD.md` | Integração | src/middlewares/error.middleware.ts — captura AppError, ZodError e Prisma sem alterações | CODIGO | `src/middlewares/error.middleware.ts:14-65` |
| FDD-INTEGRACAO-06 | `docs/FDD.md` | Integração | src/middlewares/validate.middleware.ts — schemas Zod para validação de endpoints de webhook | CODIGO | `src/middlewares/validate.middleware.ts:11-37` |
| FDD-INTEGRACAO-07 | `docs/FDD.md` | Integração | src/app.ts — buildControllers registra WebhookRepository/Service/Controller | CODIGO | `src/app.ts:26-53` |
| FDD-INTEGRACAO-08 | `docs/FDD.md` | Integração | prisma/schema.prisma — novos models (WebhookEndpoint, WebhookOutbox, WebhookDelivery, WebhookDeadLetter) adicionados sem alterar existentes | CODIGO | `prisma/schema.prisma` |
| FDD-OBS-01 | `docs/FDD.md` | Observabilidade | Métrica webhook_outbox_lag_seconds — tempo do evento pendente mais antigo | EDITORIAL | Nome da métrica é inferido; a necessidade de monitorar lag da outbox decorre de [09:08] Diego (índices/arquivamento) e [09:11] Diego (worker cair e eventos acumularem) |
| FDD-OBS-02 | `docs/FDD.md` | Observabilidade | Logs com eventId em todos os níveis para correlação | EDITORIAL | Síntese a partir de [09:29] Bruno (reuso do logger Pino) e [09:36] Sofia (log de auditoria no replay da DLQ). A exigência de eventId em todos os níveis é inferida, mas a correlação via eventId é discutida em [09:25] Diego (X-Event-Id) |
| FDD-OBS-03 | `docs/FDD.md` | Observabilidade | Métrica webhook_delivery_duration_ms (p50, p95, p99) | EDITORIAL | Nome da métrica é inferido; a necessidade de medir latência decorre de [09:02] Marcos (requisito <10s) e [09:42] Diego (timeout HTTP de 10s) |
| FDD-RESIL-01 | `docs/FDD.md` | Resiliência | Timeout HTTP de 10s por chamada | TRANSCRICAO | [09:42] Diego |
| FDD-RESIL-02 | `docs/FDD.md` | Resiliência | Graceful shutdown do worker (completa batch atual) | EDITORIAL | Não discutido explicitamente na transcrição. Decorre da decisão de worker em processo separado em [09:11] Diego e é boa prática de ciclo de vida de processo |
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Padrão Outbox no MySQL com transação atômica | TRANSCRICAO | [09:06] Diego, [09:08] Larissa |
| ADR-002 | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Decisão | 5 retries com backoff 1m/5m/30m/2h/12h + DLQ em tabela separada | TRANSCRICAO | [09:17] Diego, [09:18] Diego |
| ADR-003 | `docs/adrs/ADR-003-hmac-sha256-autenticacao-webhook.md` | Decisão | HMAC-SHA256 com secret única por endpoint e rotação com grace period 24h | TRANSCRICAO | [09:20] Sofia, [09:22] Sofia |
| ADR-004 | `docs/adrs/ADR-004-at-least-once-x-event-id.md` | Decisão | Garantia at-least-once com X-Event-Id UUID para dedup do cliente | TRANSCRICAO | [09:25] Diego, [09:26] Larissa |
| ADR-005 | `docs/adrs/ADR-005-worker-polling-processo-separado.md` | Decisão | Worker em processo separado com polling de 2s, entry point src/worker.ts | TRANSCRICAO | [09:09] Diego, [09:11] Diego, [09:11] Larissa |
| ADR-006 | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Decisão | Reuso de AppError, Pino, error middleware, estrutura modular, Zod, paginação, UUID | TRANSCRICAO | [09:28] Bruno, [09:29] Larissa, [09:30] Larissa |
| ADR-007 | `docs/adrs/ADR-007-snapshot-payload-na-insercao.md` | Decisão | Payload renderizado e armazenado como snapshot na inserção da outbox | TRANSCRICAO | [09:52] Larissa, [09:52] Diego, [09:52] Bruno |
| ADR-001-ALT | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa Descartada | Redis Streams descartado — custo de infra adicional para time pequeno | TRANSCRICAO | [09:07] Larissa, [09:07] Diego |
| ADR-002-ALT | `docs/adrs/ADR-002-retry-backoff-dlq.md` | Alternativa Descartada | 3 tentativas — descartado por não cobrir janela de manutenção de 2h | TRANSCRICAO | [09:16] Bruno, [09:16] Diego |
| ADR-003-ALT | `docs/adrs/ADR-003-hmac-sha256-autenticacao-webhook.md` | Alternativa Descartada | Secret global única — descartado por risco de vazamento comprometer todos os clientes | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| ADR-005-ALT | `docs/adrs/ADR-005-worker-polling-processo-separado.md` | Alternativa Descartada | MySQL triggers — descartado por falta de mecanismo nativo de notificação externa | TRANSCRICAO | [09:09] Bruno, [09:09] Diego |
| ADR-006-REF | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Referência ao Código | Ordem de módulos (controller, service, repository, routes, schemas) como padrão para webhooks | CODIGO | `src/modules/orders/` |
| ADR-006-REF-2 | `docs/adrs/ADR-006-reuso-padroes-existentes.md` | Referência ao Código | changeStatus com prisma.$transaction como modelo de transação para outbox | CODIGO | `src/modules/orders/order.service.ts:131-178` |
| ADR-007-ALT | `docs/adrs/ADR-007-snapshot-payload-na-insercao.md` | Alternativa Descartada | Payload montado no envio (lazy) — descartado por risco de refletir estado desatualizado | TRANSCRICAO | [09:51] Bruno |
| GERAL-DEC-01 | `docs/PRD.md`, `docs/RFC.md`, `docs/adrs/` | Decisão | Worker em polling, 2s, latência mínima 2s no pior caso | TRANSCRICAO | [09:10] Larissa |
| GERAL-DEC-02 | `docs/RFC.md`, `docs/FDD.md` | Decisão | Formato do payload: JSON enxuto, sem itens, com order_number e total_cents | TRANSCRICAO | [09:43] Diego, [09:44] Bruno |
| GERAL-DEC-03 | `docs/RFC.md`, `docs/FDD.md` | Decisão | Headers HTTP: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego, [09:44] Sofia |
| GERAL-DEC-04 | `docs/PRD.md`, `docs/RFC.md` | Decisão | Endpoint admin de replay DLQ restrito a role ADMIN | TRANSCRICAO | [09:36] Sofia |
| GERAL-DEC-05 | `docs/FDD.md`, `docs/adrs/ADR-007-*.md` | Decisão | Filtro de eventos na inserção da outbox (não no envio) — economiza linhas | TRANSCRICAO | [09:34] Bruno |
| GERAL-DEC-06 | `docs/FDD.md` | Decisão | UUID para IDs (segue padrão do projeto), confirmado para outbox | TRANSCRICAO | [09:51] Larissa |
| GERAL-DEC-07 | `docs/PRD.md`, `docs/RFC.md` | Trade-off | Prazo de 3 sprints (~6 semanas) incluindo revisão de segurança da Sofia | TRANSCRICAO | [09:46] Larissa, [09:46] Sofia |
| GERAL-DEC-08 | `docs/FDD.md` | Decisão | Função publishWebhookEvent(tx, order, from, to) pura, sem injetar repositório inteiro | TRANSCRICAO | [09:41] Bruno, [09:41] Diego |
| GERAL-COD-01 | `docs/FDD.md` | Referência ao Código | Estrutura modular: src/modules/ com controller, service, repository, routes, schemas | CODIGO | `src/modules/orders/` |
| GERAL-COD-02 | `docs/FDD.md` | Referência ao Código | Máquina de estados: canTransition, isTerminal, shouldDebitStock, shouldReplenishStock | CODIGO | `src/modules/orders/order.status.ts:1-38` |
| GERAL-COD-03 | `docs/FDD.md` | Referência ao Código | Classes de erro: InsufficientStockError, InvalidStatusTransitionError estendem ConflictError/UnprocessableEntityError | CODIGO | `src/shared/errors/http-errors.ts:45-63` |
| GERAL-COD-04 | `docs/FDD.md` | Referência ao Código | Paginação padronizada: paginated() helper, PaginatedResponse<T> | CODIGO | `src/shared/http/response.ts:1-24` |
| PRD-AC-01 | `docs/PRD.md` | Critério de Aceitação | Cliente consegue CRUD de configurações de webhook via API | TRANSCRICAO | [09:31] Marcos, [09:33] Bruno |
| PRD-AC-02 | `docs/PRD.md` | Critério de Aceitação | Evento com payload (event_id, order_id, from/to_status, timestamp) enviado ao endpoint | TRANSCRICAO | [09:43] Diego |
| PRD-AC-03 | `docs/PRD.md` | Critério de Aceitação | 5 retries com backoff 1m/5m/30m/2h/12h (6 tentativas no total) | TRANSCRICAO | [09:17] Diego |
| PRD-AC-04 | `docs/PRD.md` | Critério de Aceitação | Após esgotar tentativas, evento na DLQ reprocessável por admin | TRANSCRICAO | [09:18] Diego |
| PRD-AC-05 | `docs/PRD.md` | Critério de Aceitação | Assinatura HMAC-SHA256 no header X-Signature | TRANSCRICAO | [09:20] Sofia |
| PRD-AC-06 | `docs/PRD.md` | Critério de Aceitação | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-AC-07 | `docs/PRD.md` | Critério de Aceitação | URLs HTTP rejeitadas com erro WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| PRD-AC-08 | `docs/PRD.md` | Critério de Aceitação | Histórico de entregas com status sucesso/falha e detalhes | TRANSCRICAO | [09:34] Marcos |
| FDD-AC-01 | `docs/FDD.md` | Critério de Aceite Técnico | PATCH /orders/:id/status insere evento na outbox na mesma transação | TRANSCRICAO | [09:40] Bruno |
| FDD-AC-02 | `docs/FDD.md` | Critério de Aceite Técnico | Rollback da transação não deixa evento na outbox | TRANSCRICAO | [09:06] Diego |
| FDD-AC-03 | `docs/FDD.md` | Critério de Aceite Técnico | Worker entrega evento em < 10s (medido do changeStatus ao recebimento) | TRANSCRICAO | [09:02] Marcos |
| FDD-AC-04 | `docs/FDD.md` | Critério de Aceite Técnico | Worker realiza retry com intervalos configurados (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| FDD-AC-05 | `docs/FDD.md` | Critério de Aceite Técnico | Após 6 falhas, evento movido para DLQ com payload e erro preservados | TRANSCRICAO | [09:18] Diego |
| FDD-AC-06 | `docs/FDD.md` | Critério de Aceite Técnico | Endpoint admin replay reinsere evento na outbox (apenas role ADMIN) | TRANSCRICAO | [09:36] Sofia |
| FDD-AC-07 | `docs/FDD.md` | Critério de Aceite Técnico | URL http:// retorna 400 WEBHOOK_INVALID_URL | TRANSCRICAO | [09:23] Sofia |
| FDD-AC-08 | `docs/FDD.md` | Critério de Aceite Técnico | Secret não aparece em GET nem listagem | TRANSCRICAO | [09:22] Diego |
| FDD-AC-09 | `docs/FDD.md` | Critério de Aceite Técnico | X-Event-Id único por evento e consistente entre retries | TRANSCRICAO | [09:25] Diego |
| FDD-AC-10 | `docs/FDD.md` | Critério de Aceite Técnico | Rotação de secret: antiga válida por 24h, rejeitada depois | TRANSCRICAO | [09:21] Sofia |
| FDD-AC-11 | `docs/FDD.md` | Critério de Aceite Técnico | Logs incluem eventId em todos os níveis (info, warn, error) | TRANSCRICAO | [09:29] Bruno |
| FDD-AC-12 | `docs/FDD.md` | Critério de Aceite Técnico | Cobertura de testes cobre: outbox, worker (sucesso/falha/retry/DLQ), validação HTTPS, rotação secret, autorização ADMIN | TRANSCRICAO | [09:48] Larissa |
