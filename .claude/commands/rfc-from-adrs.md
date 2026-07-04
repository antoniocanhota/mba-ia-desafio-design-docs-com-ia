---
description: Gera docs/RFC.md a partir de TRANSCRICAO.md e das ADRs já fechadas em docs/adrs/, e atualiza docs/TRACKER.md
argument-hint: "[caminho opcional para a transcrição, padrão: TRANSCRICAO.md]"
---

# Gerar o RFC a partir da transcrição e das ADRs

Você vai consolidar a proposta técnica de uma feature num RFC (`docs/RFC.md`), usando as ADRs já fechadas em
`docs/adrs/` como esqueleto de decisão e a transcrição da reunião como fonte de contexto, alternativas
descartadas e questões em aberto.

Input padrão: `TRANSCRICAO.md` na raiz do repo. Se `$ARGUMENTS` tiver um caminho, use-o no lugar.

Lembrete de altitude — o que é (e o que não é) um RFC: é um documento de deliberação técnica, não uma decisão
final. Ele circula uma proposta para receber comentários, objeções e alternativas antes de uma decisão ser
tomada. Aparece antes da decisão fechada, expõe o raciocínio enquanto ainda há debate, e permite revisão
coletiva em vez de validação tardia. Ele **não registra a decisão definitiva** (isso já está nas ADRs) e **não
detalha implementação** (isso fica no FDD).

## Regras

- Toda alternativa, questão em aberto ou afirmação de contexto precisa ser rastreável a uma fala da transcrição
  no formato `[hh:mm] Nome`, ou a um ADR/arquivo real do código. Nunca invente alternativa, trade-off ou questão
  que não esteja na transcrição.
- Não toque em `src/`, `prisma/`, `tests/`, `docs/adrs/*.md`, `docs/FDD.md`, nem `docs/PRD.md`.
- Disciplina de altitude — o RFC opera em nível de arquitetura, não de decisão pontual nem de implementação:
  - Se uma alternativa rejeitada já é o assunto central de uma ADR inteira (ex.: outbox vs. disparo síncrono vs.
    fila externa), ela entra no RFC.
  - Se uma alternativa rejeitada é só uma variação interna de uma decisão já fechada numa ADR (ex.: "3
    tentativas" dentro da ADR de retry, que já decidiu 5), ela **não** entra como alternativa do RFC — já está
    coberta nos "Prós e Contras das Opções" daquela ADR. Cite a ADR em vez de duplicar o trade-off.
  - Não desça a detalhe de contrato, payload, matriz de erros ou fluxo passo a passo — isso é FDD.
- Todo o conteúdo gerado deve ser em português do Brasil. Mantenha em inglês apenas termos técnicos consagrados
  sem tradução natural (outbox, webhook, backoff, polling, DLQ, at-least-once).

## Passo a passo

1. Leia a transcrição inteira e todas as ADRs em `docs/adrs/`, para saber quais decisões já estão fechadas e
   quais arquivos/trechos de código elas já citam — o RFC vai linká-las, não repeti-las.
2. Identifique autor e revisores: quem se compromete na transcrição a escrever/abrir o documento de design da
   feature (normalmente quem conduz a reunião) é o autor; os demais participantes são os revisores.
3. Levante as alternativas para "Alternativas Consideradas": abordagens de arquitetura inteiras cogitadas e
   descartadas na reunião (mínimo 2), cada uma com o trade-off explícito que motivou o descarte e a ADR que
   registrou a decisão vencedora. Aplique a disciplina de altitude acima para não incluir variações internas de
   uma ADR já fechada.
4. Levante as "Questões em Aberto" (mínimo 2): pontos levantados na reunião e explicitamente não decididos,
   adiados para fase futura, ou deixados como "observar depois", que não viraram nenhuma ADR nem são apenas
   detalhe de FDD.
5. Antes de escrever, releia cada trecho da transcrição citado como referência para alternativas, questões em
   aberto e riscos, e confirme que o resumo bate com o que foi dito, sem inverter quem propôs o quê nem misturar
   falas de momentos diferentes.
6. Escreva `docs/RFC.md` com, no mínimo, estas seções, nesta ordem:

```markdown
# RFC — {Nome da feature}

## Metadados

| Campo | Valor |
| --- | --- |
| **Autor** | {do passo 2} |
| **Status** | Em revisão |
| **Data** | {data da reunião se houver em TRANSCRICAO.md, senão [A DEFINIR]} |
| **Revisores** | {demais participantes do passo 2} |

## Resumo Executivo (TL;DR)

{1 parágrafo: problema, proposta, garantia de entrega/segurança principal, estimativa de prazo se mencionada.}

## Contexto e Problema

{2-3 parágrafos: motivação de negócio citando [hh:mm] Nome, e o desafio técnico central que força a arquitetura
escolhida — ex. onde a mudança de estado acontece hoje no código e por que não pode acoplar a um terceiro.}

## Proposta Técnica

{Visão geral por sub-tópico (captura do evento, processamento, entrega/resiliência, segurança, contrato de
entrega, estrutura de código, superfície de API), cada um linkando a ADR correspondente com
`[ADR-NNN](adrs/ADR-NNN-titulo.md)`. Sem contrato de payload nem matriz de erros — isso é FDD.}

## Alternativas Consideradas

{Uma subseção por alternativa do passo 3, com a alternativa, por que foi descartada e o trade-off explícito,
citando [hh:mm] Nome.}

## Questões em Aberto

{Uma entrada numerada por questão do passo 4, citando [hh:mm] Nome de quem levantou e por que ficou sem
decisão.}

## Impacto e Riscos

{Tabela: Risco | Impacto | Mitigação proposta — cada risco rastreável à transcrição ou a uma ADR.}

## Decisões Relacionadas

- {Link markdown para cada ADR em docs/adrs/, título completo}
```

7. Atualize `docs/TRACKER.md` sem sobrescrever linhas existentes de outros documentos:
   - Uma linha por alternativa, `Documento` = RFC, `Tipo` = "Alternativa Descartada", `Fonte` = `TRANSCRICAO`,
     `Localização` = `[hh:mm] Nome`.
   - Uma linha por questão em aberto, `Tipo` = "Questão em Aberto", mesma convenção de fonte.
   - Se a Proposta Técnica ou os Riscos citarem um caminho de código ainda não coberto no tracker, acrescente uma
     linha com `Fonte` = `CODIGO`.
8. Feche com um resumo para o usuário: quantas alternativas/questões em aberto entraram, quantos ADRs foram
   linkados, e quantas linhas novas entraram no tracker — para ele conferir antes de seguir para o FDD.
