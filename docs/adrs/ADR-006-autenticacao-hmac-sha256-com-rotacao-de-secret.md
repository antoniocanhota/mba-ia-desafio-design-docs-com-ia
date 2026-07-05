# ADR-006: Autenticação de Webhooks via HMAC-SHA256 com Rotação de Secret

**Status:** Rascunho
**Data:** 2026-07-04
**Tags:** segurança, HMAC, autenticação, rotação de secret
**Supersedes:** Nenhuma
**Amends:** Nenhuma
**Superseded by:** Nenhuma

## Contexto e Problema

Ao entrar na parte de segurança, Sofia levantou o problema central: os eventos de webhook expõem dados de
pedidos para um endpoint fora da infraestrutura da empresa, e o cliente precisa conseguir validar que a
requisição realmente veio da plataforma e que o payload não foi adulterado no caminho ([09:19] Sofia). Ela
propôs o padrão HMAC: assinar o payload com uma secret compartilhada entre a plataforma e o cliente, enviando a
assinatura num header (`X-Signature`), que o cliente verifica do lado dele ([09:20] Sofia).

Questionada sobre o algoritmo, Sofia especificou HMAC-SHA256 por ser padrão de mercado, com suporte em
bibliotecas de praticamente qualquer cliente ([09:20] Sofia). Em seguida, definiu que cada endpoint de webhook
cadastrado precisa ter uma secret única — não uma secret global da plataforma — para que o vazamento de uma não
comprometa todos os clientes ([09:21] Sofia), reforçado por Diego com um precedente real: um cliente já vazou
uma secret em log de aplicação no passado ([09:22] Diego). Sofia também definiu que a secret precisa ser
rotacionável via API, com a secret antiga permanecendo válida por 24 horas em paralelo à nova, dando tempo do
cliente migrar seus sistemas ([09:21] Sofia).

## Fatores da Decisão

- O cliente precisa validar autenticidade e integridade do payload recebido ([09:19] Sofia)
- Vazamento de uma secret não pode comprometer todos os clientes — precisa ser isolado por endpoint ([09:21]
  Sofia), risco já materializado antes em um vazamento real de secret em log ([09:22] Diego)
- O cliente precisa de uma forma segura de trocar a secret sem interromper a integração ([09:21] Sofia)

## Opções Consideradas

1. **HMAC-SHA256 com secret única por endpoint e rotação com grace period de 24h** (proposta por Sofia)

Não houve um mecanismo concorrente real discutido na reunião: Sofia apresentou HMAC-SHA256 diretamente como
padrão de mercado, e as perguntas do grupo serviram para detalhar a proposta (algoritmo, escopo da secret,
rotação), não para avaliar uma alternativa de autenticação distinta.

## Decisão Tomada

*"Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de
24h."* ([09:22] Sofia).

## Prós e Contras das Opções

### HMAC-SHA256 + secret por endpoint + rotação com grace period de 24h
- Bom, porque é padrão de mercado amplamente suportado por bibliotecas do lado do cliente ([09:20] Sofia)
- Bom, porque isola o impacto de um vazamento de secret a um único endpoint/cliente, em vez de comprometer toda
  a plataforma ([09:21] Sofia)
- Bom, porque permite rotação sem downtime na integração, dando 24 horas para o cliente migrar ([09:21] Sofia)
- Ruim, porque exige gerenciar duas secrets válidas simultaneamente durante a janela de rotação, aumentando a
  complexidade da verificação

## Consequências

A configuração de cada webhook (url, secret, customer_id, estado ativo, [09:21] Bruno/Sofia) precisa suportar
duas secrets simultâneas durante a janela de rotação (atual + anterior, válida por 24h). É necessário um
endpoint para o cliente solicitar rotação da própria secret. A verificação de assinatura no momento da entrega
deve aceitar ambas as secrets enquanto a antiga não expirar. Fica como ponto de atenção para revisão de
segurança dedicada antes do deploy (compromisso assumido por Sofia mais adiante na reunião).

## Referências

- TRANSCRICAO.md — [09:19]-[09:20] Sofia
- TRANSCRICAO.md — [09:20]-[09:21] Bruno/Sofia
- TRANSCRICAO.md — [09:21]-[09:22] Sofia/Diego
