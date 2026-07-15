# ADR-002: Política de retry com backoff exponencial e DLQ

**Status:** Decidido

**Data:** 2026-07-12

**Participantes:** Diego (Eng. Plataforma), Bruno (Eng. Pedidos), Larissa (Tech Lead), Marcos (PM)

## Contexto

O worker que dispara webhooks para endpoints externos está sujeito a falhas transientes: cliente fora do ar, timeout de rede, manutenção programada do lado do cliente. É necessário definir uma política de retry que equilibre resiliência (não desistir cedo demais) com higiene operacional (não acumular eventos pendentes indefinidamente). Após esgotadas as tentativas, o evento precisa ser preservado para diagnóstico e reprocessamento manual.

## Decisão

Adotaremos **5 retries com backoff exponencial após a 1ª tentativa (6 tentativas no total)**, seguidas de movimentação para uma **DLQ (Dead Letter Queue) em tabela separada** (`webhook_dead_letter`).

Progressão das tentativas:
1. 1ª tentativa (envio inicial): imediata, na primeira passagem do worker
2. 2ª tentativa: 1 minuto após a 1ª falha
3. 3ª tentativa: 5 minutos após a 2ª falha
4. 4ª tentativa: 30 minutos após a 3ª falha
5. 5ª tentativa: 2 horas após a 4ª falha
6. 6ª tentativa (última): 12 horas após a 5ª falha

Total da janela de retry: aproximadamente 15 horas. Após a 6ª falha (última tentativa), o evento é movido para `webhook_dead_letter` com payload, motivo da falha e timestamp.

Reprocessamento da DLQ será feito via endpoint admin: `POST /admin/webhooks/dead-letter/:id/replay`, que reinsere o evento na outbox como pendente. O endpoint exige role `ADMIN` (reaproveitando o middleware `requireRole` existente) e registra em log quem executou o replay.

## Alternativas consideradas

### 3 tentativas com backoff mais agressivo

Sugerida pelo Bruno. **Descartada** porque:
- 3 tentativas com intervalos curtos cobririam cerca de 30 minutos
- Clientes já tiveram indisponibilidade de 2 horas em manutenção planejada
- Uma janela tão curta geraria falsos positivos de falha permanente

### Retry indefinido com backoff

**Descartada** porque:
- Um cliente que descontinue o serviço sem avisar deixaria eventos pendurados para sempre
- Polui a tabela de outbox e dificulta monitoramento
- A DLQ explícita força uma ação consciente (reprocessar ou descartar)

### Marcar como "failed" na própria tabela outbox

Em vez de tabela separada para DLQ. **Descartada** porque:
- Mistura eventos ativos com eventos mortos na mesma tabela, piorando a performance do polling
- Dificulta queries de monitoramento e diagnóstico
- Tabela separada funciona como evidência para debugging e auditoria

## Consequências

### Positivas
- Cobre janelas de indisponibilidade de até ~15h, suficiente para manutenções planejadas típicas
- DLQ em tabela separada mantém a outbox principal enxuta e rápida de ler
- Reprocessamento manual com registro de auditoria (quem, quando) atende requisitos de compliance
- Backoff progressivo evita bombardear o cliente enquanto ele se recupera

### Negativas
- Eventos que falham permanentemente só são percebidos após ~15h (a não ser que haja monitoramento ativo da DLQ)
- Reprocessamento é manual — não há mecanismo automático de retry após entrada na DLQ
- Duas tabelas para gerenciar (outbox + dead_letter) em vez de uma
