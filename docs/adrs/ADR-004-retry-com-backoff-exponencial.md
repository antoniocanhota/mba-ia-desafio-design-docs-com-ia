# ADR-004: Retry com Backoff Exponencial e Limite de 5 Tentativas

**Status:** Rascunho
**Data:** [A DEFINIR]
**Tags:** retry, backoff, resiliência, DLQ
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Com o worker capaz de consumir o outbox (ADR-002/ADR-003), faltava definir o que acontece quando o cliente está
offline no momento da entrega: *"Vamos pra retry. Se o cliente tá offline, o que a gente faz?"* ([09:14]
Larissa). Diego propôs backoff exponencial: tenta de novo depois de um tempo, aumentando o intervalo
progressivamente, e depois de um teto de tentativas considera falha permanente, movendo o evento para a DLQ
(ADR-005) ([09:15] Diego).

A discussão então girou em torno de quantas tentativas fazer. Diego sugeriu 5, descartando retry indefinido por
deixar eventos pendurados para sempre caso o cliente tenha sumido definitivamente ([09:15] Diego). Bruno propôs
3, por ser "mais agressivo" ([09:16] Bruno). Diego rejeitou 3 citando um precedente real: um cliente já teve uma
indisponibilidade de duas horas por manutenção planejada, e três tentativas em 30 minutos matariam o evento
antes disso ([09:16] Diego).

## Fatores da Decisão

- Clientes podem ter indisponibilidades temporárias e planejadas (já houve caso real de 2 horas de manutenção)
  que o mecanismo de retry precisa tolerar ([09:16] Diego)
- Retry indefinido deixaria eventos pendurados para sempre caso o cliente tenha desaparecido definitivamente
  ([09:15] Diego)
- É preciso um teto que equilibre tolerância a falhas temporárias com a necessidade de eventualmente mover o
  evento para a DLQ ([09:15]-[09:17])

## Opções Consideradas

1. **5 tentativas com backoff 1m/5m/30m/2h/12h** (proposta por Diego)
2. **3 tentativas** (proposta por Bruno, mais agressiva)
3. **Retry indefinido com backoff**

## Decisão Tomada

5 tentativas: *"Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h."* ([09:17] Larissa), com progressão de 1
minuto, 5 minutos, 30 minutos, 2 horas e 12 horas — um total de quase 15 horas entre a primeira falha e a última
tentativa ([09:17] Diego). Marcos considerou a janela aceitável: *"Se um cliente meu cair por 15 horas, ele já
tá com problema sério dele. Acho aceitável."* ([09:17] Marcos).

## Prós e Contras das Opções

### 5 tentativas (1m/5m/30m/2h/12h)
- Bom, porque cobre uma janela de quase 15 horas, tolerando indisponibilidades planejadas já observadas em
  clientes reais ([09:16] Diego)
- Bom, porque foi validada pelo lado de produto como aceitável: uma indisponibilidade de 15 horas já indica
  problema sério do lado do cliente ([09:17] Marcos)
- Ruim, porque ainda assim não é uma garantia perene — eventualmente o evento vai para a DLQ

### 3 tentativas
- Bom, porque falha mais rápido e libera recursos do worker antes
- Ruim, porque mataria o evento em cerca de 30 minutos, insuficiente para cobrir indisponibilidades reais já
  observadas em clientes (2 horas de manutenção planejada) ([09:16] Diego)

### Retry indefinido
- Bom, porque nunca desiste de tentar entregar um evento
- Ruim, porque um evento pode ficar pendurado para sempre se o cliente sumir definitivamente, sem sinalizar
  falha permanente nem liberar o worker desse evento ([09:15] Diego)

## Consequências

Após 5 tentativas falhas (~15 horas de janela total), o evento é considerado falha permanente e movido para a
DLQ (ADR-005), parando de ser retentado automaticamente. A progressão de backoff é fixa (1m/5m/30m/2h/12h) e não
configurável por cliente nesta fase. Aceita-se conscientemente que indisponibilidades de clientes acima de ~15
horas exigirão intervenção manual via replay da DLQ.

## Referências

- TRANSCRICAO.md — [09:14]-[09:15] Larissa/Diego
- TRANSCRICAO.md — [09:15]-[09:16] Bruno/Diego
- TRANSCRICAO.md — [09:16]-[09:17] Diego/Larissa/Marcos
