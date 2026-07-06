# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

A tarefa pedida neste desafio é a elaboração de design docs, tais como PRD, RFC, FDD e ADR, de uma nova funcionalidade utilizando inteligência artificial. Para isso, foi fornecido o código fonte da aplicação onde a nova feature deveria ser especificada e a transcrição da reunião em que houve a discussão técnica.

## Ferramentas de IA utilizadas

* Claude Code - utilizado porque é a ferramenta que já assino justamente por ser amplamente utilizada em ambientes de desenvolvimento.

* Plugin project-analizer da Full Cycle - permitiu a geração da documentação de arquitetura atual para entender o projeto.

## Workflow adotado

1. **Inicialização do Claude code.** Rodei o comando `/init` para gerar o arquivo `CLAUDE.md`.
2. **Análise da aplicação existente.** Rodei o plugin `project-analizer` sobre `src/` e `prisma/` para entender a arquitetura do OMS antes de escrever qualquer documento — os relatórios ficaram em `docs/others/reports/`. 
3. **Ata legível da reunião.** Com o prompt descrito em "Prompt para geração de documento da transcrição" abaixo, transformei a `TRANSCRICAO.md` bruta num documento legível (`docs/others/ata-reuniao-webhooks.md`), já separando decisão fechada de item descartado/adiado. Esse documento foi gerado para que eu pudesse entender melhora as decisões da reunião.
4. **Extração das ADRs.** Rodei `/adr-from-meeting`: a Fase 1 leu a transcrição `TRANSCRICAO.MD` inteira e levantou 11 decisões candidatas rastreadas a `[hh:mm] Nome`, apresentando também os itens fora de escopo (e-mail de falha, rate limiting, dashboard) e os detalhes não arquiteturais (HTTPS, limite de payload) — sem gerar nenhum arquivo. Depois de eu confirmar quais formalizar, a Fase 2 gerou `docs/adrs/ADR-001` a `ADR-008` no formato MADR e atualizou `docs/TRACKER.md` com a rastreabilidade de cada decisão até a transcrição ou o código.
5. **Consolidação do RFC.** Rodei `/rfc-from-adrs`: leu a transcrição `TRANSCRICAO.MD` inteira e as 8 ADRs já fechadas para não repetir decisão nenhuma e escreveu `docs/RFC.md` com 4 alternativas de arquitetura descartadas na reunião (disparo síncrono, fila externa Redis Streams, trigger de banco, entrega exactly-once) e 4 questões em aberto (e-mail de alerta, rate limiting, dashboard visual, ordering com múltiplos workers), cada uma linkando a ADR correspondente e citando `[hh:mm] Nome`. Também acrescentou as linhas correspondentes em `docs/TRACKER.md`.
6. **Especificação do FDD.** Criei o comando [`.claude/commands/geracao-de-fdd.md`](.claude/commands/geracao-de-fdd.md), invocável como `/geracao-de-fdd`, a partir do prompt de geração de FDD apresentado no curso, adaptando-o pra este desafio (leitura prévia obrigatória de `docs/` inteiro antes de perguntar, e regra de rastreabilidade a `TRANSCRICAO`/ADR/código). Ao rodar, antes de perguntar qualquer coisa ele leu a transcrição inteira, o RFC, as 8 ADRs e o Tracker, e me apresentou, seção por seção do FDD, o que já estava determinado por essas fontes (contexto, objetivos técnicos, escopo, matriz de erros com os 3 códigos `WEBHOOK_*` citados na reunião) pra eu só confirmar ou ajustar. Reservou perguntas reais só pro que nenhuma fonte cobria: caminho de rota do CRUD de webhook (a reunião só definiu os verbos HTTP, não os paths — decidi seguir a convenção `/api/v1/<módulo>` já usada por `order.routes.ts`), como armazenar a secret em repouso (decidi texto claro no banco, já que HMAC precisa do valor recuperável, diferente de hash de senha) e se tracing entrava como hipótese na seção de observabilidade (decidi marcar como fora de escopo, já que o projeto não tem nenhuma infra de tracing hoje). Ao final, escreveu `docs/FDD.md` com as 11 seções exigidas, incluindo a seção obrigatória "Integração com o sistema existente" citando 9 caminhos reais de `src/`/`prisma/`, e atualizou `docs/TRACKER.md`.
7. **Consolidação do PRD.** Usei o prompt de entrevista de PRD disponibilizado no curso, salvo como comando de projeto em [`.claude/commands/geracao-de-prd.md`](.claude/commands/geracao-de-prd.md) e invocável como `/geracao-de-prd`, seguindo a "Ordem de execução sugerida" do enunciado original (PRD por último entre os documentos grandes). Antes de perguntar qualquer coisa, ele leu a transcrição inteira, o RFC, as 8 ADRs, o FDD e o Tracker, e me apresentou etapa por etapa (contexto, problema, escopo, requisitos funcionais espelhando os 8 contratos já fechados no FDD, decisões e riscos) o que já estava determinado por essas fontes, pra eu só confirmar. As perguntas reais ficaram para o que nenhum documento cobria: uma segunda métrica de sucesso além da latência (propus redução de chamadas a `GET /orders`, com meta conservadora sugerida por mim), a meta de disponibilidade (não discutida na reunião, assumi 99.9% como hipótese), e a estratégia de testes (também não discutida, propus TDD para a lógica crítica de HMAC/backoff mais QA manual guiado por roteiro). Ao final, escreveu `docs/PRD.md` com os 8 requisitos funcionais, e atualizou `docs/TRACKER.md` com 41 novas linhas.

## Prompts customizados

### Prompt para geração de documento da transcrição

Usei este prompt para transformar a transcrição bruta em uma ata legível, separando decisões fechadas de pontos descartados/adiados, antes de extrair os documentos formais (PRD, RFC, FDD, ADRs). O resultado está em `docs/others/ata-reuniao-webhooks.md`.

```
O arquivo @TRANSCRICAO.md  contém a gravação literal da reunião técnica. Cinco participantes discutem por aproximadamente 55 minutos no formato [hh:mm] Nome: fala.

A transcrição inclui decisões fechadas, requisitos funcionais explícitos, restrições, ganchos com o código existente, pontos descartados ou adiados para fases futuras e detalhes técnicos secundários. Nem tudo que foi mencionado vira requisito. Algumas coisas foram explicitamente descartadas, outras foram adiadas. Identificar o que NÃO entra é tão importante quanto identificar o que entra. 

Seu objetivo é gerar um documento legível sobre o que foi discutido nessa reunião. Para cada decisão, deixe registrado TUDO que foi discutido até a decisão final. Inclua as pessoas envolvidas e os momentos relevantes sobre a decisão.
```

### Prompt para extração de ADRs a partir da transcrição

Transformei este em um comando de projeto reutilizável, disponível em [`.claude/commands/adr-from-meeting.md`](.claude/commands/adr-from-meeting.md) e invocável como `/adr-from-meeting`. Ele roda em duas fases: primeiro levanta as decisões arquiteturais candidatas da transcrição e apresenta pra eu escolher quais formalizar, e só depois da minha confirmação gera os arquivos em `docs/adrs/` e atualiza `docs/TRACKER.md`.

````markdown
---
description: Extrai decisões arquiteturais de TRANSCRICAO.md, gera ADRs em docs/adrs/ e atualiza docs/TRACKER.md
argument-hint: "[caminho opcional para a transcrição, padrão: TRANSCRICAO.md]"
---

# Extrair ADRs da transcrição da reunião

Você vai identificar decisões arquiteturais fechadas dentro de uma transcrição de reunião técnica e, com a
confirmação do usuário, transformá-las em ADRs formais em `docs/adrs/`. Este comando roda em **duas fases
separadas por um ponto de parada obrigatório**: você nunca escreve arquivo de ADR sem antes o usuário escolher
explicitamente quais decisões formalizar.

Input padrão: `TRANSCRICAO.md` na raiz do repo. Se `$ARGUMENTS` tiver um caminho, use-o no lugar.

## Regras gerais (valem para as duas fases)

- Toda decisão registrada precisa ser rastreável a uma fala da transcrição no formato `[hh:mm] Nome`, ou a um
  arquivo/símbolo real do código em `src/`. Nunca invente decisão, alternativa ou consequência que não esteja na
  transcrição ou no código.
- Não toque em `src/`, `prisma/`, `tests/`, nem em `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`.
- Todo o conteúdo gerado (títulos, corpo, tabela do tracker) deve ser em português do Brasil. Mantenha em inglês
  apenas termos técnicos consagrados sem tradução natural (outbox, webhook, backoff, polling, DLQ) e os rótulos
  de metadados do cabeçalho do ADR (`Status`, `Tags`, `Supersedes`, `Amends`, `Superseded by`), que seguem
  convenção de ferramentas de ADR mesmo em documentos não ingleses.

---

## Fase 1 — Levantamento (não escreve nenhum arquivo)

1. Leia a transcrição inteira.
2. Leia `docs/adrs/` para descobrir os números de ADR já usados (procure o padrão `ADR-(\d+)` no nome de
   qualquer arquivo, mesmo os sem extensão `.md`) — os novos ADRs desta fase começam no próximo número livre.
3. Percorra a transcrição falas por falas e separe:
   - **Decisões fechadas de arquitetura**: pontos em que alguém propõe e o grupo converge (frases como
     "decidido", "fica registrado como decisão", "anotado", ou uma proposta seguida de concordância explícita).
     Uma decisão não precisa ter uma alternativa explicitamente debatida e rejeitada na transcrição para virar
     candidata — decisões de convenção/reuso (ex.: "reaproveitar os padrões já existentes no código" em vez de
     criar um padrão novo para o módulo) contam da mesma forma, mesmo quando a única alternativa é implícita.
     A Fase 2 já permite alternativa "plausível" além de "real discutida" — não descarte na Fase 1 um item só
     porque a transcrição não mostra um debate de opção A vs. opção B.
   - **Itens descartados ou adiados**: coisas cogitadas mas explicitamente rejeitadas ou empurradas pra uma fase
     futura. Para cada um, verifique se ele é uma **alternativa rejeitada de uma das decisões fechadas
     identificadas acima** (ex.: Redis Streams rejeitado em favor do outbox em MySQL; 3 tentativas rejeitadas em
     favor de 5) ou se é **genuinamente independente de qualquer decisão candidata** (ex.: notificação por
     e-mail, dashboard visual, rate limiting de saída — nenhuma delas é alternativa de uma decisão tomada, são
     features à parte que ficaram de fora do escopo). No primeiro caso, associe o item à decisão candidata
     correspondente — ele vai alimentar a seção "Opções Consideradas" / "Prós e Contras" do ADR dela na Fase 2,
     e não deve aparecer como excluído. Só entram na lista de excluídos os itens do segundo caso.
   - **Detalhes não arquiteturais**: o próprio grupo às vezes classifica algo como não sendo decisão
     arquitetural (ex.: exigir HTTPS, limite de tamanho de payload — tratados na reunião como validação de
     schema / requisito não funcional). Não os transforme em ADR; eles pertencem ao FDD.
4. **Verificação cruzada** (antes de apresentar qualquer coisa ao usuário): para cada item candidato a ADR, cada
   alternativa associada e cada item excluído, releia o(s) trecho(s) da transcrição citados como referência e
   confirme que:
   - o resumo descrito bate com o que foi dito, sem parafrasear de um jeito que muda o sentido, sem inverter
     quem propôs ou quem decidiu, e sem misturar falas de momentos diferentes da reunião;
   - a classificação do item (decisão fechada / alternativa associada a uma decisão / excluído por ser
     independente / detalhe não arquitetural) está correta à luz do que o grupo realmente definiu;
   - toda alternativa associada a uma decisão realmente foi discutida e rejeitada *para aquela decisão
     específica*, não para outra.

   Se encontrar alguma divergência, corrija o item (resumo, classificação ou associação) e repita esta
   verificação **apenas para os itens corrigidos**. Limite de **2 iterações de correção por item**. Se, depois
   da 2ª iteração, ainda sobrar divergência não resolvida nesse item específico, pare de tentar corrigi-lo, use
   a melhor versão obtida e marque-o como "⚠️ não confirmado" quando for apresentá-lo ao usuário, com uma nota
   de uma linha explicando o que continua inconsistente.

   **Checklist de cobertura**: se a reunião tiver uma fala de resumo/fechamento (um participante recapitulando
   as decisões antes de encerrar a call), trate essa fala como um checklist independente. Liste cada decisão
   mencionada nela e confirme que cada uma tem um item candidato correspondente na lista montada no passo 3. Se
   faltar alguma, volte ao passo 3 e adicione o item antes de seguir para o passo 5 — não presuma que, por ser
   breve ou sem debate de alternativas, ela não é candidata.

5. Monte uma lista numerada de **decisões candidatas a ADR** e apresente ao usuário uma tabela assim:

   | # | Título proposto | Resumo em 1 linha | Alternativas rejeitadas associadas | Referência na transcrição | Confirmação |
   |---|---|---|---|---|---|

   Logo abaixo, liste em uma segunda tabela apenas os **itens realmente excluídos** (sem decisão candidata
   associada: fora de escopo, adiados para fase futura, ou não arquiteturais) com a justificativa, a referência
   e a mesma coluna de confirmação.

   Na coluna "Confirmação", use "OK" para itens que passaram na verificação do passo 4, ou "⚠️ não confirmado —
   {nota}" para os que ainda restaram divergentes após as 2 iterações.

6. Pare e pergunte ao usuário quais números da primeira tabela devem virar ADR (aceite respostas como uma lista
   de números, ou "todas"). Se houver itens marcados como "⚠️ não confirmado", chame atenção para eles
   explicitamente antes de perguntar. **Não gere nenhum arquivo antes dessa resposta.**

---

## Fase 2 — Geração (só após o usuário escolher)

Para cada decisão escolhida, na ordem em que aparecem na transcrição:

1. **Numeração e nome do arquivo**: `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`, `NNN` com 3 dígitos,
   continuando a sequência detectada na Fase 1 (não reutilize nem pule números).
2. **Verifique duplicidade**: se uma decisão equivalente já existir como ADR em `docs/adrs/`, não gere de novo —
   avise o usuário e siga para a próxima.
3. **Preencha o template abaixo.** Headings do corpo ficam em português; são a tradução direta das 7 seções
   MADR (Context and Problem Statement → Contexto e Problema, e assim por diante). Em "Opções Consideradas" e
   "Prós e Contras das Opções", use as alternativas rejeitadas associadas a esta decisão que você já levantou
   na Fase 1 — não pesquise de novo do zero nem invente alternativa que não tenha aparecido lá.

```markdown
# ADR-NNN: {Título da decisão}

**Status:** Rascunho
**Data:** {AAAA-MM-DD, data da reunião: 2026-06-?? — use a data mencionada em TRANSCRICAO.md se houver, senão marque [A DEFINIR]}
**Tags:** {2 a 5 tags curtas, ex.: outbox, resiliência, segurança}
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

{2-3 parágrafos: qual problema motivou a decisão, citando a fala relevante com [hh:mm] Nome. Cite arquivo/
método real de src/ quando a decisão se conecta ao código existente (ex.: OrderService.changeStatus).}

## Fatores da Decisão

- {bullet curto, um fator por linha, cada um rastreável à transcrição}

## Opções Consideradas

1. **{opção escolhida}**
2. **{alternativa real discutida na reunião, com quem propôs}**
3. **{terceira opção, só se realmente mencionada}**

## Decisão Tomada

{Qual opção venceu e por quê, com a citação da fala que fecha a decisão.}

## Prós e Contras das Opções

### {Opção escolhida}
- Bom, porque {...}
- Ruim, porque {...}

### {Alternativa rejeitada}
- Bom, porque {...}
- Ruim, porque {...} — cite o trade-off que motivou a rejeição, conforme dito na reunião.

## Consequências

{2-3 parágrafos: impacto positivo e negativo, limitações conhecidas assumidas conscientemente (ex.: ordering
só por order_id em single-worker), e o que fica em aberto para revisão futura.}

## Referências

- TRANSCRICAO.md — [hh:mm] Nome (repita uma linha por citação usada acima)
- {caminho real em src/, se aplicável}
```

4. Preencha `Supersedes` / `Amends` / `Superseded by` apenas se houver uma relação real de versionamento entre
   ADRs deste mesmo lote ou com algum ADR pré-existente em `docs/adrs/` — o padrão é "Nenhuma". Relações de
   "assunto relacionado" (sem substituição) vão na seção Referências, não nesses campos.
5. Depois de escrever todos os arquivos, atualize `docs/TRACKER.md`:
   - Garanta que a tabela tenha o cabeçalho exigido:
     `| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |`
   - Acrescente (não sobrescreva linhas existentes de outros documentos) uma linha por ADR novo, `Tipo` =
     "Decisão", `Fonte` = `TRANSCRICAO`, `Localização` = a citação `[hh:mm] Nome` principal daquela decisão. Se
     a decisão aparecer em mais de um momento da reunião (ex.: discutida no meio e depois reafirmada num resumo
     de fechamento), cite o momento em que ela foi de fato **confirmada/fechada**, não o da discussão inicial —
     mesmo que a fala de confirmação seja mais curta que a da discussão.
   - Acrescente também uma linha por caminho de código citado nas Referências do ADR, com `Fonte` = `CODIGO` e
     `Localização` = o caminho do arquivo.
6. Feche com um resumo para o usuário: tabela com número, título, status e caminho de cada ADR criado, e a lista
   de itens que ficaram de fora (fase 1) para ele confirmar que nada relevante foi perdido.
````

### Prompt para consolidação do RFC a partir das ADRs

Também virou comando de projeto, em [`.claude/commands/rfc-from-adrs.md`](.claude/commands/rfc-from-adrs.md), invocável como `/rfc-from-adrs`. Ao contrário do prompt de ADRs, esse não tem fase de confirmação prévia — ele lê as ADRs já fechadas como esqueleto de decisão e usa a transcrição só para contexto, alternativas descartadas e questões em aberto, com uma regra explícita de "disciplina de altitude" pra não deixar variação interna de uma ADR (ex. "3 tentativas" já coberta na ADR de retry) virar alternativa duplicada no RFC.

````markdown
---
description: Gera docs/RFC.md a partir de TRANSCRICAO.md e das ADRs já fechadas em docs/adrs/, e atualiza docs/TRACKER.md
argument-hint: "[caminho opcional para a transcrição, padrão: TRANSCRICAO.md]"
---

# Gerar o RFC a partir da transcrição e das ADRs

Você vai consolidar a proposta técnica de uma feature num RFC (`docs/RFC.md`), usando as ADRs já fechadas em
`docs/adrs/` como esqueleto de decisão e a transcrição da reunião como fonte de contexto, alternativas
descartadas e questões em aberto.

## Regras

- Toda alternativa, questão em aberto ou afirmação de contexto precisa ser rastreável a uma fala da transcrição
  no formato `[hh:mm] Nome`, ou a um ADR/arquivo real do código. Nunca invente alternativa, trade-off ou questão
  que não esteja na transcrição.
- Disciplina de altitude — o RFC opera em nível de arquitetura, não de decisão pontual nem de implementação:
  - Se uma alternativa rejeitada já é o assunto central de uma ADR inteira (ex.: outbox vs. disparo síncrono vs.
    fila externa), ela entra no RFC.
  - Se uma alternativa rejeitada é só uma variação interna de uma decisão já fechada numa ADR (ex.: "3
    tentativas" dentro da ADR de retry, que já decidiu 5), ela **não** entra como alternativa do RFC — já está
    coberta nos "Prós e Contras das Opções" daquela ADR. Cite a ADR em vez de duplicar o trade-off.
  - Não desça a detalhe de contrato, payload, matriz de erros ou fluxo passo a passo — isso é FDD.

## Passo a passo (resumo)

1. Leia a transcrição inteira e todas as ADRs em `docs/adrs/` — o RFC vai linká-las, não repeti-las.
2. Identifique autor (quem se compromete a abrir o doc de design) e revisores (demais participantes).
3. Levante as alternativas para "Alternativas Consideradas" (mínimo 2): abordagens de arquitetura inteiras
   cogitadas e descartadas na reunião, cada uma com o trade-off explícito e a ADR que registrou a decisão
   vencedora.
4. Levante as "Questões em Aberto" (mínimo 2): pontos levantados e explicitamente não decididos, adiados, ou
   deixados como "observar depois", que não viraram ADR nem são apenas detalhe de FDD.
5. Antes de escrever, releia cada trecho citado e confirme que o resumo bate com o que foi dito, sem inverter
   quem propôs o quê nem misturar falas de momentos diferentes.
6. Escreva `docs/RFC.md` com Metadados, Resumo Executivo, Contexto e Problema, Proposta Técnica (linkando
   `[ADR-NNN](adrs/ADR-NNN-titulo.md)` por sub-tópico), Alternativas Consideradas, Questões em Aberto, Impacto e
   Riscos, e Decisões Relacionadas.
7. Atualize `docs/TRACKER.md` sem sobrescrever linhas existentes: uma linha por alternativa (`Tipo` = "Alternativa
   Descartada"), uma por questão em aberto (`Tipo` = "Questão em Aberto"), e uma por caminho de código novo
   citado na Proposta Técnica ou nos Riscos.
8. Feche com um resumo: quantas alternativas/questões entraram, quantos ADRs foram linkados, quantas linhas
   novas entraram no tracker.
````

## Iterações e ajustes

Cheguei ao `/adr-from-meeting` acima depois de 4 iterações, todas motivadas por problemas concretos encontrados ao revisar (não ao rodar às cegas):

1. **Alternativas rejeitadas se perdendo.** Itens como "Redis Streams" (rejeitado em favor do outbox em MySQL) ou "3 tentativas" (rejeitado em favor de 5) estavam caindo direto na lista de itens excluídos, em vez de alimentar a seção "Opções Consideradas" / "Prós e Contras" do ADR da decisão correspondente. Adicionei uma etapa que associa cada alternativa rejeitada à decisão candidata a que ela pertence.
2. **Risco de alucinação sem verificação.** Não havia nenhum passo que checasse se o resumo apresentado ao usuário batia de fato com a transcrição. Adicionei uma verificação cruzada obrigatória antes de exibir qualquer tabela, com limite de 2 iterações de correção por item e um marcador "⚠️ não confirmado" para divergências que não fecham nem depois disso.
3. **Decisão real ficando de fora por falta de "debate explícito".** Ao rodar contra a transcrição, percebi que a decisão de "reaproveitar os padrões já existentes do projeto" (reafirmada por Larissa no resumo de fechamento, `[09:48]`) não aparecia como candidata — o prompt estava implicitamente exigindo um debate de "opção A vs. opção B" pra considerar algo decisão, e decisões de convenção/reuso não têm esse formato. Removi esse viés e acrescentei uma checklist de cobertura que usa a fala de resumo/fechamento da reunião como lista de conferência.
4. **Tracker apontando pro momento errado.** Quando uma decisão era discutida no meio da reunião e só reafirmada no resumo final, o tracker citava a discussão inicial em vez da confirmação. Ajustei a regra de preenchimento do `docs/TRACKER.md` para sempre priorizar a fala que efetivamente fecha a decisão.
5. **Hipótese de rota sem fonte, entrando como se fosse fato.** Ao especificar os contratos públicos do FDD, a IA propôs caminhos concretos para o CRUD de webhook (`/api/v1/webhooks/...`) misturados com os dois caminhos que a reunião realmente cita (`GET /webhooks/:id/deliveries`, `POST /admin/webhooks/dead-letter/:id/replay`), sem distinguir claramente o que veio da fala do que era inferência de convenção. Pedi pra remover as hipóteses de rota da narrativa de fluxo e resolvi a lacuna com uma pergunta direta em vez de deixar a IA assumir silenciosamente — mesmo padrão que apliquei para armazenamento da secret e para tracing na seção de observabilidade.
6. **Tracker abaixo do limiar de rastreabilidade.** Depois de fechar o FDD, pedi uma auditoria contra os critérios de aceite do `README.original.md` e a IA encontrou que a proporção de linhas com Fonte = `TRANSCRICAO` tinha caído para 68,5% (o mínimo exigido é 70%), porque a seção de Integração adicionou várias linhas `CODIGO` de uma vez. Em vez de inflar a tabela artificialmente, a correção certa foi acrescentar linhas que já deveriam existir por conteúdo (fluxos de "processamento pelo worker", "retry" e "DLQ" como itens próprios do FDD, e a restrição de escopo outbound-only), o que também fechou uma lacuna real de cobertura dos "quatro momentos de fluxo" exigidos pelo enunciado.
7. **Valor de `Fonte` fora do formato exigido.** Ao gerar o PRD, a skill de entrevista registrou no Tracker `Fonte = ENTREVISTA` para 5 itens que só existiam porque eu decidi assim durante a entrevista (sem origem na reunião nem no código): meta de redução de chamadas a `GET /orders`, disponibilidade de 99.9%, observabilidade, acessibilidade e ausência de infraestrutura nova. Uma auditoria posterior contra `README.original.md` mostrou que o enunciado define a coluna `Fonte` como tendo só dois valores possíveis, `TRANSCRICAO` ou `CODIGO` — `ENTREVISTA` era uma convenção própria da skill, não do desafio. Removi as 5 linhas do Tracker (os fatos permanecem no PRD, marcados como hipótese no texto) e renumerei os IDs restantes pra não deixar gaps.

## Como navegar a entrega

- `TRANSCRICAO.md`
- `docs/adrs/ADR-001-*.md` a `ADR-008-*.md`
- `docs/RFC.md`
- `docs/FDD.md`
- `docs/PRD.md`
- `docs/TRACKER.md`

Ordem sugerida de leitura: `TRANSCRICAO.md` → `docs/adrs/` → `docs/RFC.md` → `docs/FDD.md` → `docs/PRD.md` → `docs/TRACKER.md`.
