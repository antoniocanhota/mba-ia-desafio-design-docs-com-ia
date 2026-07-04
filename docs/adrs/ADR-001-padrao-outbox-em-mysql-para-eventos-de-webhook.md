# ADR-001: Padrão Outbox em MySQL para Captura Transacional de Eventos de Webhook

**Status:** Rascunho
**Data:** [A DEFINIR]
**Tags:** outbox, transação, resiliência, integração
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados em tempo real
quando o status de um pedido muda, em vez de continuar fazendo polling no `GET /orders` ([09:00] Marcos). Para
eles, qualquer coisa abaixo de 10 segundos já conta como "tempo real" ([09:02] Marcos).

A primeira pergunta de arquitetura foi se o disparo do webhook deveria ser síncrono, dentro do próprio serviço
de pedidos, ou passar por algum mecanismo assíncrono ([09:03] Larissa). Bruno apontou que a transação de mudança
de status já é pesada — atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` — e
que adicionar uma chamada HTTP no meio dela faria qualquer cliente lento travar a mudança de status de outros
pedidos, além de não haver como fazer rollback de uma chamada HTTP já enviada caso o cliente esteja fora do ar
([09:04] Bruno). Isso descartou o disparo síncrono dentro de `OrderService.changeStatus`.

Diego propôs então o padrão outbox: dentro da mesma transação SQL que atualiza `orders` e insere em
`order_status_history`, uma linha é inserida numa tabela `webhook_outbox` com o evento; um worker separado lê
essa tabela e dispara as chamadas HTTP. Se a transação principal der commit, o evento foi registrado; se der
rollback, o evento some junto — sem inconsistência possível ([09:06] Diego).

## Fatores da Decisão

- A transação de `changeStatus` já é pesada (orders + order_status_history + estoque); acrescentar uma chamada
  HTTP síncrona aumentaria a exposição a falhas externas ([09:04] Bruno)
- Não há como reverter (rollback) o efeito de uma chamada HTTP já enviada a um cliente externo ([09:04] Bruno)
- O cliente pode estar fora do ar; um disparo síncrono bloquearia a mudança de status de outros pedidos
  ([09:04] Bruno)
- O time é pequeno; subir infraestrutura nova apenas para isso é overengineering ([09:07] Diego)

## Opções Consideradas

1. **Padrão outbox em MySQL** (proposta por Diego)
2. **Disparo síncrono dentro de `OrderService.changeStatus`**
3. **Fila externa dedicada (Redis Streams)**

## Decisão Tomada

Outbox em MySQL, dentro da mesma transação da mudança de status: *"Tá decidido então: outbox em MySQL."*
([09:08] Larissa). A tabela tem índice em status (pendente, processando, falhou, entregue) e em `created_at`; o
worker lê em lotes pequenos apenas os eventos pendentes ([09:07]-[09:08] Diego). O arquivamento de linhas já
entregues após ~30 dias ficou fora do escopo desta feature ([09:08] Diego).

A inserção acontece através de uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o
client de transação (`tx`) já aberto pelo `OrderService.changeStatus`, em vez de injetar um repository inteiro
no serviço de pedidos: *"Boa, função pura recebendo o tx. Não precisa injetar repository inteiro."* ([09:41]
Diego). Diego reforçou o motivo: *"Essencial. Se ficar fora da transação, perde a garantia toda."* ([09:41]
Diego). O identificador de cada evento na outbox segue o padrão de UUID já usado no resto do projeto ([09:51]
Larissa).

## Prós e Contras das Opções

### Padrão outbox em MySQL
- Bom, porque garante atomicidade entre a mudança de status e o registro do evento — commit e rollback ficam
  sempre consistentes ([09:06] Diego)
- Bom, porque reaproveita a infraestrutura de banco já existente, sem subir componente operacional novo
  ([09:07] Diego)
- Ruim, porque exige uma tabela adicional e um processo worker separado para operar (ver ADR-002 e ADR-003)

### Disparo síncrono
- Bom, porque é a opção mais simples de implementar, sem componente adicional
- Ruim, porque acopla a disponibilidade de um cliente externo à transação crítica de mudança de status,
  arriscando travar pedidos de outros clientes quando o endpoint deles está lento ou fora do ar ([09:04] Bruno)

### Fila externa (Redis Streams)
- Bom, porque oferece um mecanismo de fila dedicado e nativo para processamento de eventos
- Ruim, porque exigiria subir e operar uma infraestrutura nova (ex. Redis Cluster) para um time pequeno,
  configurando overengineering para o problema em questão ([09:07] Diego/Larissa)

## Consequências

A mudança de status de pedidos passa a ter um efeito colateral adicional dentro da mesma transação: a inserção
de um evento na `webhook_outbox`, via `publishWebhookEvent(tx, ...)` chamada a partir de
`OrderService.changeStatus`. Isso garante que nunca existirá um caso de status alterado sem o evento
correspondente registrado, nem o inverso.

Como consequência negativa, o sistema passa a depender de um processo worker separado para de fato entregar os
eventos (detalhado em ADR-002 e ADR-003), e de uma tabela adicional cujo crescimento precisa ser monitorado. A
política de arquivamento de eventos já entregues (após ~30 dias) ficou deliberadamente fora do escopo desta
decisão e desta fase da feature.

## Referências

- TRANSCRICAO.md — [09:03] Larissa
- TRANSCRICAO.md — [09:04] Bruno
- TRANSCRICAO.md — [09:06] Diego
- TRANSCRICAO.md — [09:07] Larissa/Diego
- TRANSCRICAO.md — [09:08] Diego/Larissa
- TRANSCRICAO.md — [09:40]-[09:41] Bruno/Diego
- TRANSCRICAO.md — [09:51] Larissa
- src/modules/orders/order.service.ts (`OrderService.changeStatus`)
- prisma/schema.prisma (`OrderNumberSequence`, precedente de tabela transacional single-row no projeto)
