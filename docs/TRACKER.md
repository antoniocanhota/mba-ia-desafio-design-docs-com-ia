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
