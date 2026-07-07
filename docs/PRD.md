### PRD: OMS Sistema de Webhooks de Notificação de Pedidos

Versão: 1.0
Data: 2026-07-04
Responsável: Antonio Canhota

---

### Resumo

Três clientes B2B do Order Management System (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram para ser notificados em até 10 segundos quando o status de um pedido muda, em vez de continuar fazendo polling manual em `GET /orders`, integração que hoje é lenta e cara para eles ([09:00]-[09:02] Marcos). A Atlas Comercial sinalizou risco de migrar para um concorrente caso a feature não saia até o fim do trimestre ([09:00] Marcos). A solução proposta é um sistema de webhooks outbound: quando o status de um pedido muda, os clientes inscritos naquele status recebem uma notificação HTTP assinada, com garantias de atomicidade, retry e segurança, entregue como um novo módulo dentro do OMS já existente.

---

### Contexto e problema

Onde essa feature será implantada
- Sistema existente: novo módulo (`src/modules/webhooks`) dentro do Order Management System já existente, reaproveitando banco de dados, autenticação e padrões de código já estabelecidos ([RFC.md](RFC.md), [FDD.md](FDD.md))

Problemas priorizados
- Polling repetido em `GET /orders` torna a integração dos clientes B2B lenta e cara, sem números de custo quantificados nesta fase (hipótese não quantificada); prioridade alta, dado o risco concreto de a Atlas Comercial migrar para um concorrente ([09:00] Marcos)
- Não há registro de nenhuma tentativa paliativa anterior a essa proposta

---

### Público-alvo e cenários de uso

Público-alvo
- Clientes B2B que consomem a API do OMS e hoje fazem polling em `GET /orders` (nominalmente Atlas Comercial, MaxDistribuição e Nova Cargo) ([09:00] Marcos)
- Usuários operadores autenticados que cadastram e gerenciam webhooks em nome do cliente ([09:31]-[09:33] Marcos/Bruno/Larissa)
- Administradores que reprocessam manualmente entregas que falharam definitivamente ([09:35]-[09:36] Larissa/Sofia)

Cenários de uso chave
- Cliente B2B cadastra um webhook e passa a receber notificações automáticas de mudança de status, sem precisar mais consultar `GET /orders` repetidamente
- Operador configura, edita ou remove webhooks de um cliente, e consulta o histórico de entregas para diagnosticar problemas de integração
- Administrador reprocessa manualmente um evento que esgotou as tentativas automáticas de entrega

---

### Objetivos e métricas

| Objetivo                                                               | Métrica                                                         | Meta                      |
| ---------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------- |
| Eliminar a necessidade de polling manual pelos clientes B2B             | Latência entre a mudança de status do pedido e a tentativa de entrega do webhook | Abaixo de 10 segundos ([09:02] Marcos) |
| Reduzir a carga de polling sobre a API de pedidos                      | Volume de chamadas a `GET /orders` pelos clientes com webhook ativo | Redução de pelo menos 50% em 60 dias após ativação (hipótese conservadora) |

---

### Escopo

Incluso
- Cadastro, edição, remoção e listagem de webhooks por cliente, com escolha de quais status de pedido deseja receber ([09:31]-[09:34] Marcos/Bruno)
- Entrega de notificação via HTTP assinado quando o status de um pedido muda
- Rotação de secret pelo próprio cliente, com período de transição de 24h ([09:21] Sofia)
- Histórico de entregas por webhook, últimos 100 registros ([09:34] Marcos)
- Reprocessamento manual de eventos que falharam definitivamente, restrito a administradores ([09:18]-[09:19], [09:35]-[09:36])

Fora de escopo
- Notificação de falha persistente por e-mail ao cliente — fica para fase futura ([09:37]-[09:38] Marcos/Larissa)
- Rate limiting de envio ao cliente — apenas observação futura, sem mitigação nesta fase ([09:38]-[09:39] Diego/Larissa)
- Dashboard visual para o cliente acompanhar seus webhooks — projeto separado do time de frontend ([09:39]-[09:40] Marcos/Larissa)
- Garantia de ordem de entrega global entre múltiplos processos de envio simultâneos (só garantida em regime de processo único) ([09:12]-[09:13] Diego)
- Webhooks recebidos pela plataforma (inbound) — escopo é estritamente de saída ([09:02]-[09:03] Sofia/Marcos)
- Arquivamento de eventos já entregues após 30 dias ([09:08] Diego)

---

### Requisitos funcionais

#### FR-001 Cadastro de webhook
Cliente registra um endpoint HTTPS e a lista de status de pedido que quer receber ([09:31] Marcos).

**Fluxo principal**
- Cliente informa URL e lista de eventos desejados
- Sistema valida que a URL é HTTPS
- Sistema gera a secret do webhook
- Sistema retorna a secret em texto claro apenas nesta resposta

**Fluxos alternativos e exceções**
- URL cadastrada não é HTTPS ([09:23] Sofia)

**Erros previstos**
- URL inválida
- Cliente inexistente

**Prioridade:** alta

---

#### FR-002 Edição de webhook
Cliente altera URL, lista de eventos ou ativa/desativa um webhook já cadastrado ([09:33] Bruno).

**Fluxo principal**
- Cliente informa os campos a alterar
- Sistema valida os novos dados
- Sistema atualiza a configuração do webhook

**Fluxos alternativos e exceções**
- Nova URL informada não é HTTPS

**Erros previstos**
- Webhook não encontrado
- URL inválida

**Prioridade:** alta

---

#### FR-003 Remoção de webhook
Cliente remove um webhook cadastrado ([09:33] Bruno).

**Fluxo principal**
- Cliente solicita a remoção do webhook
- Sistema remove a configuração

**Fluxos alternativos e exceções**
- Eventos já entregues ou já em falha permanente não são afetados pela remoção

**Erros previstos**
- Webhook não encontrado

**Prioridade:** média

---

#### FR-004 Listagem de webhooks
Cliente consulta os webhooks cadastrados para ele ([09:33] Bruno).

**Fluxo principal**
- Cliente solicita a lista de seus webhooks
- Sistema retorna os webhooks do cliente, sem expor a secret

**Prioridade:** alta

---

#### FR-005 Rotação de secret
Cliente solicita nova secret sem perder a integração em andamento ([09:21] Sofia).

**Fluxo principal**
- Cliente solicita a rotação da secret
- Sistema gera uma nova secret
- Secret anterior permanece válida por 24h em paralelo
- Após 24h, apenas a nova secret é aceita

**Fluxos alternativos e exceções**
- Motivação de negócio: precedente real de secret vazada em log de aplicação de um cliente ([09:22] Diego)

**Erros previstos**
- Webhook não encontrado

**Prioridade:** alta

---

#### FR-006 Consulta de histórico de entregas
Cliente acompanha os últimos envios feitos para o seu webhook ([09:34] Marcos).

**Fluxo principal**
- Cliente solicita o histórico de entregas de um webhook
- Sistema retorna os últimos 100 registros, com sucesso ou falha, payload, resposta e tempo de resposta

**Prioridade:** média

---

#### FR-007 Entrega de notificação de mudança de status
Quando o status de um pedido muda, o cliente inscrito naquele status recebe uma notificação HTTP assinada ([09:00]-[09:26] diversos).

**Fluxo principal**
- Status do pedido muda
- Sistema identifica os webhooks do cliente inscritos naquele status
- Sistema entrega a notificação assinada com HMAC, com identificador único do evento e do webhook de origem

**Fluxos alternativos e exceções**
- Se a entrega falhar, o sistema tenta novamente algumas vezes antes de desistir ([09:14]-[09:17] diversos)
- Se o webhook estiver desativado ou nenhum webhook estiver inscrito naquele status, nenhuma notificação é enviada

**Erros previstos**
- Entrega falha por indisponibilidade do cliente
- Timeout de resposta do cliente
- Payload do evento acima do limite permitido

**Prioridade:** alta (núcleo da feature)

---

#### FR-008 Reprocessamento administrativo de entregas malsucedidas
Um administrador pode forçar o reenvio de um evento que esgotou as tentativas automáticas ([09:18]-[09:19], [09:35]-[09:36]).

**Fluxo principal**
- Administrador solicita o reprocessamento de um evento específico
- Sistema reenfileira o evento para nova tentativa de entrega
- A ação fica registrada para auditoria

**Fluxos alternativos e exceções**
- Exige papel de administrador ([09:36] Sofia)

**Erros previstos**
- Evento não encontrado
- Usuário sem permissão de administrador

**Prioridade:** média

---

### Requisitos não funcionais

Performance
- Timeout de 10 segundos por tentativa de entrega HTTP ao cliente ([09:42] Diego)
- Latência de notificação abaixo de 10 segundos entre a mudança de status e a tentativa de entrega ([09:02] Marcos)

Disponibilidade
- 99.9% de uptime mensal (hipótese, feature voltada a clientes B2B externos)

Segurança e autorização
- Autenticação HMAC-SHA256 por webhook, com secret única por endpoint ([09:20] Sofia)
- Rotação de secret com grace period de 24h para a secret anterior ([09:21] Sofia)
- URL de destino obrigatoriamente HTTPS ([09:23] Sofia)
- Reprocessamento de falhas restrito a papel ADMIN, com log de auditoria ([09:36] Sofia)
- Secret nunca deve aparecer em log de aplicação ([09:22] Diego)

Observabilidade
- Logs estruturados de cada tentativa de entrega (sucesso ou falha)
- Métricas de taxa de sucesso de entrega por cliente

Confiabilidade e integridade de dados
- Mudança de status do pedido e registro do evento de notificação são atômicos: nunca existe um sem o outro ([09:04]-[09:07] Bruno/Diego)
- Garantia de entrega "at-least-once", com identificador único por evento para o cliente evitar duplicidade ([09:24]-[09:26] Diego)
- Retry com backoff antes de desistir definitivamente de um evento ([09:14]-[09:17] diversos)

Compatibilidade e portabilidade
- Payload de cada evento limitado a 64KB; acima disso o envio é recusado ([09:23]-[09:24] Sofia/Diego/Larissa)

Compliance
- Trilha de auditoria de quem executou reprocessamentos administrativos de entregas ([09:36] Sofia)

Acessibilidade no frontend consumidor
- Não se aplica: a feature é uma API backend sem interface visual direta ao cliente

---

### Arquitetura e abordagem

Abordagem
- Captura assíncrona via padrão outbox na mesma base de dados do OMS existente, processada por um worker dedicado, sem subir infraestrutura nova ([RFC.md](RFC.md), [ADR-001](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md))

Componentes
- Novo módulo `webhooks` dentro do OMS existente, com o CRUD de configuração
- Processo worker separado, responsável por processar e entregar os eventos
- Tabelas de configuração de webhook, outbox de eventos e eventos com falha permanente, no banco já existente

Integrações
- Interna: fluxo de mudança de status de pedidos já existente no OMS
- Externa: endpoint HTTPS de cada cliente B2B que recebe a notificação

### Decisões e trade-offs

#### Decisão: entrega assíncrona via outbox, não síncrona
- **Justificativa:** disparo síncrono acoplaria a disponibilidade de um cliente externo à transação crítica de mudança de status, podendo travar outros pedidos ([09:04] Bruno)
- **Trade-off:** a latência mínima de entrega passa a depender do intervalo de verificação do worker, em vez de ser instantânea ([ADR-001](adrs/ADR-001-padrao-outbox-em-mysql-para-eventos-de-webhook.md), [ADR-002](adrs/ADR-002-consumo-do-outbox-via-polling.md))

#### Decisão: garantir at-least-once, não exactly-once
- **Justificativa:** exactly-once exigiria coordenação complexa entre plataforma e cliente; padrão de mercado (Stripe, GitHub) já resolve com deduplicação do lado do cliente ([09:25] Diego)
- **Trade-off:** o cliente pode ocasionalmente receber o mesmo evento duas vezes e precisa implementar deduplicação do lado dele ([ADR-007](adrs/ADR-007-entrega-at-least-once-com-dedup-por-event-id.md))

#### Decisão: secret única por endpoint, com rotação e grace period de 24h
- **Justificativa:** evita que o vazamento de uma secret comprometa a integração de todos os clientes, precedente real já ocorrido ([09:21]-[09:22] Sofia/Diego)
- **Trade-off:** mais uma responsabilidade operacional para o cliente gerenciar, ao rotacionar a secret quando suspeitar de vazamento ([ADR-006](adrs/ADR-006-autenticacao-hmac-sha256-com-rotacao-de-secret.md))

#### Decisão: 5 tentativas de retry com backoff de até 12h antes de desistir
- **Justificativa:** cobre janelas de indisponibilidade planejada já observadas em clientes, de até 2h de manutenção, sem desistir cedo demais ([09:16] Diego)
- **Trade-off:** um cliente fora do ar pode levar até ~15 horas para ter o evento definitivamente marcado como falha, atrasando a visibilidade do problema ([ADR-004](adrs/ADR-004-retry-com-backoff-exponencial.md))

---

### Dependências

#### Organizacional: revisão de segurança antes do deploy
Sofia reservou pelo menos dois dias úteis para revisar o código de segurança (HMAC e geração de secret) antes de qualquer deploy em produção ([09:46]-[09:47] Sofia/Larissa).

#### Organizacional: documentação no portal de desenvolvedor
Marcos é responsável por documentar no portal de desenvolvedor como os clientes devem integrar (verificação de assinatura, deduplicação por evento) ([09:26], [09:36], [09:40] Marcos).

#### Externa: implementação de verificação e deduplicação pelo cliente
Cada cliente B2B precisa implementar do lado dele a verificação de HMAC e a deduplicação por `X-Event-Id`, já que essa responsabilidade é do cliente, não da plataforma ([09:25] Diego).

#### Técnica: nenhuma infraestrutura nova
A feature reaproveita banco de dados, ORM, logger e padrões de código já existentes no OMS, sem exigir nenhuma dependência técnica nova.

---

### Riscos e mitigação

#### Churn do cliente Atlas se o prazo não for cumprido
- **Probabilidade:** media
- **Impacto:** perda de contrato de um cliente B2B relevante
- **Mitigação:**
  - Prazo estimado de 3 sprints, já incluindo a revisão de segurança da Sofia ([09:45]-[09:47] Larissa/Sofia)
  - Comunicação antecipada com o cliente sobre o progresso ([09:47] Marcos)
- **Plano de contingência:** alinhar novo prazo diretamente com a Atlas caso um atraso seja identificado cedo

#### Indisponibilidade prolongada do endpoint do cliente
- **Probabilidade:** media
- **Impacto:** atraso de até ~15h para o evento ser definitivamente marcado como falha (já observado: cliente com 2h de manutenção planejada, [09:16] Diego)
- **Mitigação:**
  - Retry com backoff exponencial cobrindo essa janela antes de desistir ([ADR-004](adrs/ADR-004-retry-com-backoff-exponencial.md))
- **Plano de contingência:** reprocessamento manual do evento via endpoint administrativo ([ADR-005](adrs/ADR-005-dlq-em-tabela-separada-com-replay-administrativo.md))

#### Vazamento de secret de um cliente
- **Probabilidade:** baixa
- **Impacto:** comprometimento da integração daquele cliente específico, não de todos, pois a secret é por endpoint
- **Mitigação:**
  - Secret única por endpoint, não global ([09:21] Sofia)
  - Rotação de secret disponível a qualquer momento via API ([09:21] Sofia)
- **Plano de contingência:** rotação emergencial da secret e notificação ao cliente afetado

#### Sobrecarga do endpoint do cliente com muitas mudanças de status simultâneas
- **Probabilidade:** media
- **Impacto:** cliente pode ser bombardeado de chamadas em um curto intervalo ([09:38] Diego)
- **Mitigação:**
  - Nenhuma nesta fase, decisão consciente de observar antes de agir ([09:38]-[09:39] Diego/Larissa)
- **Plano de contingência:** implementar rate limiting de saída em fase futura, se confirmado como problema real

---

### Critérios de aceitação
Checklist objetivo que define se a feature está pronta.

- Cliente consegue cadastrar, editar, remover e listar seus webhooks estando autenticado
- A secret do webhook é exibida em texto claro apenas no momento da criação, nunca depois
- Toda mudança de status de pedido gera notificação para os webhooks ativos inscritos naquele status
- Toda notificação entregue é assinada com HMAC-SHA256 e verificável pelo cliente
- Cliente consegue rotacionar a secret sem perder a integração, com a secret anterior válida por 24h
- Cliente consegue consultar os últimos 100 envios feitos para o seu webhook, com sucesso ou falha, resposta e tempo de resposta
- Evento que esgota as tentativas automáticas fica registrado para reprocessamento manual, e só volta a ser tentado por ação explícita de um administrador
- Reprocessamento manual só é aceito para usuários com papel ADMIN e fica registrado para auditoria
- Cadastro de webhook com URL não-HTTPS é recusado
- Notificação de um evento acima de 64KB nunca é enviada
- Latência entre a mudança de status e a primeira tentativa de entrega fica abaixo de 10 segundos
- Reinício do serviço principal da API não interrompe o processamento dos eventos pendentes

---

### Testes e validação

Tipos de teste obrigatórios
- Testes unitários para a lógica crítica de geração/verificação de HMAC, cálculo de backoff e filtro de eventos por status inscrito
- Testes de integração para o fluxo completo de mudança de status gerando evento, entrega simulada, retry e DLQ, incluindo o caso de rollback
- Teste de segurança validando que a secret nunca aparece em log e que a rotação aceita ambas as secrets durante o grace period

Estratégia de validação
- TDD para a lógica crítica de assinatura e retry, QA manual guiado por roteiro para o fluxo de CRUD, e validação exploratória do comportamento fim a fim com um endpoint de teste simulando o cliente B2B
