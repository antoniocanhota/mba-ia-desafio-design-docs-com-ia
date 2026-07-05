# ADR-002: Consumo do Outbox via Polling (2 segundos)

**Status:** Rascunho
**Data:** 2026-07-04
**Tags:** outbox, polling, worker, latência
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Com o padrão outbox definido (ADR-001), a próxima pergunta foi como o worker consome os eventos pendentes na
tabela `webhook_outbox`: *"Tá decidido então: outbox em MySQL. Próximo ponto: como o worker lê isso?"* ([09:08]
Larissa). Diego propôs um loop de polling, buscando os eventos pendentes mais antigos a cada 2 segundos,
processando e marcando como entregues ([09:09] Diego).

Bruno perguntou se não seria possível usar um trigger de banco para ser mais reativo ([09:09] Bruno). Diego
explicou que o MySQL não tem um mecanismo equivalente ao `NOTIFY`/`LISTEN` do Postgres: um trigger de banco só
executa SQL, não consegue notificar um processo externo; para isso seria necessário improvisar algo como
escrever em arquivo ou chamar um endpoint, o que ficaria "esquisito" ([09:09] Diego). O requisito de latência do
cliente (abaixo de 10 segundos, [09:02] Marcos) é atendido com folga por um polling de 2 segundos.

## Fatores da Decisão

- Requisito do cliente: notificação abaixo de 10 segundos já é aceitável como "tempo real" ([09:02] Marcos)
- MySQL não possui mecanismo nativo de notificação para processos externos (sem equivalente ao `NOTIFY`/`LISTEN`
  do Postgres) ([09:09] Diego)
- Um trigger de banco de dados só executa SQL; notificar um processo externo exigiria soluções improvisadas
  (escrita em arquivo, chamada a endpoint) ([09:09] Diego)

## Opções Consideradas

1. **Polling em loop, intervalo de 2 segundos** (proposta por Diego)
2. **Trigger de banco de dados para notificação reativa** (sugerida por Bruno)

## Decisão Tomada

Polling a cada 2 segundos: *"Vamos registrar isso como uma decisão. Worker em polling, 2s. A latência mínima
vai ser 2 segundos no pior caso. Aceitamos."* ([09:10] Larissa). Marcos confirmou que o intervalo atende à
necessidade do cliente ([09:10] Marcos).

## Prós e Contras das Opções

### Polling a cada 2 segundos
- Bom, porque é simples de implementar sobre o MySQL, sem depender de recursos que o banco não oferece
  ([09:09] Diego)
- Bom, porque atende com folga o requisito de latência abaixo de 10 segundos dos clientes ([09:02] Marcos,
  [09:09]-[09:10])
- Ruim, porque impõe uma latência mínima de até 2 segundos no pior caso, por não ser reativo

### Trigger de banco de dados
- Bom, porque poderia disparar o processamento no exato momento da mudança, sem espera de intervalo
- Ruim, porque o MySQL não tem mecanismo nativo para notificar um processo externo a partir de um trigger,
  exigindo soluções improvisadas e frágeis (arquivo, endpoint) ([09:09] Diego)

## Consequências

O worker opera em um loop contínuo, lendo lotes pequenos de eventos pendentes ordenados por `created_at` (índice
já definido em ADR-001) a cada 2 segundos. Isso aceita uma latência mínima conhecida e documentada de até 2
segundos no pior caso, folgada em relação ao requisito real do cliente (10 segundos). Se o requisito de latência
do cliente mudar no futuro (por exemplo, exigir menos de 2 segundos), esta decisão precisará ser revisitada.

## Referências

- TRANSCRICAO.md — [09:02] Marcos
- TRANSCRICAO.md — [09:08] Larissa
- TRANSCRICAO.md — [09:09] Diego/Bruno
- TRANSCRICAO.md — [09:10] Marcos/Larissa
