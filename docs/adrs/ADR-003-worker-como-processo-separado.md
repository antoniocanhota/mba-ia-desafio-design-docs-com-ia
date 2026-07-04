# ADR-003: Worker como Processo Separado

**Status:** Rascunho
**Data:** [A DEFINIR]
**Tags:** worker, processo, deploy, prisma
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Definido o consumo do outbox via polling (ADR-002), Diego levantou um ponto importante sobre como o worker deve
ser executado: *"O worker tem que rodar como processo separado, não dentro da mesma instância da API. Senão se
a API reinicia, perde o worker."* ([09:11] Diego). Larissa identificou que há espaço para um novo entry point no
projeto, no mesmo estilo do já existente `src/server.ts`, propondo um `src/worker.ts` e um script `npm run
worker` ([09:11] Larissa).

Bruno apontou que, sendo processo separado, o worker precisaria conectar no mesmo banco e usar a mesma stack de
Prisma ([09:11] Bruno). O ponto foi retomado mais adiante, quando Diego perguntou se o worker abriria o mesmo
`PrismaClient` da API ou um separado ([09:29] Diego); Bruno esclareceu que `PrismaClient` é por processo, então
seria uma instância nova, ainda que conectando à mesma `DATABASE_URL` ([09:30] Bruno), o que Diego confirmou
([09:30] Diego).

## Fatores da Decisão

- Se a API reiniciar, o worker não pode cair junto ([09:11] Diego)
- O worker precisa conectar no mesmo banco de dados e usar a mesma stack (Prisma) que a API ([09:11]
  Bruno/Diego)
- `PrismaClient` é uma instância por processo — cada processo Node precisa da sua própria ([09:29]-[09:30]
  Bruno)

## Opções Consideradas

1. **Processo separado com entry point próprio (`src/worker.ts`, `npm run worker`)**
2. **Rodar o worker dentro da mesma instância/processo da API**

## Decisão Tomada

Processo separado: *"Tem espaço pra ser uma entry-point nova no projeto. Tipo o que a gente já tem em
src/server.ts, criar um src/worker.ts e um script 'npm run worker'."* ([09:11] Larissa), com Diego confirmando
que é o mesmo banco e mesma stack, só não pode ser o mesmo processo ([09:11] Diego). O worker abre sua própria
instância de `PrismaClient`, apontando para a mesma `DATABASE_URL` da API ([09:29]-[09:30] Bruno/Diego).

## Prós e Contras das Opções

### Processo separado
- Bom, porque isola o ciclo de vida do worker do da API — um reinício não derruba o outro ([09:11] Diego)
- Bom, porque segue o precedente já existente de entry point dedicado no projeto (`src/server.ts`) ([09:11]
  Larissa)
- Ruim, porque exige uma instância própria de `PrismaClient` e a operação de mais um processo em produção
  ([09:29]-[09:30] Bruno)

### Mesmo processo da API
- Bom, porque não exigiria novo script, entry point ou processo adicional a ser gerenciado
- Ruim, porque acoplaria a disponibilidade do worker à da API — um reinício da API mataria o worker junto
  ([09:11] Diego)

## Consequências

O projeto ganha um novo entry point, `src/worker.ts`, e um script `npm run worker`, análogos ao
`src/server.ts`/`npm run dev` já existentes. O worker mantém sua própria instância de `PrismaClient`, apontando
para a mesma `DATABASE_URL` da API, mas como processo Node independente. Isso adiciona complexidade operacional
(mais um processo para subir, monitorar e reiniciar em produção), compensada pelo isolamento de falhas entre API
e worker.

## Referências

- TRANSCRICAO.md — [09:10]-[09:11] Larissa/Diego
- TRANSCRICAO.md — [09:11] Bruno/Diego
- TRANSCRICAO.md — [09:29]-[09:30] Diego/Bruno
- src/server.ts (entry point existente citado como precedente)
