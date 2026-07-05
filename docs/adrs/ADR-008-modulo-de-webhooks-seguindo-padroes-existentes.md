# ADR-008: Módulo de Webhooks Seguindo Padrões Existentes do Projeto

**Status:** Rascunho
**Data:** 2026-07-04
**Tags:** convenção, módulo, tratamento de erros, reuso
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Ao entrar no bloco de estrutura de código, Bruno descreveu o padrão já existente no projeto — cada domínio é um
módulo em `src/modules` com `controller`, `service`, `repository`, `routes` e `schemas` — e propôs que o
webhook siga o mesmo modelo, numa pasta `src/modules/webhooks` ([09:27] Bruno). Diego concordou e perguntou onde
ficaria o worker; Bruno propôs `src/worker.ts` como entry point separado (ver ADR-003), com a lógica de
processamento dentro do módulo, em um arquivo como `webhook.worker.ts` ou `webhook.processor.ts` ([09:28]
Bruno).

Sobre tratamento de erros, Bruno propôs reaproveitar o padrão já existente: a classe `AppError` e classes
específicas como `InsufficientStockError`/`InvalidStatusTransitionError`, todas com um código de erro (ex.
`INSUFFICIENT_STOCK`); para o webhook, propôs códigos com prefixo `WEBHOOK_`, como `WEBHOOK_NOT_FOUND`,
`WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` ([09:28] Bruno), confirmado por Larissa: *"Prefixo WEBHOOK_
pra tudo do módulo."* ([09:29] Larissa). Bruno também apontou que o logger (Pino) já está integrado ao projeto
inteiro e que o middleware de erro centralizado já trata `AppError`, `ZodError` e erros do Prisma sem precisar
de qualquer mudança ([09:29] Bruno).

## Fatores da Decisão

- Já existe um padrão consistente de módulo (`controller`/`service`/`repository`/`schemas`/`routes`) usado por
  todos os domínios do projeto — não há motivo para criar um padrão novo só para webhooks ([09:27] Bruno)
- Já existe uma hierarquia de erros (`AppError` + subclasses com `errorCode`) e um middleware central que já
  trata `AppError`, `ZodError` e erros do Prisma sem exigir mudança ([09:28]-[09:29] Bruno)
- O logger (Pino) já está integrado ao projeto inteiro ([09:29] Bruno)

## Opções Consideradas

1. **Módulo `src/modules/webhooks` seguindo o padrão existente, reaproveitando `AppError`/Pino/error
   middleware/Zod, com códigos de erro prefixados por `WEBHOOK_`** (proposta por Bruno)

Não houve alternativa real debatida: a proposta de convenção/reuso foi apresentada por Bruno e aceita
diretamente pelo grupo, sem uma opção concorrente de criar um padrão de módulo ou tratamento de erro diferente
para o webhook.

## Decisão Tomada

*"Decisão: reuso máximo do que já existe. AppError, Pino, error middleware, padrão de módulos, padrão de
schemas Zod, padrão de códigos de erro. Webhook fica como módulo igual aos outros."* ([09:30] Larissa).

## Prós e Contras das Opções

### Módulo padrão + reuso máximo
- Bom, porque mantém consistência arquitetural com os demais módulos do projeto (`auth`, `users`, `customers`,
  `products`, `orders`) ([09:27] Bruno)
- Bom, porque o middleware de erro centralizado já trata `AppError` sem qualquer alteração necessária ([09:29]
  Bruno)
- Bom, porque reduz esforço de implementação e curva de aprendizado para quem já conhece a base de código
  ([09:29]-[09:30])
- Ruim, porque herda qualquer limitação estrutural que o padrão atual já tenha (ex. wiring manual de
  dependências em `buildControllers`, sem container de DI)

## Consequências

O projeto ganha uma pasta `src/modules/webhooks` com `webhook.controller.ts`, `webhook.service.ts`,
`webhook.repository.ts`, `webhook.schemas.ts` e `webhook.routes.ts`, seguindo a mesma estrutura dos demais
módulos, além de um arquivo de processamento (`webhook.worker.ts` ou `webhook.processor.ts`) chamado a partir de
`src/worker.ts` (ADR-003). As classes de erro do módulo estendem `AppError` com códigos prefixados por
`WEBHOOK_`, seguindo a convenção de `http-errors.ts`. Não são necessárias mudanças no middleware de erro
centralizado nem na configuração do logger.

## Referências

- TRANSCRICAO.md — [09:27]-[09:28] Bruno/Diego
- TRANSCRICAO.md — [09:28]-[09:29] Bruno
- TRANSCRICAO.md — [09:29]-[09:30] Larissa
- src/modules/orders/ (estrutura de módulo existente usada como referência)
- src/shared/errors/http-errors.ts (padrão de `AppError` e `errorCode`)
- src/middlewares/error.middleware.ts (middleware central de erro)
- src/shared/logger/index.ts (Pino)
