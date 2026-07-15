# PRD: Sistema de Webhooks de Notificação de Pedidos

## Resumo e Contexto da Feature

O Order Management System (OMS) atende clientes B2B que dependem de informações atualizadas sobre o status de seus pedidos para operar suas integrações. Atualmente, a única forma de obter essas atualizações é via polling no endpoint `GET /orders`, obrigando os clientes a fazer chamadas repetidas para detectar mudanças. Três clientes estratégicos — Atlas Comercial, MaxDistribuição e Nova Cargo — formalizaram o pedido de notificações em tempo real. A Atlas Comercial indicou que a ausência dessa funcionalidade até o fim do trimestre pode resultar em migração para um concorrente.

Esta feature introduz um sistema de webhooks outbound: quando o status de um pedido muda, o OMS notifica proativamente os endpoints configurados pelos clientes via HTTP POST com payload JSON assinado.

## Problema e Motivação

**Problema:** Clientes B2B não têm visibilidade em tempo real das mudanças de estado dos seus pedidos. O modelo atual de polling é:
- **Ineficiente:** cada cliente faz dezenas ou centenas de chamadas diárias ao `GET /orders`, a maioria retornando "sem mudanças"
- **Caro para o cliente:** consumo de banda e processamento para polling constante
- **Lento:** o cliente só descobre uma mudança no próximo ciclo de polling

**Motivação de negócio:** Reter clientes B2B estratégicos, reduzir a carga de polling na API, e posicionar a plataforma como competitiva em capacidades de integração.

## Público-alvo e Cenários de Uso

### Público-alvo primário
- **Clientes B2B integradores** (ex.: Atlas Comercial, MaxDistribuição, Nova Cargo): times técnicos que operam integrações com o OMS e precisam automatizar fluxos baseados em status de pedidos
- **Operadores internos:** usuários do OMS que configuram webhooks em nome dos clientes

### Cenários de uso
1. **Notificação de pedido pago:** Cliente quer disparar processo de separação no armazém assim que o pedido for confirmado (status `PAID`)
2. **Notificação de envio:** Cliente quer atualizar o tracking no seu próprio sistema quando o pedido for despachado (status `SHIPPED`)
3. **Notificação de entrega:** Cliente quer disparar pesquisa de satisfação quando pedido for entregue (status `DELIVERED`)
4. **Notificação seletiva:** Cliente quer receber apenas eventos de `SHIPPED` e `DELIVERED`, ignorando transições intermediárias
5. **Recuperação de falhas:** Cliente ficou offline por manutenção e quer receber os eventos que perdeu sem precisar reprocessar manualmente

## Objetivos e Métricas de Sucesso

| Objetivo | Métrica | Meta | Forma de medição |
|----------|---------|------|------------------|
| Reduzir latência de percepção de mudança de status pelo cliente | Tempo entre `changeStatus` e recebimento pelo cliente (p95) | < 10 segundos | Timestamp do evento vs. timestamp de recebimento no lado do cliente (log do worker) |
| Migrar clientes do polling para webhooks | % de clientes B2B ativos com pelo menos 1 webhook configurado | 80% em 3 meses | Query no banco: `customer_id` distintos com webhook ativo / total de clientes B2B |
| Garantir confiabilidade das notificações | Taxa de entrega bem-sucedida (excluindo DLQ) | > 99,5% | `webhook_delivery_success / (success + failed)` nos últimos 30 dias |
| Reduzir carga de polling na API | Redução de chamadas `GET /orders` por cliente migrado | > 90% de redução | Comparação de volume de requests por cliente antes/depois da adoção de webhooks |

## Escopo

### Incluso (v1)

A tabela detalhada dos requisitos funcionais está na seção [Requisitos Funcionais](#requisitos-funcionais); esta lista é o resumo do escopo desta fase, com os mesmos IDs.

- **FR-01 a FR-04:** CRUD de configuração de webhooks — cliente pode criar (com `customer_id` no path, filtro de eventos por status, secret gerada pelo servidor), listar, editar e remover endpoints
- **FR-05:** Publicação de evento de mudança de status na outbox, atômica com a transação do `changeStatus`
- **FR-06 e FR-07:** Envio HTTP POST para endpoints configurados, com payload JSON e headers padronizados (`X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`)
- **FR-08:** Retry automático com backoff exponencial (5 retries após a 1ª tentativa; 6 tentativas no total)
- **FR-09 e FR-10:** Dead Letter Queue com endpoint admin de reprocessamento manual (role `ADMIN`)
- **FR-11:** Histórico de entregas consultável por cliente (`GET /webhooks/:id/deliveries`)
- **FR-12:** Rotação de secret com grace period de 24 horas
- **FR-13:** URL do webhook obrigatoriamente HTTPS (assinatura HMAC-SHA256 detalhada em `NFR-06`)

### Fora de escopo (v1)
- **Notificação por email ao cliente em caso de falha consecutiva.** Discutido na reunião e adiado para fase futura. Será reavaliado após termos dados de volume de falhas em produção.
- **Rate limiting de envio por cliente.** O time optou por observar o comportamento em produção antes de implementar. Se um cliente receber volume excessivo de chamadas em curto intervalo, rate limiting será priorizado na sprint seguinte.
- **Dashboard visual para gestão de webhooks.** Projeto separado do time de frontend. A v1 oferece apenas os endpoints de API.
- **Garantia de ordering global de eventos.** Garantido apenas por `order_id` e enquanto houver single-worker. Ordering global entre pedidos diferentes não é garantida.
- **Exactly-once delivery.** A semântica é at-least-once. Dedup fica a cargo do cliente via `X-Event-Id`.

## Requisitos Funcionais

| ID | Requisito | Origem |
|----|-----------|--------|
| FR-01 | Cliente pode cadastrar um webhook via `POST /customers/:id/webhooks` informando URL (HTTPS obrigatório) e eventos desejados. Secret é gerada pelo servidor e devolvida na resposta | [09:31] Marcos, [09:23] Sofia |
| FR-02 | Cliente pode listar seus webhooks via `GET /customers/:id/webhooks` com paginação | [09:33] Bruno |
| FR-03 | Cliente pode editar um webhook via `PATCH /webhooks/:id` (alterar URL, eventos, ativo/inativo) | [09:33] Bruno |
| FR-04 | Cliente pode remover um webhook via `DELETE /webhooks/:id` | [09:33] Bruno |
| FR-05 | Quando o status de um pedido muda, um evento é inserido na `webhook_outbox` na mesma transação SQL | [09:06] Diego, [09:40] Bruno |
| FR-06 | Worker dispara HTTP POST para cada endpoint elegível com payload JSON contendo `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | [09:43] Diego |
| FR-07 | Headers HTTP enviados: `X-Event-Id`, `X-Signature` (HMAC-SHA256), `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` | [09:44] Diego, [09:44] Sofia |
| FR-08 | Em caso de falha, worker agenda retry com backoff exponencial: 1m, 5m, 30m, 2h, 12h (5 retries após a 1ª tentativa; 6 tentativas no total) | [09:17] Diego |
| FR-09 | Após esgotadas as 6 tentativas, evento é movido para tabela `webhook_dead_letter` com payload e motivo da falha | [09:18] Diego |
| FR-10 | Admin pode reprocessar evento da DLQ via `POST /admin/webhooks/dead-letter/:id/replay` (restrito a role ADMIN) | [09:18] Diego, [09:36] Sofia |
| FR-11 | Cliente pode consultar histórico de entregas via `GET /webhooks/:id/deliveries` com paginação | [09:34] Marcos |
| FR-12 | Cliente pode rotacionar a secret do webhook via `POST /webhooks/:id/rotate-secret`. Secret antiga permanece válida por 24h | [09:21] Sofia |
| FR-13 | URL do webhook deve usar HTTPS; URLs HTTP são rejeitadas na criação com erro de validação | [09:23] Sofia |

## Requisitos Não Funcionais

| ID | Requisito | Detalhe | Origem |
|----|-----------|---------|--------|
| NFR-01 | Latência de entrega < 10 segundos (p95) | Medido entre inserção na outbox e resposta HTTP do cliente | [09:02] Marcos, [09:10] Larissa |
| NFR-02 | Atomicidade transacional | Inserção do evento na outbox é na mesma transação SQL da mudança de status | [09:06] Diego, [09:40] Bruno |
| NFR-03 | Timeout de chamada HTTP de 10 segundos | Cliente que não responde em 10s é tratado como falha | [09:42] Diego |
| NFR-04 | Payload máximo de 64KB | Eventos que excederem geram erro `WEBHOOK_PAYLOAD_TOO_LARGE` e não são enviados | [09:24] Diego |
| NFR-05 | TLS obrigatório para endpoints de webhook | Validação no schema Zod; HTTP é rejeitado | [09:23] Sofia |
| NFR-06 | HMAC-SHA256 sobre o corpo do request | Chave secreta por endpoint, armazenada com hash | [09:20] Sofia |
| NFR-07 | Garantia at-least-once | Cliente deve deduplicar via `X-Event-Id` | [09:25] Diego |
| NFR-08 | Worker em processo separado da API | Ciclo de vida independente; não compartilha event loop com a API | [09:11] Diego |
| NFR-09 | Reuso dos padrões existentes do projeto | AppError, Pino, error middleware, estrutura modular, Zod, Prisma | [09:28] Bruno, [09:29] Larissa |
| NFR-10 | Rastreabilidade e auditoria | Logs com `eventId` em todas as operações; replay de DLQ registra `userId` | [09:29] Bruno, [09:36] Sofia |

## Decisões e Trade-offs Principais

| Decisão | Escolha | Trade-off |
|---------|---------|-----------|
| Mecanismo de publicação | Outbox no MySQL | Simplicidade e atomicidade vs. latência de polling (2s) |
| Message broker | Não usar (Redis/RabbitMQ descartados) | Sem infra adicional vs. menos reativo que um broker |
| Semântica de entrega | At-least-once | Simplicidade de implementação vs. cliente precisa implementar dedup |
| Segurança | HMAC-SHA256 com secret por endpoint | Segurança robusta vs. complexidade de rotação e grace period |
| Execução do worker | Processo separado com polling | Desacoplamento vs. single point of failure |
| Isolamento de falhas | DLQ em tabela separada | Performance de leitura da outbox vs. duas tabelas para gerenciar |

## Dependências

- **Banco MySQL existente:** a feature não introduz novo banco ou tecnologia de persistência
- **Prisma ORM:** schema adicional, mesma instância de banco
- **Middleware de autenticação existente:** `authenticate` e `requireRole` de `src/middlewares/auth.middleware.ts`
- **Logger Pino existente:** mesmo logger, mesmos níveis e redação de dados sensíveis
- **Infraestrutura de deploy:** novo processo `worker` precisa ser incluído no deploy (PM2, Docker, Kubernetes, ou equivalente)
- **Time de segurança:** revisão de código pela Sofia antes do deploy (2 dias úteis reservados)

## Riscos e Mitigação

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Cliente não adotar webhooks e continuar no polling | Média | Alto — feature não entrega valor se não for usada | Documentação no portal do desenvolvedor com exemplos de código. Onboarding proativo com os 3 clientes que pediram a feature. Métrica de adoção como critério de sucesso |
| Worker cair e eventos acumularem sem que o time perceba | Média | Alto — clientes param de receber notificações | Health check com alerta. Métrica `webhook_outbox_lag_seconds` com threshold de alerta em 60s. Procedimento operacional documentado para restart do worker |
| Atlas Comercial migrar para concorrente antes da entrega | Baixa | Crítico — perda de cliente estratégico | Prazo de 3 sprints (~6 semanas) acordado com Marcos para comunicação aos clientes. Entrega faseada: MVP funcional na sprint 2 para demonstração |
| Vazamento de secret em log do cliente | Baixa | Médio — compromete autenticidade para um endpoint | Rotação de secret self-service (grace period 24h). Secret é hasheada no nosso banco. Logs internos nunca expõem secrets |
| Volume de eventos crescer e polling de 2s virar gargalo | Baixa (curto prazo) | Médio — latência aumenta | Monitorar métrica de lag. Plano de escalabilidade documentado (múltiplos workers com particionamento por order_id) |

## Critérios de Aceitação

1. Cliente consegue criar, visualizar, editar e remover configurações de webhook via API
2. Ao mudar o status de um pedido, um evento com payload contendo `event_id`, `order_id`, `from_status`, `to_status` e `timestamp` é enviado ao endpoint configurado
3. Se o endpoint do cliente estiver offline, o sistema realiza 5 retries com intervalos de 1m, 5m, 30m, 2h e 12h (6 tentativas no total, incluindo a inicial)
4. Após esgotar as 6 tentativas, o evento aparece na DLQ e pode ser reprocessado manualmente por um admin
5. O payload enviado contém assinatura HMAC-SHA256 no header `X-Signature`, verificável pelo cliente com a secret fornecida
6. Secret de webhook pode ser rotacionada, e a secret anterior permanece funcional por 24 horas
7. URLs que não usam HTTPS são rejeitadas no momento do cadastro com código de erro `WEBHOOK_INVALID_URL`
8. Cliente consegue consultar o histórico de entregas dos seus webhooks com status de sucesso/falha e detalhes da resposta

## Estratégia de Testes e Validação

### Testes de unidade
- Criação de webhook: validação de URL HTTPS, geração de secret, persistência
- Função `publishWebhookEvent`: filtrar endpoints elegíveis por customer_id e eventos
- Cálculo de backoff: verificar `next_retry_at` para cada tentativa
- Montagem de payload: verificar campos obrigatórios, tamanho máximo
- Assinatura HMAC: verificar geração e verificação com secret conhecida

### Testes de integração
- Fluxo completo `changeStatus` → outbox: verificar que evento é inserido em transação e não aparece em rollback
- Worker: mock de endpoint HTTP, verificar entrega com sucesso, retry em falha, movimentação para DLQ
- Rotação de secret: verificar que secret antiga funciona por 24h e é rejeitada depois
- Endpoint admin: verificar que replay requer role ADMIN e move evento de volta para outbox

### Testes ponta a ponta
- Fluxo completo: criar pedido → mudar status → worker entrega em endpoint mock → verificar payload, headers, assinatura
- Cenário de falha: endpoint mock retorna 503 → verificar retry com intervalos → verificar DLQ → admin replay → entrega com sucesso

### Testes de segurança (revisão pela Sofia)
- Geração de secret: entropia, armazenamento hasheado
- HMAC: verificação de assinatura com payload adulterado
- Grace period: comportamento exato na transição de 24h
- Autorização: endpoints admin acessíveis apenas com role ADMIN
