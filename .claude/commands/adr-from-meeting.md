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
   - **Itens descartados ou adiados**: coisas cogitadas mas explicitamente rejeitadas ou empurradas pra uma fase
     futura (ex.: notificação por e-mail, dashboard visual, rate limiting de saída). Esses **não** viram ADR —
     apenas liste-os à parte, com a justificativa, para transparência.
   - **Detalhes não arquiteturais**: o próprio grupo às vezes classifica algo como não sendo decisão
     arquitetural (ex.: exigir HTTPS, limite de tamanho de payload — tratados na reunião como validação de
     schema / requisito não funcional). Não os transforme em ADR; eles pertencem ao FDD.
4. Monte uma lista numerada de **decisões candidatas a ADR** e apresente ao usuário uma tabela assim:

   | # | Título proposto | Resumo em 1 linha | Referência na transcrição |
   |---|---|---|---|

   Logo abaixo, liste em uma segunda tabela os **itens excluídos** (descartados/adiados/não arquiteturais) com
   a justificativa e a referência.

5. Pare e pergunte ao usuário quais números da primeira tabela devem virar ADR (aceite respostas como uma lista
   de números, ou "todas"). **Não gere nenhum arquivo antes dessa resposta.**

---

## Fase 2 — Geração (só após o usuário escolher)

Para cada decisão escolhida, na ordem em que aparecem na transcrição:

1. **Numeração e nome do arquivo**: `docs/adrs/ADR-NNN-titulo-em-kebab-case.md`, `NNN` com 3 dígitos,
   continuando a sequência detectada na Fase 1 (não reutilize nem pule números).
2. **Verifique duplicidade**: se uma decisão equivalente já existir como ADR em `docs/adrs/`, não gere de novo —
   avise o usuário e siga para a próxima.
3. **Preencha o template abaixo.** Headings do corpo ficam em português; são a tradução direta das 7 seções
   MADR (Context and Problem Statement → Contexto e Problema, e assim por diante):

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
     "Decisão", `Fonte` = `TRANSCRICAO`, `Localização` = a citação `[hh:mm] Nome` principal daquela decisão.
   - Acrescente também uma linha por caminho de código citado nas Referências do ADR, com `Fonte` = `CODIGO` e
     `Localização` = o caminho do arquivo.
6. Feche com um resumo para o usuário: tabela com número, título, status e caminho de cada ADR criado, e a lista
   de itens que ficaram de fora (fase 1) para ele confirmar que nada relevante foi perdido.
