# ADR-005: Entrega Pelo-Menos-Uma-Vez com Idempotência via X-Event-Id

**Status:** Aceito
**Data:** 24-06-2026
**Depends on:** [ADR-001: Padrão Transactional Outbox para Despacho de Eventos Webhook](./ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md)
**Related to:**
- [ADR-002: Estratégia de Retry com Backoff Exponencial e Fila de Mensagens Mortas para Webhooks](./ADR-002-exponential-backoff-retry-com-dead-letter-queue.md)
- [ADR-004: Assinatura de Payload HMAC-SHA256 por Endpoint com Rotação de Segredo](./needs-input/ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md)

---

## Contexto e Declaração do Problema

O sistema de webhooks adota semântica de entrega pelo-menos-uma-vez para notificações de eventos de pedido enviadas a clientes B2B. Falhas de rede, reinicializações de worker e lógica de retry fazem com que o mesmo evento possa ser entregue mais de uma vez ao endpoint do cliente. A questão central é: como permitir que os clientes detectem e descartem entregas duplicadas sem impor ao sistema uma garantia de exatamente-uma-vez, que exigiria coordenação bidirecional com cada sistema externo?

A decisão foi tomada na concepção do projeto (junho de 2026) com base em precedentes da indústria — Stripe e GitHub adotam o mesmo modelo. O Product Manager se comprometeu a documentar o contrato no portal do desenvolvedor, transferindo ao cliente a responsabilidade pela deduplicação.

O contrato de entrega pelo-menos-uma-vez será documentado no portal do desenvolvedor por Marcos (PM) antes da liberação das integrações de produção (`TRANSCRICAO.md:226`). A versão de release específica será definida pelo time de produto.

## Fatores de Decisão

- Exatamente-uma-vez exige two-phase commit ou protocolo de acknowledgment bidirecional, inviável para a escala e maturidade atual do sistema.
- Duplicatas não tratadas pelo cliente causam inconsistências de dados em sistemas externos de clientes B2B.
- O `event_id` UUID deve ser gerado na inserção do outbox e reutilizado em todas as tentativas de retry para garantir estabilidade do identificador.
- Padrões consolidados do setor (Stripe, GitHub) validam a abordagem como contrato aceitável para integrações B2B.
- O contrato externo impacta diretamente as integrações de Atlas Comercial, MaxDistribuição e Nova Cargo.

## Opções Consideradas

1. Entrega pelo-menos-uma-vez com cabeçalho `X-Event-Id` para deduplicação no cliente
2. Entrega exatamente-uma-vez com coordenação bidirecional
3. Entrega no-máximo-uma-vez (fire-and-forget)

## Resultado da Decisão

Opção escolhida: **Entrega pelo-menos-uma-vez com `X-Event-Id`**, porque elimina a complexidade de coordenação exatamente-uma-vez e segue o padrão consolidado da indústria, com responsabilidade de deduplicação explicitamente delegada ao cliente por contrato documentado.

Não há janela de deduplicação definida nesta fase. A recomendação documentada no portal do desenvolvedor deve orientar os clientes a deduplicarem por `event_id` sem assumir uma janela máxima — o `event_id` é estável por evento e não muda entre tentativas de retry.

## Prós e Contras das Opções

### Entrega pelo-menos-uma-vez com `X-Event-Id`

- Pró: Implementação simples no lado do servidor; sem overhead de coordenação.
- Pró: Padrão reconhecido pela indústria; documentação de referência disponível (Stripe, GitHub).
- Pró: O `event_id` estável por evento permite deduplicação confiável pelo cliente.
- Contra: Transfere responsabilidade de deduplicação para cada cliente integrador.

### Entrega exatamente-uma-vez com coordenação bidirecional

- Pró: Elimina por completo o problema de duplicatas no lado do cliente.
- Contra: Requer two-phase commit ou acknowledgment protocol com cada sistema externo.
- Contra: Complexidade desproporcional para a escala atual do projeto.

### Entrega no-máximo-uma-vez (fire-and-forget)

- Pró: Implementação trivial; sem lógica de retry.
- Contra: Risco de perda silenciosa de eventos — inaceitável para notificações de status de pedido.
- Contra: Sem garantia de entrega para integrações B2B críticas.

## Consequências

A adoção do modelo pelo-menos-uma-vez estabelece um contrato permanente com os clientes B2B: qualquer cliente que não implemente idempotência no processamento de webhooks estará sujeito a duplicatas. Este contrato deve ser comunicado de forma clara no portal do desenvolvedor antes da liberação das integrações de produção.

A estabilidade do `event_id` é um invariante crítico do sistema: o identificador deve ser gerado na inserção do registro no outbox e reaproveitado em todas as tentativas de reentrega. Qualquer refatoração do worker de despacho deve preservar esse invariante. Desenvolvedores que alterem o fluxo de retry precisam compreender que gerar um novo `event_id` por tentativa quebra o contrato de idempotência.

Esta decisão é uma restrição permanente na infraestrutura de entrega. Migrar para semântica exatamente-uma-vez exigiria reestruturação significativa do protocolo de entrega e renegociação do contrato com clientes existentes.

## Referências

- `decisions.md` — Decisões #10 (at-least-once + X-Event-Id), #34 (exatamente-uma-vez rejeitado), #20 (lista de cabeçalhos), #21 (estrutura do payload)
- `TRANSCRICAO.md:147-158` — Rationale de Diego, referência Stripe/GitHub, compromisso de Marcos com documentação
