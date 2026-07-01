# Ata da Reunião Técnica — Sistema de Webhooks de Notificação de Pedidos

**Data:** quinta-feira, 09:00 — 09:53 (~55 min)
**Local:** Call remota (Meet)
**Participantes:**
- Larissa — Tech Lead (conduzindo a reunião)
- Marcos — Product Manager
- Bruno — Engenheiro Pleno, time de Pedidos
- Diego — Engenheiro Sênior, time de Plataforma (entrou às 09:05)
- Sofia — Engenheira de Segurança (saiu às 09:49)

> Fonte: transcrição literal em `TRANSCRICAO.md`. Todos os horários abaixo referenciam trechos daquele arquivo.

---

## 1. Contexto e motivação (09:00–09:02)

Marcos abriu a reunião explicando que três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — pediram formalmente notificação em tempo real de mudança de status de pedidos. Hoje eles fazem polling manual em `GET /orders`, o que está caro e lento para eles. A Atlas sinalizou risco de churn ("migrar pro concorrente") se isso não sair até o fim do trimestre.

Bruno perguntou se "tempo real" era literal ou tinha tolerância. Marcos esclareceu que, para os clientes, **qualquer latência abaixo de 10 segundos** já é aceitável — o requisito real é não precisar mais fazer polling manual.

Sofia perguntou o escopo logo no início: os webhooks são só **outbound** (a plataforma envia para os clientes) — os clientes não enviam nada de volta. Isso simplifica o desenho (sem necessidade de endpoint de recebimento).

**Decisão:** escopo é notificação outbound de mudança de status de pedido, latência-alvo abaixo de 10s.

---

## 2. Mecanismo de disparo: síncrono vs. fila vs. outbox (09:03–09:08)

Larissa levantou a pergunta inicial: o disparo do webhook deveria ser síncrono dentro do `OrderService`, ou via algum mecanismo assíncrono?

Bruno descartou a opção síncrona rapidamente, com dois argumentos:
- A transação de mudança de status já é pesada (atualiza `orders`, insere em `order_status_history`, decrementa `stock_quantity`). Adicionar uma chamada HTTP no meio dela faria um cliente lento travar mudanças de status de *outros* pedidos.
- Se o cliente estiver fora do ar, não há como fazer rollback sensato da mudança de status.

Diego, ao entrar na call, reforçou que síncrono estava fora de cogitação e propôs formalmente o **padrão outbox**: dentro da mesma transação SQL que já atualiza `orders` e `order_status_history`, inserir também uma linha em uma tabela `webhook_outbox`. Um worker separado lê essa tabela e dispara as chamadas HTTP. Isso garante atomicidade: se a transação principal for commitada, o evento existe; se for revertida, o evento desaparece junto — sem inconsistência possível.

Foi cogitada a alternativa de usar Redis Streams (ou fila dedicada similar), mas Diego e Larissa descartaram por ser **overengineering** para um time pequeno — subir infraestrutura nova (ex.: Redis Cluster) só para isso não se justifica quando o MySQL já existente resolve.

Bruno questionou performance da tabela outbox em caso de acúmulo de eventos. Diego respondeu que a tabela terá índice em `status` (pendente, processando, falhou, entregue) e em `created_at`; o worker lê apenas os pendentes em lotes pequenos. Arquivamento de linhas entregues após ~30 dias foi mencionado, mas explicitamente marcado como **fora do escopo desta feature**.

**Decisão fechada:** padrão outbox em MySQL (tabela `webhook_outbox`), inserção na mesma transação da mudança de status. Alternativa de fila externa (Redis) descartada por overengineering.

---

## 3. Worker: modelo de leitura e processo (09:08–09:12)

Diego propôs polling em loop, a cada 2 segundos, buscando os eventos pendentes mais antigos.

Bruno perguntou se não seria melhor usar triggers de banco para algo mais reativo. Diego explicou que MySQL não tem um mecanismo nativo equivalente ao `NOTIFY/LISTEN` do Postgres — um trigger só executa SQL, não notifica processos externos, e improvisar isso (escrever em arquivo, chamar endpoint) seria artificial e "esquisito". Polling de 2s já atende confortavelmente o requisito de latência abaixo de 10s.

Marcos e Larissa concordaram, aceitando explicitamente que a latência mínima no pior caso será de 2 segundos.

Diego trouxe um ponto adicional importante: o worker **precisa rodar como processo separado** da API, não dentro da mesma instância — senão, um restart da API mataria o worker junto. Larissa sugeriu seguir o padrão já existente (`src/server.ts`), criando um `src/worker.ts` novo e um script `npm run worker`. Bruno lembrou que isso implica conectar ao mesmo banco usando o mesmo Prisma, ao que Diego confirmou: mesmo banco e mesma stack, mas processo distinto (e, mais adiante, às 09:29–09:30, ficou explícito que o **PrismaClient também deve ser uma instância separada**, já que é por processo Node).

**Decisão fechada:** worker dedicado (processo separado, `src/worker.ts` / `npm run worker`), polling a cada 2 segundos, PrismaClient próprio conectado ao mesmo banco.

---

## 4. Ordenação de eventos (09:12–09:14)

Larissa perguntou se, havendo mudanças rápidas em sequência (ex.: PAID → PROCESSING → SHIPPED), o cliente receberia na ordem correta.

Diego explicou que isso depende do número de workers: com **um único worker**, o processamento respeita a ordem de `created_at` da outbox, preservando a ordem por pedido. Se no futuro o sistema escalar para múltiplos workers em paralelo, essa garantia se perde — nesse cenário futuro, seria necessário particionar por `order_id` ou usar lock pessimista, mas isso foi explicitamente adiado ("problema do futuro, não agora").

Marcos confirmou que os clientes nunca pediram garantia de ordenação *global*, apenas querem saber quando o pedido deles muda.

**Decisão fechada:** ordenação garantida apenas por `order_id`, e apenas enquanto a arquitetura for single-worker. Documentada como **limitação conhecida**, não como garantia formal do sistema. Escalonamento multi-worker fica fora de escopo.

---

## 5. Retry e backoff (09:14–09:17)

Diego propôs backoff exponencial com teto de tentativas, movendo para DLQ após esgotar.

Houve negociação sobre o número de tentativas:
- Diego sugeriu **5**, argumentando que retry indefinido deixaria eventos pendurados para sempre se o cliente sumisse, e que 5 tentativas cobrem uma janela de 12–24h.
- Bruno contrapôs sugerindo **3**, por ser "mais agressivo".
- Diego rebateu citando um caso real: um cliente já teve indisponibilidade de duas horas por manutenção planejada — com 3 tentativas em 30 minutos, o evento seria descartado prematuramente nesse cenário.

Larissa fechou em 5 tentativas e perguntou a progressão do backoff. Diego propôs: **1 min, 5 min, 30 min, 2h, 12h** — totalizando quase 15h entre a primeira falha e a última tentativa. Marcos considerou aceitável, avaliando que um cliente fora do ar por 15h já teria um problema sério por conta própria.

**Decisão fechada:** 5 tentativas, backoff exponencial 1m / 5m / 30m / 2h / 12h.

---

## 6. Dead Letter Queue (DLQ) (09:17–09:19)

Diego propôs uma tabela separada, `webhook_dead_letter`, contendo payload, motivo da falha e timestamp — em vez de apenas marcar como "failed" na própria outbox. Justificativa: mantém a outbox principal limpa e serve como evidência para debug e reprocessamento.

Bruno perguntou como seria o reprocessamento. Diego propôs um **endpoint admin manual**: `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente (sem reprocessamento automático).

**Decisão fechada:** DLQ em tabela própria (`webhook_dead_letter`); reprocessamento é manual, via endpoint admin de replay.

---

## 7. Segurança: assinatura, secrets e transporte (09:19–09:24) — conduzido por Sofia

Sofia abriu o bloco de segurança levantando a necessidade de o cliente validar que a requisição realmente veio da plataforma e que o payload não foi adulterado.

**Assinatura HMAC:** proposta e fechada como HMAC-SHA256 sobre o corpo do request, enviada no header `X-Signature`. Bruno perguntou o algoritmo especificamente; Sofia justificou SHA-256 como padrão de mercado, amplamente suportado por bibliotecas cliente.

**Secret por endpoint:** Sofia insistiu que cada endpoint de webhook do cliente deve ter uma **secret única**, não uma secret global da plataforma — para que o vazamento de uma não comprometa todos os clientes. Isso implica que a tabela de configuração de webhook armazena `url + secret + customer_id + estado ativo` (confirmado por Bruno).

**Rotação de secret:** deve haver endpoint para o cliente solicitar rotação via API. A secret antiga permanece válida por **24 horas em paralelo** com a nova (grace period), dando tempo de migração. Diego trouxe contexto real: já houve caso de cliente vazando secret em log de aplicação — reforçando a necessidade de rotação fácil.

**TLS obrigatório:** URLs de webhook devem ser HTTPS; cadastro com HTTP é rejeitado por erro de validação. Sofia e Larissa concordaram que isso **não é uma decisão arquitetural**, apenas uma validação de schema (Zod).

**Limite de payload:** Sofia propôs limitar o tamanho do payload e **rejeitar com erro** (não truncar) caso ultrapasse o limite — se chegou a esse tamanho, algo está errado. Diego sugeriu 64KB como teto generoso, considerando que nenhum evento real chegaria perto disso. Larissa também classificou isso como requisito não funcional, não decisão arquitetural separada.

**Decisão fechada:** HMAC-SHA256 (header `X-Signature`), secret única por endpoint com suporte a rotação e grace period de 24h, HTTPS obrigatório (validação de schema), limite de payload de 64KB com erro (não truncamento).

---

## 8. Garantia de entrega e idempotência (09:24–09:26)

Diego declarou que a entrega será **at-least-once**: o cliente pode receber o mesmo evento mais de uma vez e precisa estar preparado para isso.

Para permitir deduplicação do lado do cliente, cada evento carrega um `event_id` (UUID gerado na inserção na outbox) enviado no header `X-Event-Id`.

Sofia observou que isso transfere responsabilidade para o cliente. Diego defendeu a escolha citando que é o padrão de mercado (Stripe, GitHub fazem o mesmo) e que garantir exactly-once exigiria coordenação bilateral, aumentando muito a complexidade — at-least-once com `event_id` resolve a maioria dos casos práticos. Marcos se comprometeu a documentar isso claramente no portal do desenvolvedor.

**Decisão fechada:** garantia at-least-once; deduplicação via `X-Event-Id` é responsabilidade do cliente. Exactly-once foi avaliado e descartado por complexidade desproporcional.

---

## 9. Estrutura de código e padrões reaproveitados (09:27–09:31) — conduzido por Bruno

Bruno propôs seguir o padrão de módulos já existente na codebase (`src/modules/<domínio>` com controller, service, repository, routes, schemas), criando `src/modules/webhooks`. Diego concordou.

Quanto ao worker: entry-point separado em `src/worker.ts`, com a lógica de processamento dentro do módulo (ex.: `webhook.worker.ts` ou `webhook.processor.ts`).

**Erros:** reaproveitar o padrão existente de `AppError` e classes específicas (como já existe para `InsufficientStockError`, `InvalidStatusTransitionError`), com códigos de erro prefixados por `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`).

**Logging e middleware de erro:** reaproveitar Pino (já usado em todo o projeto) e o middleware de erro centralizado existente, que já trata `AppError`, Zod e Prisma — sem necessidade de alterações.

**PrismaClient do worker:** instância separada (é por processo Node), mas apontando para o mesmo `DATABASE_URL`.

**Decisão fechada:** módulo `webhooks` segue integralmente os padrões já estabelecidos no projeto (estrutura de módulos, `AppError`, Pino, error middleware, prefixo de código `WEBHOOK_`). Nenhuma ferramenta ou padrão novo introduzido nesta camada.

---

## 10. Requisitos funcionais de configuração e consulta (09:31–09:37) — conduzido por Marcos

- **Cadastro de webhook:** `POST`, campos `url`, lista de status desejados; `secret` é gerada pela plataforma e devolvida na criação (cliente não define a própria secret).
- Bruno levantou dúvida sobre autenticação: o JWT atual é de usuário operador, não do cliente final. Marcos esclareceu que o cadastro é feito via API autenticada com JWT do sistema interno (usuários que representam o cliente) — **não é o cliente final se autenticando diretamente**. Larissa concluiu que `customer_id` é passado no body/path, e **não** extraído do JWT.
- **Edição/remoção/listagem:** `PATCH` para editar, `DELETE` para remover, `GET` para listar os webhooks de um customer.
- **Filtro de eventos por status:** cada webhook escolhe quais status quer ouvir (ex.: apenas SHIPPED e DELIVERED). Diego perguntou se o filtro é aplicado na inserção na outbox ou no envio; ficou decidido que é **na inserção** — se nenhum webhook do customer quer aquele status, o evento nem é inserido na outbox, economizando armazenamento.
- **Histórico de entregas:** `GET /webhooks/:id/deliveries`, retornando os últimos 100 envios com status (sucesso/falha), payload, response e tempo de resposta.
- **Replay de DLQ:** `POST /admin/webhooks/dead-letter/:id/replay`, exigindo explicitamente **role ADMIN** (não basta ser autenticado) — decisão de Sofia, reforçando que mexer na fila de entrega não é atividade de operador comum, e que o endpoint deve **logar quem executou o replay** para fins de auditoria. Reaproveita o middleware `requireRole` já existente no projeto.
- O restante do CRUD de configuração de webhook (cadastro, edição, remoção, listagem) pode ser feito por **qualquer role autenticada**, por ora — Sofia deixou aberto endurecer isso no futuro.

**Decisão fechada:** CRUD de configuração aberto a qualquer usuário autenticado; endpoint de replay de DLQ restrito a role ADMIN com log de auditoria.

---

## 11. Itens explicitamente descartados ou adiados

Estes pontos foram discutidos mas **não entram no escopo desta fase**:

| Item | Quem levantou | Decisão | Momento |
|---|---|---|---|
| Notificação por e-mail ao cliente quando o webhook dele falha repetidamente | Marcos | **Descartado para esta fase.** Pode voltar em fase futura, após medir impacto. | 09:37–09:38 |
| Rate limiting de envio ao cliente (ex.: 50 pedidos mudando de status em 1 min gerando 50 chamadas simultâneas) | Diego | **Não implementado agora.** Fica como ponto em aberto — "observar e decidir depois" caso vire problema real. | 09:38–09:39 |
| Dashboard visual para o cliente acompanhar seus webhooks | Marcos | **Fora de escopo.** Seria projeto separado do time de frontend; nesta fase só existem endpoints de API. | 09:39–09:40 |
| Escalonamento para múltiplos workers em paralelo / garantia de ordenação global | Bruno, Diego | **Adiado.** Reconhecido como possível necessidade futura (partição por `order_id` ou lock pessimista), mas não faz parte do escopo atual. | 09:12–09:13 |
| Arquivamento de eventos entregues na outbox (após ~30 dias) | Diego | Mencionado como necessário eventualmente, mas **fora do escopo desta feature**. | 09:08 |
| Redis Streams / fila dedicada como alternativa à outbox | Larissa, Diego | **Descartado** por overengineering para o tamanho do time; MySQL existente resolve. | 09:07 |
| Endurecimento de permissões no CRUD de configuração (além do ADMIN no replay) | Sofia | **Adiado** — "por enquanto" qualquer role autenticada pode usar o CRUD. | 09:36–09:37 |

---

## 12. Regras de negócio e detalhes técnicos definidos ao final (09:40–09:53)

- **Integração com o código existente:** a alteração crítica acontece em `OrderService.changeStatus`. A inserção na `webhook_outbox` deve ocorrer **dentro da mesma transação** que já atualiza `orders`, insere em `order_status_history` e atualiza estoque — se a inserção na outbox falhar, toda a transação sofre rollback. Bruno e Diego concordaram que essa é a garantia essencial do design; sem ela, a atomicidade do outbox pattern se perde. (09:40–09:41)
- **Interface de integração:** Bruno propôs uma função pura `publishWebhookEvent(tx, order, fromStatus, toStatus)`, que recebe o client de transação (`tx`) já em curso, em vez de injetar um repository inteiro no `OrderService`. Diego endossou a abordagem. (09:41)
- **Timeout HTTP do worker:** 10 segundos. Cliente que não responde dentro desse prazo é tratado como falha e entra no fluxo de retry. (09:42)
- **Formato do payload:** JSON contendo `event_id`, `event_type` (ex.: `"order.status_changed"`), `timestamp` (ISO 8601), `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e campos básicos do pedido (ex.: `total_cents`). Itens do pedido **não são incluídos** — se o cliente precisar de detalhes, deve consultar `GET /orders/:id`. Isso mantém o payload enxuto. (09:43–09:44)
- **Headers do request de webhook:** `X-Event-Id` (UUID), `X-Signature` (HMAC), `X-Timestamp` (timestamp do envio, permitindo o cliente detectar replay attacks), `Content-Type: application/json`, e `X-Webhook-Id` (id do cadastro de webhook, sugestão de Sofia para clientes com múltiplos endpoints identificarem qual cadastro foi acionado). (09:44–09:45)
- **Snapshot do payload na outbox:** decidido, já ao final da call (com Marcos e Sofia já fora), que o evento armazena o **payload já renderizado** no momento da inserção — não apenas o `order_id` para renderização tardia. Justificativa de Larissa: se o pedido mudar de estado depois, o evento ainda deve refletir o estado exato do momento da mudança, evitando inconsistências. Diego e Bruno concordaram. (09:51–09:52)
- **Formato de ID da outbox:** UUID, seguindo o padrão já usado em todo o projeto (decidido em conversa final entre Larissa e Diego, após o encerramento formal da reunião). (09:50–09:51)

---

## 13. Prazo e organização (09:45–09:50)

Marcos informou que a Atlas quer entrega até o fim de novembro. Larissa estimou o esforço em **três sprints**, decompondo:
- ~1 sprint: modelagem de outbox e DLQ
- ~1 sprint: worker e retry
- ~0,5 sprint: CRUD de configuração e deliveries
- ~0,5 sprint: integração no `OrderService` e testes ponta a ponta
- tempo adicional para HMAC, schemas e validações

Sofia solicitou reservar **pelo menos dois dias úteis** para revisão de segurança antes do deploy, com foco especial em HMAC e geração de secret. Essa revisão foi incorporada ao total de três sprints.

Larissa encerrou anunciando que abrirá o documento de design da feature e marcará uma sessão de revisão com Bruno e Diego antes do início da implementação.

**Decisão fechada:** prazo de três sprints, incluindo revisão de segurança da Sofia ao final.

---

## Resumo das decisões fechadas (confirmado por todos às 09:48)

1. Padrão outbox em MySQL, inserido na mesma transação da mudança de status.
2. Worker dedicado (processo separado), polling a cada 2 segundos.
3. Retry com backoff exponencial: 1m / 5m / 30m / 2h / 12h, total de 5 tentativas, depois DLQ em tabela separada.
4. HMAC-SHA256 sobre o payload, secret única por endpoint, rotação com grace period de 24h.
5. Entrega at-least-once, idempotência via header `X-Event-Id` (responsabilidade do cliente).
6. Reaproveitamento total de padrões existentes: `AppError`, Pino, error middleware, estrutura de módulos, prefixo `WEBHOOK_` nos códigos de erro.
7. CRUD de configuração de webhook aberto a qualquer usuário autenticado; endpoint de replay de DLQ exige role ADMIN e log de auditoria.
8. Notificação por e-mail em caso de falha: adiada para fase futura.
9. Rate limiting de envio: não implementado agora, será observado.
10. Dashboard visual para o cliente: fora de escopo (projeto de frontend separado).
11. Prazo: três sprints, incluindo revisão de segurança.

Adicionalmente, definido em conversa final (após o encerramento formal): IDs da outbox em UUID; payload armazenado como snapshot renderizado no momento da inserção do evento.
