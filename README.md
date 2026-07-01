# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

## Ferramentas de IA utilizadas

* Claude Code - utilizado porque já tenho assinatura

* Plugin project-analizer da Full Cycle - permitiu a geração da documentação de arquitetura atual para entender o projeto

## Workflow adotado

Primeiro fiz uma análise do projeto para enteder o que ele faz. E seguida, pedi para gerar um documento mais legível sobre o que foi discutido na reunião.

## Prompts customizados

### Prompt para geração de documento da transcrição

Usei este prompt para transformar a transcrição bruta em uma ata legível, separando decisões fechadas de pontos descartados/adiados, antes de extrair os documentos formais (PRD, RFC, FDD, ADRs). O resultado está em `docs/others/ata-reuniao-webhooks.md`.

```
O arquivo @TRANSCRICAO.md  contém a gravação literal da reunião técnica. Cinco participantes discutem por aproximadamente 55 minutos no formato [hh:mm] Nome: fala.

A transcrição inclui decisões fechadas, requisitos funcionais explícitos, restrições, ganchos com o código existente, pontos descartados ou adiados para fases futuras e detalhes técnicos secundários. Nem tudo que foi mencionado vira requisito. Algumas coisas foram explicitamente descartadas, outras foram adiadas. Identificar o que NÃO entra é tão importante quanto identificar o que entra. 

Seu objetivo é gerar um documento legível sobre o que foi discutido nessa reunião. Para cada decisão, deixe registrado TUDO que foi discutido até a decisão final. Inclua as pessoas envolvidas e os momentos relevantes sobre a decisão.
```

## Iterações e ajustes

## Como navegar a entrega