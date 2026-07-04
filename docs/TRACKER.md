# Tracker

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| T001 | ADR-001 | Decisão | Padrão outbox em MySQL para captura transacional de eventos de webhook | TRANSCRICAO | [09:08] Larissa |
| T002 | ADR-001 | Código | `OrderService.changeStatus` como ponto de integração transacional | CODIGO | src/modules/orders/order.service.ts |
| T003 | ADR-001 | Código | Precedente de tabela transacional single-row (`OrderNumberSequence`) | CODIGO | prisma/schema.prisma |
| T004 | ADR-002 | Decisão | Consumo do outbox via polling a cada 2 segundos | TRANSCRICAO | [09:10] Larissa |
| T005 | ADR-003 | Decisão | Worker roda como processo separado, com `PrismaClient` próprio | TRANSCRICAO | [09:11] Larissa |
| T006 | ADR-003 | Código | Entry point existente citado como precedente para `src/worker.ts` | CODIGO | src/server.ts |
| T007 | ADR-004 | Decisão | Retry com backoff exponencial, 5 tentativas (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Larissa |
| T008 | ADR-005 | Decisão | DLQ em tabela separada (`webhook_dead_letter`) com replay administrativo restrito a ADMIN | TRANSCRICAO | [09:36] Larissa |
| T009 | ADR-005 | Código | Reuso do middleware `requireRole` para restringir replay a ADMIN | CODIGO | src/middlewares/auth.middleware.ts |
| T010 | ADR-006 | Decisão | Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | TRANSCRICAO | [09:22] Sofia |
| T011 | ADR-007 | Decisão | Entrega at-least-once com deduplicação por `X-Event-Id` | TRANSCRICAO | [09:26] Larissa |
| T012 | ADR-008 | Decisão | Módulo `src/modules/webhooks` seguindo padrões existentes do projeto | TRANSCRICAO | [09:30] Larissa |
| T013 | ADR-008 | Código | Estrutura de módulo existente usada como referência | CODIGO | src/modules/orders/ |
| T014 | ADR-008 | Código | Padrão de `AppError` e `errorCode` | CODIGO | src/shared/errors/http-errors.ts |
| T015 | ADR-008 | Código | Middleware central de erro | CODIGO | src/middlewares/error.middleware.ts |
| T016 | ADR-008 | Código | Logger Pino já integrado ao projeto | CODIGO | src/shared/logger/index.ts |
| T017 | RFC | Alternativa Descartada | Disparo síncrono dentro de `OrderService.changeStatus` descartado por acoplar disponibilidade de cliente externo à transação crítica | TRANSCRICAO | [09:04] Bruno |
| T018 | RFC | Alternativa Descartada | Fila externa dedicada (Redis Streams) descartada por overengineering para um time pequeno | TRANSCRICAO | [09:07] Larissa |
| T019 | RFC | Alternativa Descartada | Trigger de banco de dados para notificação reativa descartado por MySQL não ter equivalente a `NOTIFY`/`LISTEN` | TRANSCRICAO | [09:09] Diego |
| T020 | RFC | Alternativa Descartada | Entrega exactly-once descartada por exigir coordenação complexa entre plataforma e cliente | TRANSCRICAO | [09:25] Diego |
| T021 | RFC | Questão em Aberto | Notificação de falhas persistentes ao cliente (ex. e-mail) adiada para fase futura | TRANSCRICAO | [09:37] Marcos |
| T022 | RFC | Questão em Aberto | Rate limiting de envio ao cliente deixado para observação futura | TRANSCRICAO | [09:38] Diego |
| T023 | RFC | Questão em Aberto | Dashboard visual para o cliente considerado fora de escopo, projeto separado do time de frontend | TRANSCRICAO | [09:39] Marcos |
| T024 | RFC | Questão em Aberto | Escala para múltiplos workers e garantia de ordering global tratada como limitação conhecida, não decisão agora | TRANSCRICAO | [09:12] Diego |
| T025 | RFC | Decisão | Proposta da função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada a partir de `changeStatus`, recebendo o `tx` em vez de repository inteiro | TRANSCRICAO | [09:41] Bruno/Diego |
