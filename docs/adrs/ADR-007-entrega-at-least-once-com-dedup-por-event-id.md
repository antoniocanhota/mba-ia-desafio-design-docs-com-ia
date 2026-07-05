# ADR-007: Entrega At-Least-Once com Deduplicação por Event ID

**Status:** Rascunho
**Data:** 2026-07-04
**Tags:** at-least-once, idempotência, contrato de API
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Diego trouxe uma garantia importante sobre a semântica de entrega: *"A gente vai garantir at-least-once. Pode
acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado."* ([09:24] Diego). Bruno
perguntou como o cliente diferenciaria uma entrega duplicada de um evento novo ([09:25] Bruno). Diego propôs
enviar um `event_id` (UUID gerado quando o evento entra na outbox, ver ADR-001) no header `X-Event-Id`, único
por evento, para que o cliente dedupique do lado dele ([09:25] Diego).

Sofia observou que isso transfere a responsabilidade de deduplicação para o cliente ([09:25] Sofia). Diego
defendeu que essa é a prática de mercado — Stripe e GitHub fazem da mesma forma — e que garantir exactly-once
exigiria coordenação entre os dois lados, tornando a solução muito mais complexa para resolver um problema que
at-least-once com `event_id` já cobre na prática ([09:25] Diego). Marcos se comprometeu a documentar isso de
forma destacada no portal do desenvolvedor ([09:26] Marcos).

## Fatores da Decisão

- Garantir exactly-once exigiria coordenação entre os dois lados, aumentando muito a complexidade de
  implementação ([09:25] Diego)
- O padrão de mercado (Stripe, GitHub) já resolve entregas duplicadas com deduplicação do lado do cliente via
  identificador único ([09:25] Diego)
- É necessário um identificador estável e único por evento para o cliente conseguir dedupicar ([09:25] Diego)

## Opções Consideradas

1. **At-least-once com `X-Event-Id` para deduplicação do lado do cliente** (proposta por Diego)
2. **Exactly-once delivery**

## Decisão Tomada

*"Beleza. At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão."* ([09:26] Larissa).

## Prós e Contras das Opções

### At-least-once + X-Event-Id
- Bom, porque é significativamente mais simples de implementar do lado do servidor ([09:25] Diego)
- Bom, porque segue um padrão já validado por players de mercado como Stripe e GitHub ([09:25] Diego)
- Ruim, porque transfere a responsabilidade de deduplicação para o cliente ([09:25] Sofia) — mitigado por
  documentação clara no portal do desenvolvedor ([09:26] Marcos)

### Exactly-once
- Bom, porque eliminaria duplicatas sem exigir tratamento adicional do lado do cliente
- Ruim, porque exigiria coordenação entre os dois lados (algo próximo de um commit distribuído), aumentando
  muito a complexidade de implementação para o ganho obtido ([09:25] Diego)

## Consequências

Todo evento carrega um `event_id` (UUID) gerado no momento da inserção na outbox (ADR-001), enviado no header
`X-Event-Id` em toda tentativa de entrega, incluindo retries (ADR-004). O cliente é responsável por implementar
sua própria deduplicação com base nesse identificador; essa responsabilidade precisa estar documentada de forma
clara e destacada no portal do desenvolvedor. Fica assumido conscientemente que a plataforma não garante
exactly-once.

## Referências

- TRANSCRICAO.md — [09:24] Diego
- TRANSCRICAO.md — [09:25] Bruno/Diego/Sofia
- TRANSCRICAO.md — [09:26] Marcos/Larissa
