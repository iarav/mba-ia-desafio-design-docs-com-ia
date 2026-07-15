# ADR-005: Worker em processo separado com polling de 2 segundos

**Status:** Decidido

**Data:** 2026-07-12

**Participantes:** Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Larissa (Tech Lead)

## Contexto

O disparo dos webhooks exige um componente que leia a tabela `webhook_outbox` e execute as chamadas HTTP para os endpoints dos clientes. Esse componente precisa rodar continuamente, ser independente do ciclo de vida da API HTTP, e conseguir processar eventos com latência inferior a 10 segundos (requisito definido pelo PM junto aos clientes). É necessário decidir: o worker roda no mesmo processo da API ou em processo separado? Qual o mecanismo de leitura da outbox?

## Decisão

O worker rodará como **processo separado da API HTTP**, com **polling da outbox a cada 2 segundos**.

- **Entry point dedicada:** `src/worker.ts`, com script `npm run worker` no `package.json`
- **Polling:** A cada 2 segundos, o worker consulta os eventos pendentes mais antigos (ordenados por `created_at`), processa em batch, e marca como entregues
- **Processo independente:** O worker conecta no mesmo banco MySQL com a mesma `DATABASE_URL`, mas instancia seu próprio `PrismaClient` (já que `PrismaClient` é por processo)
- **Garantia de ordering:** Com um único worker (single-process), os eventos de um mesmo `order_id` são processados em ordem de inserção (`created_at`). Essa garantia não se mantém com múltiplos workers em paralelo, o que está documentado como limitação conhecida

## Alternativas consideradas

### Worker dentro do mesmo processo da API

**Descartada** porque:
- Se a API reinicia (deploy, crash), o worker morre junto e eventos ficam parados
- Competição por event loop entre requests HTTP e processamento de webhooks
- Dificulta escalar API e worker independentemente no futuro

### MySQL triggers + notificação externa

Usar triggers do MySQL para reagir a inserções na outbox. **Descartada** porque:
- MySQL não tem mecanismo nativo de notificação a processos externos (equivalente ao `NOTIFY/LISTEN` do PostgreSQL)
- Triggers executam apenas SQL; para notificar o worker seria necessário improvisar (escrever em arquivo, bater em endpoint interno), criando acoplamento frágil

### Intervalo de polling menor (ex.: 500ms)

**Descartado** porque:
- Aumentaria carga no banco sem benefício perceptível — 2s já atende o requisito de <10s
- Com 500ms, o banco receberia 4x mais queries de polling para o mesmo volume de eventos

## Consequências

### Positivas
- Desacoplamento: API e worker têm ciclos de vida independentes; deploy de um não afeta o outro
- Simplicidade: polling é trivial de implementar, testar e depurar comparado a mecanismos reativos
- Determinístico: com single-worker, eventos do mesmo pedido são entregues em ordem
- Reusa stack existente: mesmo Prisma, mesmo banco, mesmo logger Pino

### Negativas
- Latência mínima de 2 segundos (pior caso) entre mudança de status e início do processamento
- Single-worker é ponto único de falha: se o worker cair, eventos acumulam até ser reiniciado
- Escalabilidade limitada: para aumentar throughput no futuro, múltiplos workers exigiriam particionamento ou lock pessimista
- Polling gera queries constantes no banco mesmo quando não há eventos (carga baixa, mas contínua)
