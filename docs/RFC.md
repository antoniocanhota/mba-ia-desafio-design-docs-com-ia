# RFC — Sistema de Webhooks de Notificação de Pedidos

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | Larissa (Tech Lead) |
| **Status** | Em revisão |
| **Data** | quinta-feira, 09:00 |
| **Revisores** | Marcos (Product Manager), Bruno (Engenheiro Pleno, time de Pedidos), Diego (Engenheiro Sênior, time de Plataforma), Sofia (Engenheira de Segurança) |

## Resumo Executivo (TL;DR)

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados em até 10 segundos quando o status de um pedido muda, em vez de continuar fazendo polling manual no `GET /orders` ([09:00]-[09:02] Marcos). Propomos um sistema de webhooks outbound baseado em padrão outbox transacional no MySQL, consumido por um worker dedicado em polling, com retry com backoff exponencial e DLQ para falhas permanentes, autenticado via HMAC-SHA256 com secret por endpoint, e entrega com garantia at-least-once deduplicável por `event_id`. A estimativa é de três sprints, incluindo a revisão de segurança de Sofia ([09:45]-[09:47] Larissa/Sofia), para atender o prazo de fim de novembro pedido pela Atlas ([09:45] Marcos).

## Contexto e Problema

A motivação de negócio partiu de um pedido formal de três clientes B2B que hoje ficam fazendo polling repetido no `GET /orders` para saber se o status de seus pedidos mudou, o que torna a integração deles lenta e cara; a Atlas Comercial sinalizou risco de migrar para um concorrente caso a feature não saia até o fim do trimestre ([09:00] Marcos). O requisito de latência é flexível: qualquer notificação abaixo de 10 segundos já é considerada "tempo real" pelos clientes, desde que não precisem mais atualizar manualmente ([09:02] Marcos). O escopo foi delimitado a webhooks outbound (a plataforma envia, os clientes apenas recebem) ([09:02]-[09:03] Sofia/Marcos).

O desafio técnico central é que o ponto de mudança de status de pedidos, `OrderService.changeStatus`, já executa uma transação pesada — atualiza `orders`, insere em `order_status_history` e decrementa `stock_quantity` — e não pode ficar acoplada à disponibilidade de um sistema de terceiros: um cliente lento ou fora do ar travaria a mudança de status de outros pedidos, e não haveria como reverter uma chamada HTTP já enviada em caso de rollback ([09:04] Bruno). Essa restrição descartou o disparo síncrono e motivou a arquitetura assíncrona detalhada abaixo.

Depois de fechado o desenho de captura e entrega, a discussão avançou por resiliência (retry, DLQ), segurança (assinatura, rotação de secret) e, por fim, estrutura de código e requisitos funcionais de CRUD — sempre reaproveitando padrões já existentes na base de código em vez de introduzir componentes novos, dado que o time é pequeno ([09:07] Diego).

## Proposta Técnica

**Captura do evento.** A mudança de status em `OrderService.changeStatus` passa a inserir, na mesma transação SQL, um evento na tabela `webhook_outbox`, garantindo que commit e rollback da transação principal e do evento nunca fiquem inconsistentes entre si ([ADR-001](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md)). O evento é gravado como snapshot já renderizado no momento da inserção, para refletir o estado do pedido no instante da mudança, mesmo que o pedido seja alterado depois ([09:51]-[09:52] Larissa/Diego/Bruno).

**Processamento.** Um worker dedicado consome a outbox via polling a cada 2 segundos ([ADR-002](adrs/ADR-002-consumo-do-outbox-via-polling.md)), rodando como processo Node separado da API, com seu próprio `PrismaClient` apontando para a mesma `DATABASE_URL` ([ADR-003](adrs/ADR-003-worker-como-processo-separado.md)). A inserção do evento na outbox já é filtrada pela lista de status que cada webhook do cliente deseja receber, evitando gravar eventos que não seriam entregues a ninguém ([09:33]-[09:34] Marcos/Bruno/Diego).

**Entrega e resiliência.** Falhas de entrega acionam retry com backoff exponencial (5 tentativas, 1m/5m/30m/2h/12h) antes de considerar falha permanente ([ADR-004](adrs/ADR-004-retry-com-backoff-exponencial.md)). Eventos que esgotam as tentativas vão para uma tabela `webhook_dead_letter` separada, reprocessável manualmente por um endpoint administrativo restrito a `ADMIN` ([ADR-005](adrs/ADR-005-dlq-em-tabela-separada-com-replay-administrativo.md)). A garantia de entrega é at-least-once, com deduplicação do lado do cliente via `X-Event-Id` ([ADR-007](adrs/ADR-007-entrega-at-least-once-com-dedup-por-event-id.md)).

**Segurança.** Cada evento é assinado com HMAC-SHA256 usando uma secret única por endpoint webhook, com suporte a rotação e grace period de 24h para a secret antiga ([ADR-006](adrs/ADR-006-autenticacao-hmac-sha256-com-rotacao-de-secret.md)). Complementarmente, o cadastro de URL exige HTTPS e o payload tem um teto de 64KB, tratados como requisitos não funcionais e validação de schema, não como decisão arquitetural separada ([09:23]-[09:24] Sofia/Diego/Larissa).

**Estrutura de código.** O módulo segue o padrão já estabelecido do projeto (`controller`/`service`/`repository`/`schemas`/`routes` em `src/modules/webhooks`), reaproveitando `AppError`, o middleware de erro centralizado, Pino e o padrão de códigos de erro com prefixo `WEBHOOK_` ([ADR-008](adrs/ADR-008-modulo-de-webhooks-seguindo-padroes-existentes.md)). A integração com `OrderService.changeStatus` acontece por uma função `publishWebhookEvent(tx, order, fromStatus, toStatus)` que recebe o client de transação já aberto, evitando injetar um repository inteiro no serviço de pedidos ([09:41] Bruno/Diego, ver [ADR-001](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md)).

**Superfície de API.** O cliente gerencia seus webhooks via CRUD autenticado (POST/PATCH/DELETE/GET), informando URL, lista de status desejados e recebendo a secret gerada na criação; `customer_id` é passado explicitamente no corpo/path, não extraído do JWT, pois o JWT atual representa o usuário operador, não o cliente B2B ([09:31]-[09:33] Marcos/Bruno/Larissa). Um endpoint de histórico de entregas (`GET /webhooks/:id/deliveries`) expõe sucesso/falha, payload, resposta e tempo de resposta dos últimos envios ([09:34]-[09:35] Marcos). O detalhamento de payload, headers e matriz de erros fica para o FDD.

## Alternativas Consideradas

### Disparo síncrono dentro de `OrderService.changeStatus`

Cogitado como primeira opção antes de qualquer outra, foi descartado porque acoplaria a disponibilidade de um cliente externo à transação crítica de mudança de status: um cliente lento travaria a mudança de status de outros pedidos, e não haveria como reverter uma chamada HTTP já enviada em caso de falha na transação ([09:04] Bruno). A decisão vencedora (outbox assíncrono) está registrada em [ADR-001](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md).

### Fila externa dedicada (Redis Streams)

Levantada por Larissa como alternativa ao outbox em MySQL, foi descartada por exigir subir e operar infraestrutura nova (ex. Redis Cluster) para um time pequeno, configurando overengineering para o problema em questão ([09:07] Larissa/Diego). A decisão vencedora está registrada em [ADR-001](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md).

### Trigger de banco de dados para notificação reativa

Bruno sugeriu usar um trigger de banco para tornar o consumo do outbox mais reativo, evitando a espera do polling ([09:09] Bruno). Diego descartou a ideia porque o MySQL não tem um mecanismo nativo equivalente ao `NOTIFY`/`LISTEN` do Postgres: um trigger só executa SQL e não notifica um processo externo, exigindo soluções improvisadas e frágeis (escrever em arquivo, chamar um endpoint) ([09:09] Diego). A decisão vencedora (polling a cada 2 segundos) está registrada em [ADR-002](adrs/ADR-002-consumo-do-outbox-via-polling.md).

### Entrega exactly-once

Diego considerou garantir exactly-once em vez de at-least-once, mas descartou por exigir coordenação entre os dois lados (plataforma e cliente), aumentando muito a complexidade de implementação para o ganho obtido; o padrão de mercado (Stripe, GitHub) já resolve duplicatas com deduplicação do lado do cliente via identificador único ([09:25] Diego). A decisão vencedora está registrada em [ADR-007](adrs/ADR-007-entrega-at-least-once-com-dedup-por-event-id.md).

## Questões em Aberto

1. **Notificação de falhas persistentes ao cliente (ex. e-mail).** Marcos perguntou se haveria como avisar o cliente quando o webhook dele estivesse falhando repetidamente (ex. após 3 falhas seguidas). Larissa decidiu que está fora de escopo desta fase, ficando como possível fase futura após medição de impacto ([09:37]-[09:38] Marcos/Larissa).

2. **Rate limiting de envio ao cliente.** Diego levantou a preocupação de bombardear um cliente com muitas chamadas simultâneas caso vários pedidos dele mudem de status em um curto intervalo (ex. 50 pedidos em um minuto). O grupo decidiu não incluir no escopo atual, optando por observar o comportamento em produção e decidir depois se vira necessário ([09:38]-[09:39] Diego/Larissa).

3. **Dashboard visual para o cliente acompanhar seus webhooks.** Marcos perguntou sobre um painel visual; Larissa definiu que, por ora, a feature expõe apenas endpoints, e um painel seria um projeto separado do time de frontend ([09:39]-[09:40] Marcos/Larissa).

4. **Escala para múltiplos workers e garantia de ordering global.** Com um único worker, a ordem de entrega dos eventos por `order_id` é preservada (processamento em ordem de `created_at`), mas essa garantia se perde caso o sistema escale para múltiplos workers em paralelo no futuro. Diego apontou que, se isso for necessário, seria possível particionar por `order_id` ou usar lock pessimista, mas tratou isso como "problema do futuro", registrado apenas como limitação conhecida e não como decisão arquitetural agora ([09:12]-[09:13] Diego/Larissa/Marcos).

## Impacto e Riscos

| Risco | Impacto | Mitigação proposta |
| --- | --- | --- |
| Cliente fora do ar ou lento durante o disparo do webhook | Poderia travar a transação de mudança de status de outros pedidos se fosse síncrono | Captura assíncrona via outbox, desacoplada da transação de negócio ([ADR-001](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md)) |
| Indisponibilidade planejada e prolongada de um cliente (já observado: 2h de manutenção) | Eventos poderiam ser descartados cedo demais se o retry fosse muito agressivo | Retry com backoff exponencial cobrindo ~15h de janela antes de mover para DLQ ([ADR-004](adrs/ADR-004-retry-com-backoff-exponencial.md)) |
| Vazamento de secret de um cliente (já ocorreu: secret vazada em log de aplicação) | Comprometeria a integração de todos os clientes se a secret fosse global | Secret única por endpoint webhook, com suporte a rotação e grace period de 24h ([ADR-006](adrs/ADR-006-autenticacao-hmac-sha256-com-rotacao-de-secret.md)) |
| Perda da garantia de ordering ao escalar para múltiplos workers | Cliente poderia receber mudanças de status fora de ordem | Aceito como limitação conhecida nesta fase (worker único); mitigação futura via particionamento por `order_id` ou lock pessimista fica em aberto ([09:12]-[09:13] Diego) |
| Bombardeio de chamadas a um cliente com muitas mudanças de status simultâneas | Poderia sobrecarregar o endpoint do cliente | Sem mitigação nesta fase; ponto em aberto para observação futura ([09:38]-[09:39] Diego/Larissa) |

## Decisões Relacionadas

- [ADR-001: Padrão Outbox em MySQL para Captura Transacional de Eventos de Webhook](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md)
- [ADR-002: Consumo do Outbox via Polling (2 segundos)](adrs/ADR-002-consumo-do-outbox-via-polling.md)
- [ADR-003: Worker como Processo Separado](adrs/ADR-003-worker-como-processo-separado.md)
- [ADR-004: Retry com Backoff Exponencial e Limite de 5 Tentativas](adrs/ADR-004-retry-com-backoff-exponencial.md)
- [ADR-005: Dead Letter Queue em Tabela Separada com Replay Administrativo](adrs/ADR-005-dlq-em-tabela-separada-com-replay-administrativo.md)
- [ADR-006: Autenticação de Webhooks via HMAC-SHA256 com Rotação de Secret](adrs/ADR-006-autenticacao-hmac-sha256-com-rotacao-de-secret.md)
- [ADR-007: Entrega At-Least-Once com Deduplicação por Event ID](adrs/ADR-007-entrega-at-least-once-com-dedup-por-event-id.md)
- [ADR-008: Módulo de Webhooks Seguindo Padrões Existentes do Projeto](adrs/ADR-008-modulo-de-webhooks-seguindo-padroes-existentes.md)
