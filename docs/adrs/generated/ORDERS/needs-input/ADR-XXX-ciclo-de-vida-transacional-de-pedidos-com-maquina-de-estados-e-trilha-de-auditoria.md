# ADR-XXX: Ciclo de Vida Transacional de Pedidos Orientado por Máquina de Estados com Trilha de Auditoria

**Status:** Proposto
**Data:** 24-06-2026
**ADRs Relacionadas:** ADR-XXX (versão em inglês desta mesma decisão, módulo ORDERS)

## Contexto e Declaração do Problema

O sistema de gestão de pedidos precisa garantir que o status de um pedido, seu efeito sobre o estoque de produtos e seu histórico de auditoria permaneçam consistentes mesmo sob acesso concorrente ou falha parcial. A criação de pedidos e as transições de status são as duas operações de escrita mais críticas do sistema, afetando simultaneamente dados de pedido, item de pedido, estoque e histórico de auditoria; erros aqui se traduzem diretamente em contagens de estoque corrompidas ou mudanças de estado não rastreáveis.

O sistema também precisa gerar números de pedido sequenciais e legíveis por humanos (por exemplo, `ORD-000001`) no momento da criação, o que introduz um ponto compartilhado de contenção entre requisições concorrentes de criação de pedido. Como esse padrão governa os dois endpoints que alteram pedidos (criação e mudança de status) e alcança validação de estoque de produto e de cliente, é a lógica de maior consequência entre módulos em todo o domínio de gestão de pedidos, e qualquer engenheiro que estenda o comportamento de pedidos precisa entender suas garantias antes de adicionar novas transições ou efeitos colaterais.

Como devem ser estruturadas as escritas do ciclo de vida do pedido de modo que as transições de status sejam sempre válidas, os efeitos de estoque nunca sejam aplicados sem uma mudança de status correspondente já confirmada, cada transição seja auditável e os números de pedido permaneçam sequenciais, mantendo a implementação simples o suficiente para uma equipe pequena raciocinar e evoluir com segurança?

## Fatores da Decisão

- Débitos/reposições de estoque nunca podem ser confirmados sem uma transição de status válida correspondente, e vice-versa.
- Toda criação de pedido e mudança de status deve ser atribuível a um usuário e mantida para rastreabilidade.
- Somente transições de estado válidas devem poder ser persistidas; transições inválidas devem ser rejeitadas antes de qualquer efeito colateral ocorrer.
- A solução deve evitar a introdução de uma dependência externa de motor de workflow enquanto o sistema e seu volume de pedidos ainda são recentes.
- Os números de pedido devem ser previsíveis e sequenciais para fins de identificação posterior.

## Opções Consideradas

1. **Máquina de estados finita em processo, com uma única transação de banco de dados multi-tabela, tabela de auditoria embutida e numeração de pedidos via tabela de sequência** (escolhida)
2. **Transições impostas pelo banco de dados (constraints/triggers) com identificador de pedido baseado em `AUTO_INCREMENT`**
3. **Ciclo de vida desacoplado/orientado a eventos**, no qual mudanças de status, efeitos de estoque e registro de auditoria são tratados como efeitos colaterais assíncronos e eventualmente consistentes, em vez de uma única transação

## Decisão Tomada

Opção escolhida: "Máquina de estados finita em processo com uma única transação multi-tabela, tabela de auditoria embutida e numeração via tabela de sequência", porque oferece a garantia de correção mais forte e mais simples disponível sem dependências externas: cada mudança de status é validada contra uma tabela de transições declarativa antes de ser persistida, e a transição, seus efeitos colaterais de estoque e sua linha de histórico de auditoria são confirmados atomicamente em uma única transação de banco de dados. Isso torna estruturalmente impossível que o estoque seja ajustado sem uma mudança de status correspondente já confirmada, ou que uma mudança de status exista sem um registro de auditoria correspondente.

Essa abordagem troca concorrência de escrita por consistência: a transação mantém locks de linha (notadamente na sequência de numeração de pedidos e nas linhas de produto tocadas) durante toda a sua duração, o que serializa a criação de pedidos no nível do banco de dados. Essa troca é tratada como aceitável para o volume de pedidos atual e ainda não comprovado do sistema, e espera-se que seja revisitada caso os requisitos de throughput sejam esclarecidos.

[ENTRADA NECESSÁRIA: Qual é o throughput esperado de criação de pedidos, e a partir de qual volume a sequência de numeração de pedidos de linha única se torna um gargalo inaceitável?]

## Prós e Contras das Opções

### Opção 1: FSM em processo + transação única + tabela de auditoria embutida (escolhida)

- Bom, porque transições de estado inválidas são rejeitadas antes de qualquer escrita ocorrer, prevenindo uma classe inteira de bugs de corrupção de dados.
- Bom, porque efeitos de estoque e histórico de auditoria são garantidamente consistentes com o status persistido do pedido, mesmo sob acesso concorrente ou falha.
- Bom, porque não possui dependência externa (motor de workflow, message broker) e é simples de ler, auditar e estender para uma equipe pequena.
- Ruim, porque a reserva de número de pedido via tabela de sequência força a criação de pedidos a ser efetivamente single-writer, criando um teto de escalabilidade conforme o volume cresce.

### Opção 2: Transições impostas pelo banco de dados + identificador `AUTO_INCREMENT`

- Bom, porque elimina completamente o ponto de contenção de lock de linha da tabela de sequência, melhorando a concorrência de escrita na criação de pedidos.
- Bom, porque as regras de transição passam a ser impostas pelo próprio banco de dados, independentemente da correção do código da aplicação.
- Ruim, porque constraints/triggers são mais difíceis de testar, versionar e raciocinar do que uma tabela declarativa em nível de aplicação, e ocultam regras de negócio fora do código-fonte.
- Ruim, porque números de pedido `AUTO_INCREMENT` não são garantidamente sem lacunas/sequenciais da mesma forma sob rollback de transações, o que pode conflitar com expectativas de negócio quanto à numeração de pedidos.

### Opção 3: Ciclo de vida desacoplado/orientado a eventos

- Bom, porque remove a fronteira de transação de longa duração, permitindo que mudanças de status, efeitos de estoque e registro de auditoria escalem de forma independente.
- Bom, porque acomoda naturalmente futuras integrações com sistemas externos (gateways de pagamento, transportadoras) sem estender a transação.
- Ruim, porque introduz consistência eventual, o que significa que estoque, status e estado de auditoria podem divergir temporariamente, exigindo lógica de reconciliação adicional hoje inexistente.
- Ruim, porque exige infraestrutura (barramento de eventos, consumidores idempotentes) atualmente não presente no sistema, adicionando complexidade operacional desproporcional às necessidades atuais.

## Consequências

Como as transições de status, os efeitos de estoque e o registro de auditoria são validados e confirmados em conjunto, engenheiros que adicionarem um novo status de pedido ou um novo efeito colateral vinculado a uma transição devem atualizar a tabela de transições e manter qualquer novo efeito colateral dentro da mesma fronteira transacional existente; deixar de fazer isso reintroduziria silenciosamente o risco de corrupção de dados que este design existe para prevenir. Isso torna a fronteira transacional e a validação da FSM contexto essencial para qualquer trabalho futuro em `order.service.ts` ou `order.status.ts`.

A trilha de auditoria é hoje uma tabela relacional simples, transacionalmente consistente, lida junto com o próprio pedido, e não um log de eventos de propósito geral. Se necessidades de relatórios, conformidade ou replay de estado crescerem além de uma simples consulta cronológica, a estrutura dessa tabela precisará ser revisitada.

Manter locks de linha na sequência de numeração de pedidos e nas linhas de produto tocadas durante toda a duração de cada transação significa que o throughput de criação de pedidos é limitado pela contenção single-writer nessa sequência. Essa é uma troca aceita e deliberada hoje, mas também é a primeira restrição que precisará ser revisitada caso o sistema caminhe para maior volume de pedidos, integrações assíncronas com sistemas externos (gateways de pagamento, transportadoras) ou uma decomposição orientada a eventos/microsserviços do domínio de gestão de pedidos.

[ENTRADA NECESSÁRIA: A numeração sequencial/sem lacunas de pedidos (`ORD-000001`) é um requisito rígido de negócio ou compliance, ou um identificador não sequencial pode ser adotado caso a tabela de sequência se torne uma restrição de escalabilidade?]

[ENTRADA NECESSÁRIA: Fluxos de pedido paralelos ou ramificados (ex.: envios parciais, devoluções/reembolsos) estão planejados, o que exigiria redesenhar a atual tabela linear de transição de estados?]

[ENTRADA NECESSÁRIA: A trilha de auditoria precisa evoluir para um mecanismo de nível de compliance ou replayable de event sourcing, ou uma tabela transacional de histórico é suficiente para o futuro previsível?]

## Referências

- `src/modules/orders/order.service.ts:50-124` - transação de criação de pedido (`create`)
- `src/modules/orders/order.service.ts:126-179` - transação de transição de status (`changeStatus`)
- `src/modules/orders/order.service.ts:245-254` - reserva de número via `reserveOrderNumber`
- `src/modules/orders/order.status.ts:3-37` - tabela declarativa de transições e predicados de efeito sobre estoque
- `prisma/schema.prisma:74-138` - modelos `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`
