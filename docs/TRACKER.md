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
| T018 | RFC | Alternativa Descartada | Fila externa dedicada (Redis Streams) descartada por overengineering para um time pequeno | TRANSCRICAO | [09:07] Larissa/Diego |
| T019 | RFC | Alternativa Descartada | Trigger de banco de dados para notificação reativa descartado por MySQL não ter equivalente a `NOTIFY`/`LISTEN` | TRANSCRICAO | [09:09] Diego |
| T020 | RFC | Alternativa Descartada | Entrega exactly-once descartada por exigir coordenação complexa entre plataforma e cliente | TRANSCRICAO | [09:25] Diego |
| T021 | RFC | Questão em Aberto | Notificação de falhas persistentes ao cliente (ex. e-mail) adiada para fase futura | TRANSCRICAO | [09:37] Marcos |
| T022 | RFC | Questão em Aberto | Rate limiting de envio ao cliente deixado para observação futura | TRANSCRICAO | [09:38] Diego |
| T023 | RFC | Questão em Aberto | Dashboard visual para o cliente considerado fora de escopo, projeto separado do time de frontend | TRANSCRICAO | [09:39] Marcos |
| T024 | RFC | Questão em Aberto | Escala para múltiplos workers e garantia de ordering global tratada como limitação conhecida, não decisão agora | TRANSCRICAO | [09:12] Diego |
| T025 | RFC | Decisão | Proposta da função `publishWebhookEvent(tx, order, fromStatus, toStatus)` chamada a partir de `changeStatus`, recebendo o `tx` em vez de repository inteiro | TRANSCRICAO | [09:41] Bruno/Diego |
| T026 | ADR-006 | Restrição | Precedente real de vazamento de secret em log de aplicação de cliente, motivando secret única por endpoint | TRANSCRICAO | [09:22] Diego |
| T027 | ADR-004 | Restrição | Precedente real de indisponibilidade de 2h por manutenção planejada em cliente, motivando rejeitar 3 tentativas de retry | TRANSCRICAO | [09:16] Diego |
| T028 | ADR-002 | Requisito Não Funcional | Requisito de latência do cliente (abaixo de 10 segundos) que valida a folga do polling de 2 segundos | TRANSCRICAO | [09:02] Marcos |
| T029 | FDD | Fluxo | Fluxo principal: mudança de status insere evento na webhook_outbox na mesma transação de changeStatus | TRANSCRICAO | [09:41] Bruno/Diego |
| T030 | FDD | Fluxo | Filtro de eventos aplicado na inserção do outbox, não no envio | TRANSCRICAO | [09:33]-[09:34] Marcos/Bruno/Diego |
| T031 | FDD | Fluxo | Fluxo alternativo de rotação de secret com grace period de 24h para a secret anterior | TRANSCRICAO | [09:21] Sofia |
| T032 | FDD | Fluxo | Fluxo alternativo de replay administrativo de DLQ reenfileira evento como pendente | TRANSCRICAO | [09:18]-[09:19] Diego/Bruno |
| T033 | FDD | Contrato Público | POST /api/v1/webhooks — cadastro de webhook, secret retornada só na criação | TRANSCRICAO | [09:31] Marcos |
| T034 | FDD | Contrato Público | GET /api/v1/webhooks — listagem de webhooks do customer | TRANSCRICAO | [09:33] Bruno |
| T035 | FDD | Contrato Público | PATCH /api/v1/webhooks/:id — edição de webhook | TRANSCRICAO | [09:33] Bruno |
| T036 | FDD | Contrato Público | DELETE /api/v1/webhooks/:id — remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| T037 | FDD | Contrato Público | POST /api/v1/webhooks/:id/secret/rotate — rotação de secret | TRANSCRICAO | [09:21] Sofia |
| T038 | FDD | Contrato Público | GET /api/v1/webhooks/:id/deliveries — histórico dos últimos 100 envios | TRANSCRICAO | [09:34] Marcos |
| T039 | FDD | Contrato Público | POST /api/v1/admin/webhooks/dead-letter/:id/replay — replay administrativo restrito a ADMIN | TRANSCRICAO | [09:18] Diego |
| T040 | FDD | Contrato Público | Entrega HTTP outbound ao cliente: headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id e payload snake_case (event_id, event_type, order_id, order_number, from_status, to_status, customer_id, total_cents) | TRANSCRICAO | [09:43]-[09:45] Diego/Sofia |
| T041 | FDD | Erro | WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED — códigos de exemplo citados na reunião | TRANSCRICAO | [09:28] Bruno |
| T042 | FDD | Erro | WEBHOOK_PAYLOAD_TOO_LARGE — payload de evento acima de 64KB é erro, não truncamento | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego/Larissa |
| T043 | FDD | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| T044 | FDD | Requisito Não Funcional | Limite de payload de 64KB por evento | TRANSCRICAO | [09:23]-[09:24] Diego |
| T045 | FDD | Requisito Não Funcional | Redação de secret/signature em log, prevenindo repetição de vazamento em log de aplicação | TRANSCRICAO | [09:22] Diego |
| T046 | FDD | Código | Ponto de integração: OrderService.changeStatus chama publishWebhookEvent dentro da transação | CODIGO | src/modules/orders/order.service.ts |
| T047 | FDD | Código | Padrão AppError/errorCode reaproveitado para classes de erro WEBHOOK_* | CODIGO | src/shared/errors/http-errors.ts |
| T048 | FDD | Código | Middleware central de erro já trata AppError sem mudança | CODIGO | src/middlewares/error.middleware.ts |
| T049 | FDD | Código | authenticate e requireRole('ADMIN') reaproveitados para CRUD e replay de DLQ | CODIGO | src/middlewares/auth.middleware.ts |
| T050 | FDD | Código | Logger Pino reaproveitado, com novos redactPaths para secret/signature | CODIGO | src/shared/logger/index.ts |
| T051 | FDD | Código | src/server.ts como precedente de entry point para o novo src/worker.ts | CODIGO | src/server.ts |
| T052 | FDD | Código | Modelos do schema Prisma como convenção para novas tabelas webhook_outbox/webhook_dead_letter | CODIGO | prisma/schema.prisma |
| T053 | FDD | Código | Estrutura de módulo (controller/service/repository/schemas/routes) como molde para src/modules/webhooks | CODIGO | src/modules/orders/ |
| T054 | FDD | Código | buildControllers como padrão de wiring manual para os novos controllers do módulo webhook | CODIGO | src/app.ts |
| T055 | FDD | Fluxo | Processamento pelo worker: leitura de eventos pendentes em lote, ordenados por created_at, a cada 2s | TRANSCRICAO | [09:07]-[09:09] Diego |
| T056 | FDD | Fluxo | Retry: após falha de entrega, evento é reagendado conforme backoff exponencial até a 5ª tentativa | TRANSCRICAO | [09:14]-[09:17] Larissa/Diego |
| T057 | FDD | Fluxo | DLQ: evento que esgota as tentativas sai da outbox e é registrado em tabela separada para reprocessamento manual | TRANSCRICAO | [09:17]-[09:19] Larissa/Diego/Bruno |
| T058 | FDD | Restrição | Escopo restrito a webhooks outbound; plataforma não recebe webhooks de clientes | TRANSCRICAO | [09:02]-[09:03] Sofia/Marcos |
| T059 | docs/PRD.md | Objetivo | Eliminar polling manual: latência de notificação abaixo de 10s | TRANSCRICAO | [09:02] Marcos |
| T060 | docs/PRD.md | Escopo | Cadastro, edição, remoção e listagem de webhooks com filtro de status | TRANSCRICAO | [09:31]-[09:34] Marcos/Bruno |
| T061 | docs/PRD.md | Escopo | Entrega de notificação HTTP assinada na mudança de status | TRANSCRICAO | [09:00]-[09:26] Marcos/Bruno/Larissa/Diego/Sofia |
| T062 | docs/PRD.md | Escopo | Rotação de secret com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| T063 | docs/PRD.md | Escopo | Histórico de entregas, últimos 100 registros | TRANSCRICAO | [09:34] Marcos |
| T064 | docs/PRD.md | Escopo | Reprocessamento manual de DLQ restrito a admin | TRANSCRICAO | [09:18]-[09:19], [09:35]-[09:36] Diego/Bruno/Sofia/Larissa |
| T065 | docs/PRD.md | Escopo | Notificação de falha por e-mail fora de escopo | TRANSCRICAO | [09:37]-[09:38] Marcos/Larissa |
| T066 | docs/PRD.md | Escopo | Rate limiting de envio fora de escopo | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
| T067 | docs/PRD.md | Escopo | Dashboard visual fora de escopo | TRANSCRICAO | [09:39]-[09:40] Marcos/Larissa |
| T068 | docs/PRD.md | Escopo | Garantia de ordering global fora de escopo | TRANSCRICAO | [09:12]-[09:13] Diego |
| T069 | docs/PRD.md | Escopo | Webhooks inbound fora de escopo | TRANSCRICAO | [09:02]-[09:03] Sofia/Marcos |
| T070 | docs/PRD.md | Escopo | Arquivamento de eventos entregues após 30 dias fora de escopo | TRANSCRICAO | [09:08] Diego |
| T071 | docs/PRD.md | Requisito Funcional | FR-001 Cadastro de webhook | TRANSCRICAO | [09:31] Marcos |
| T072 | docs/PRD.md | Requisito Funcional | FR-002 Edição de webhook | TRANSCRICAO | [09:33] Bruno |
| T073 | docs/PRD.md | Requisito Funcional | FR-003 Remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| T074 | docs/PRD.md | Requisito Funcional | FR-004 Listagem de webhooks | TRANSCRICAO | [09:33] Bruno |
| T075 | docs/PRD.md | Requisito Funcional | FR-005 Rotação de secret | TRANSCRICAO | [09:21] Sofia |
| T076 | docs/PRD.md | Requisito Funcional | FR-006 Consulta de histórico de entregas | TRANSCRICAO | [09:34] Marcos |
| T077 | docs/PRD.md | Requisito Funcional | FR-007 Entrega de notificação de mudança de status | TRANSCRICAO | [09:00]-[09:26] Marcos/Bruno/Larissa/Diego/Sofia |
| T078 | docs/PRD.md | Requisito Funcional | FR-008 Reprocessamento administrativo de entregas malsucedidas | TRANSCRICAO | [09:18]-[09:19], [09:35]-[09:36] |
| T079 | docs/PRD.md | Requisito Não Funcional | Performance: timeout de 10s por tentativa e latência abaixo de 10s | TRANSCRICAO | [09:42] Diego / [09:02] Marcos |
| T080 | docs/PRD.md | Requisito Não Funcional | Segurança: HMAC-SHA256, secret por endpoint, rotação, HTTPS obrigatório, ADMIN no replay | TRANSCRICAO | [09:20]-[09:23], [09:36] Sofia |
| T081 | docs/PRD.md | Requisito Não Funcional | Confiabilidade: atomicidade outbox, entrega at-least-once, retry com backoff | TRANSCRICAO | [09:04]-[09:26] Bruno/Diego/Larissa/Marcos/Sofia |
| T082 | docs/PRD.md | Requisito Não Funcional | Compatibilidade: payload de evento limitado a 64KB | TRANSCRICAO | [09:23]-[09:24] Sofia/Diego/Larissa |
| T083 | docs/PRD.md | Requisito Não Funcional | Compliance: auditoria de reprocessamentos administrativos | TRANSCRICAO | [09:36] Sofia |
| T084 | docs/PRD.md | Decisão | Entrega assíncrona via outbox, não síncrona | TRANSCRICAO | [09:04] Bruno |
| T085 | docs/PRD.md | Decisão | Garantir at-least-once, não exactly-once | TRANSCRICAO | [09:25] Diego |
| T086 | docs/PRD.md | Decisão | Secret única por endpoint, com rotação e grace period de 24h | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| T087 | docs/PRD.md | Decisão | 5 tentativas de retry com backoff de até 12h antes de desistir | TRANSCRICAO | [09:16] Diego |
| T088 | docs/PRD.md | Dependência | Revisão de segurança da Sofia antes do deploy (2 dias úteis) | TRANSCRICAO | [09:46]-[09:47] Sofia/Larissa |
| T089 | docs/PRD.md | Dependência | Documentação no portal de desenvolvedor para os clientes | TRANSCRICAO | [09:26], [09:40] Marcos |
| T090 | docs/PRD.md | Dependência | Cliente implementa verificação de HMAC e dedup do lado dele | TRANSCRICAO | [09:25] Diego |
| T091 | docs/PRD.md | Risco | Churn do cliente Atlas se o prazo não for cumprido | TRANSCRICAO | [09:45]-[09:47] Larissa/Sofia/Marcos |
| T092 | docs/PRD.md | Risco | Indisponibilidade prolongada do endpoint do cliente | TRANSCRICAO | [09:16]-[09:17] Diego |
| T093 | docs/PRD.md | Risco | Vazamento de secret de um cliente | TRANSCRICAO | [09:21]-[09:22] Sofia/Diego |
| T094 | docs/PRD.md | Risco | Sobrecarga do endpoint do cliente com muitas mudanças simultâneas | TRANSCRICAO | [09:38]-[09:39] Diego/Larissa |
