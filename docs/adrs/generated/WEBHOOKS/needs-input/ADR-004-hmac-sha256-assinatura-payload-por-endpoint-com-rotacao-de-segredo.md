# ADR-004: Assinatura de Payload HMAC-SHA256 por Endpoint com Rotação de Segredo

**Status:** Aceito
**Data:** 2026-06-24
**Depends on:** [ADR-003: Processo Worker Separado com Polling para Consumo do Outbox](./ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md)
**Related to:** [ADR-005: Entrega Pelo-Menos-Uma-Vez com Idempotência via X-Event-Id](../ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md)

---

## Contexto e Problema

O sistema de webhooks envia notificações de eventos para endpoints HTTP registrados por clientes B2B. Sem um mecanismo de assinatura, os consumidores não têm como verificar a autenticidade ou integridade dos payloads recebidos. Um ator malicioso poderia forjar requisições, tornando o sistema vulnerável a injeção de eventos fraudulentos.

O modelo de assinatura precisa balancear segurança (isolamento de comprometimentos entre clientes), simplicidade de integração (os consumidores devem conseguir verificar com bibliotecas padrão) e operabilidade (rotação de segredos sem interrupção de serviço). A decisão foi tomada em 2026-06-24 na reunião de inception do projeto, conduzida pela engenheira de segurança Sofia.

Não há requisitos regulatórios ou contratuais específicos (SOC 2, PCI-DSS) identificados na reunião de inception. A postura de segurança adotada (HMAC-SHA256 por endpoint, TLS obrigatório, revisão de segurança pré-deploy) foi definida pela engenheira de segurança Sofia como suficiente para o escopo atual.

## Fatores de Decisão

- Comprometimento de um único segredo não deve impactar outros clientes
- Consumidores devem conseguir verificar assinaturas com bibliotecas criptográficas padrão
- Rotação de segredos deve ser possível sem interrupção imediata do serviço
- Detecção de replay attacks deve ser viável no lado do consumidor
- TLS obrigatório garante confidencialidade em trânsito, mas não substitui assinatura de payload
- Revisão de segurança é mandatória antes de qualquer deploy em produção

## Opções Consideradas

1. HMAC-SHA256 com segredo único por endpoint
2. Segredo HMAC global da plataforma
3. Assinatura assimétrica (RSA / Ed25519)

## Resultado da Decisão

Opção escolhida: HMAC-SHA256 com segredo único por endpoint, pois isola o raio de explosão de um vazamento para um único cliente e segue o padrão adotado pela indústria (Stripe, GitHub). O valor assinado é o corpo bruto da requisição; a assinatura é transmitida no cabeçalho `X-Signature`. O cabeçalho `X-Timestamp` acompanha cada entrega para permitir detecção de replay attacks pelo consumidor.

O armazenamento do secret em repouso e a política de retorno via GET são decisões de implementação a serem definidas durante o desenvolvimento. A reunião estabeleceu que o secret é gerado pela plataforma no momento do cadastro do webhook e retornado apenas nessa resposta — nunca em consultas GET subsequentes (`TRANSCRICAO.md:184`).

## Prós e Contras das Opções

### HMAC-SHA256 com segredo único por endpoint

- Pró: Vazamento de um segredo não compromete outros clientes
- Pró: Algoritmo amplamente suportado; consumidores integram com bibliotecas nativas
- Pró: Modelo adotado por Stripe e GitHub, reduzindo fricção de integração
- Contra: Aumento de complexidade na gestão de chaves — geração, armazenamento e rotação por linha de webhook

### Segredo HMAC global da plataforma

- Pró: Implementação e gestão operacional significativamente mais simples
- Contra: Um único vazamento compromete todas as integrações de todos os clientes simultaneamente
- Contra: Opção explicitamente rejeitada com base em incidente anterior documentado

### Assinatura assimétrica (RSA / Ed25519)

- Pró: Consumidores verificam com chave pública sem acesso ao segredo
- Contra: Complexidade de implementação e gerenciamento de chaves substancialmente maior
- Contra: Não oferece vantagem operacional suficiente para o caso de uso de webhooks outbound

## Consequências

O worker de entrega deve recuperar o segredo correto do endpoint no momento do despacho. Durante o período de graça de 24 horas após uma rotação, tanto o segredo atual quanto o anterior devem ser válidos para verificação pelo consumidor — o worker envia a assinatura gerada com o segredo atual, e o consumidor pode verificar com qualquer um dos dois. Após 24 horas, o segredo anterior é invalidado.

A máquina de estados da rotação dupla requer duas colunas na tabela `webhooks`: `secret` (segredo atual) e `previous_secret` (segredo anterior, válido por 24h). Um job ou rotina de limpeza invalida `previous_secret` após o período de graça. O mecanismo exato de limpeza é uma decisão de implementação a definir durante o desenvolvimento.

Todo novo desenvolvimento no worker de entrega ou no endpoint de rotação de segredos requer revisão de segurança de 2 dias úteis antes do deploy em produção, conforme reservado por Sofia. Sistemas legados de clientes que não conseguem implementar HMAC-SHA256 não serão suportados; esse trade-off é aceito para manter a postura de segurança.

## Referências

- `decisions.md` — Decisões #7, #8, #9, #20, #33
- `TRANSCRICAO.md:119` — Motivação de Sofia para segredo por endpoint e período de graça
- `src/config/env.ts` — Schema de validação de variáveis de ambiente (Zod)
