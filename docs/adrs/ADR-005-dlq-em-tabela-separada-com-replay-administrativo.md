# ADR-005: Dead Letter Queue em Tabela Separada com Replay Administrativo

**Status:** Rascunho
**Data:** 2026-07-04
**Tags:** DLQ, resiliência, auditoria, autorização
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Depois de definir o esquema de retry (ADR-004), faltava decidir onde registrar os eventos que esgotaram todas as
tentativas: *"DLQ. Faz numa tabela separada ou marca como 'failed' na própria outbox?"* ([09:17] Larissa). Diego
propôs uma tabela `webhook_dead_letter` separada, guardando o payload, o motivo da falha e o timestamp — deixando
a leitura da outbox principal mais limpa e servindo como evidência para debug e reprocessamento ([09:18] Diego).

Bruno perguntou como o reprocessamento aconteceria e se haveria um endpoint para isso ([09:18] Bruno). Diego
propôs um endpoint administrativo manual, `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento
na outbox como pendente ([09:18] Diego). Mais adiante, ao tratar da parte de segurança, Sofia definiu que esse
endpoint exige a role ADMIN — "mexer em fila de entrega de notificação não é coisa de operador" — e que a ação
de replay precisa ser logada para auditoria ([09:36] Sofia). Larissa fechou reaproveitando o middleware de
autorização já existente no projeto ([09:36] Larissa).

## Fatores da Decisão

- Manter a leitura da outbox principal limpa, sem misturar eventos falhos definitivamente com os pendentes
  ([09:18] Diego)
- Precisa de evidência (payload, motivo da falha, timestamp) para debug e eventual reprocessamento ([09:18]
  Diego)
- Mexer na fila de entrega de notificações é operação sensível e exige nível de permissão mais alto, com
  trilha de auditoria de quem executou o replay ([09:36] Sofia)

## Opções Consideradas

1. **Tabela `webhook_dead_letter` separada + endpoint admin de replay** (proposta por Diego)
2. **Marcar o evento como "failed" na própria tabela outbox**

## Decisão Tomada

Tabela separada com endpoint de replay administrativo: proposta de Diego aceita por Bruno ("Faz sentido",
[09:18]) e registrada por Larissa ("Anota também. Endpoint admin pra replay manual de DLQ.", [09:19]). O
requisito de autorização foi fechado depois: *"Decidido, role ADMIN obrigatório no replay e a gente reaproveita
o requireRole que já existe."* ([09:36] Larissa).

## Prós e Contras das Opções

### Tabela separada + replay admin
- Bom, porque isola eventos definitivamente falhos da outbox operacional, facilitando a leitura do worker e o
  monitoramento ([09:18] Diego)
- Bom, porque guarda motivo da falha e payload como evidência para debug e reprocessamento ([09:18] Diego)
- Bom, porque reaproveita o middleware `requireRole` já existente no projeto para restringir o replay a ADMIN
  ([09:36] Larissa)
- Ruim, porque exige uma tabela e um endpoint novos, aumentando a superfície de manutenção

### Marcar "failed" na outbox
- Bom, porque não exigiria criar uma tabela nova
- Ruim, porque misturaria eventos falhos definitivamente com eventos pendentes na mesma tabela, dificultando a
  leitura do worker e o monitoramento operacional — motivação implícita na proposta de Diego por uma tabela
  separada ([09:18] Diego)

## Consequências

O projeto ganha uma tabela `webhook_dead_letter` (payload, motivo da falha, timestamp) e um endpoint
`POST /admin/webhooks/dead-letter/:id/replay`, protegido por `requireRole(ADMIN)`. Toda execução de replay
precisa ser registrada para auditoria (quem executou, quando). Eventos reenfileirados voltam para a outbox como
pendentes e reentram no fluxo normal de retry (ADR-004).

## Referências

- TRANSCRICAO.md — [09:17]-[09:19] Larissa/Diego/Bruno
- TRANSCRICAO.md — [09:35]-[09:36] Larissa/Diego/Sofia
- src/middlewares/auth.middleware.ts (`requireRole`)
