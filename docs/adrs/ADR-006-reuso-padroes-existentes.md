# ADR-006: Reuso dos padrões existentes do projeto para o módulo de webhooks

**Status:** Decidido

**Data:** 2026-07-12

**Participantes:** Bruno (Eng. Pedidos), Diego (Eng. Plataforma), Larissa (Tech Lead), Sofia (Eng. Segurança)

## Contexto

O projeto já possui padrões consolidados de estrutura de módulos, tratamento de erros, logging, validação e middlewares. O novo módulo de webhooks poderia introduzir seus próprios padrões ou reutilizar o que já existe. A reunião foi explícita em preferir consistência com o código existente para reduzir a carga cognitiva do time e acelerar a implementação.

## Decisão

O módulo de webhooks **reaproveitará todos os padrões existentes do projeto**, sem introduzir novas bibliotecas, frameworks ou convenções.

Padrões reutilizados:

| Padrão | Localização no código | Uso no módulo de webhooks |
|--------|----------------------|---------------------------|
| Estrutura modular (controller, service, repository, routes, schemas) | `src/modules/orders/`, `src/modules/auth/`, etc. | `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts` |
| `AppError` e hierarquia de erros HTTP | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` | Erros do módulo estendem `AppError` com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`) |
| Error middleware centralizado | `src/middlewares/error.middleware.ts` | Sem alterações — o middleware já trata `AppError`, `ZodError` e `Prisma` errors, cobrindo os novos erros automaticamente |
| Logger Pino | `src/shared/logger/index.ts` | Worker e endpoints usam o mesmo `logger` com `service: 'order-management-api'` |
| Validação com Zod | `src/middlewares/validate.middleware.ts`, schemas em cada módulo | Schemas Zod para criação/edição de webhook, validação de URL HTTPS, filtros de status |
| Autenticação JWT + `requireRole` | `src/middlewares/auth.middleware.ts` | CRUD de webhooks autenticado com `authenticate`; endpoint admin de replay DLQ com `requireRole('ADMIN')` |
| `paginated()` helper | `src/shared/http/response.ts` | Listagem de webhooks e deliveries usa o mesmo formato paginado |
| UUID v4 para IDs | Todo o schema Prisma (`@default(uuid())`) | `webhook_outbox.event_id`, `webhook_endpoints.id`, `webhook_dead_letter.id` |
| Transações com `prisma.$transaction` | `src/modules/orders/order.service.ts` (método `changeStatus`) | Inserção na outbox dentro da mesma transação do `changeStatus`, usando o `tx` client |

## Alternativas consideradas

### Novas bibliotecas (ex.: Bull/BullMQ para filas, Winston para logging)

**Descartada** porque:
- Adicionaria dependências sem necessidade — os padrões atuais cobrem todos os requisitos
- Aumentaria a curva de aprendizado para o time
- Violaria o princípio de consistência: um módulo com stack diferente dos outros gera atrito em manutenção

### Módulo de webhooks com estrutura própria (fora de `src/modules/`)

**Descartada** porque:
- Todos os domínios da aplicação seguem a mesma estrutura modular
- Quebrar o padrão para um módulo novo criaria exceção sem justificativa técnica
- Ferramentas de navegação, testes e linting do time já assumem essa estrutura

## Consequências

### Positivas
- Curva de implementação mais curta: qualquer dev do time reconhece a estrutura imediatamente
- Error middleware centralizado captura erros do novo módulo sem alterações — zero boilerplate de tratamento de erro
- `requireRole('ADMIN')` para endpoint de replay DLQ é uma linha, sem código novo de autorização
- Logs do worker e dos endpoints seguem o mesmo formato e nível de log configurado, facilitando debugging
- Testes seguem os mesmos padrões dos módulos existentes (supertest + vitest)

### Negativas
- Acoplamento com padrões existentes: se o projeto migrar de Prisma ou Express no futuro, o módulo de webhooks migra junto
- O prefixo `WEBHOOK_` para códigos de erro é uma convenção local — não há mecanismo de enforce a não ser code review
- A estrutura de módulos (controller/service/repository) pode ser excessiva se o módulo crescer pouco; mas a consistência com o resto do projeto justifica
