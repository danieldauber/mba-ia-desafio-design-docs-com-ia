# ADR-003: Processo Worker Separado com Polling para Consumo do Outbox

**Status:** Aceito
**Data:** 24-06-2026
**Depends on:** [ADR-001: Padrão Transactional Outbox para Despacho de Eventos Webhook](../ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md)
**Used by:** [ADR-004: Assinatura de Payload HMAC-SHA256 por Endpoint com Rotação de Segredo](./ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md)
**Related to:** [ADR-002: Estratégia de Retry com Backoff Exponencial e Fila de Mensagens Mortas para Webhooks](../ADR-002-exponential-backoff-retry-com-dead-letter-queue.md)

---

## 1. Contexto e Problema

O sistema de webhooks requer um mecanismo para consumir eventos registrados na tabela `webhook_outbox` e despachá-los via HTTP para sistemas externos. A equipe precisava decidir onde e como esse processamento seria executado: integrado ao processo da API existente ou como processo independente.

O projeto já opera com um único processo Node.js para a API REST. A introdução do processamento de webhooks adicionou uma nova dimensão operacional: um ciclo de vida de execução contínua, independente do ciclo de requisição-resposta da API, com requisito de entrega em menos de 10 segundos declarado pelos clientes Atlas Comercial, MaxDistribuição e Nova Cargo.

A ausência de um mecanismo nativo de notificação reativa no MySQL limitou as opções de arquitetura, tornando o polling a abordagem viável para detectar novos eventos no outbox.

## 2. Fatores de Decisão

- Requisito de entrega de notificações em menos de 10 segundos definido pelos clientes Atlas Comercial, MaxDistribuição e Nova Cargo (`TRANSCRICAO.md:22`)
- Isolamento de ciclo de vida entre API e worker para evitar reinicializações acopladas
- MySQL não oferece mecanismo nativo equivalente ao LISTEN/NOTIFY do PostgreSQL
- Necessidade de instância própria de cliente de banco de dados por processo Node.js
- Simplicidade operacional para um sistema com volume inicial baixo a moderado
- [NECESSITA INFORMAÇÃO: Qual é o requisito formal de SLA para entrega de webhooks em produção?]

## 3. Opções Consideradas

1. **Worker como processo Node.js separado com polling a cada 2 segundos**
2. **Worker embutido no processo da API**
3. **Notificação reativa via mecanismo de trigger no MySQL**

## 4. Resultado da Decisão

Opção escolhida: **Worker como processo Node.js separado com polling a cada 2 segundos**, porque isola o ciclo de vida do processamento de eventos do ciclo de vida da API, satisfaz o requisito de entrega sub-10s com latência máxima de 2 segundos por polling, e é compatível com as limitações nativas do MySQL.

- [NECESSITA INFORMAÇÃO: Qual ferramenta de supervisão de processo foi aprovada para produção — systemd, Docker restart policy ou PM2? Essa decisão afeta diretamente a topologia de deployment.]

## 5. Prós e Contras das Opções

### Worker como processo Node.js separado com polling a cada 2 segundos

- Positivo: Reinicialização da API não interrompe eventos em processamento
- Positivo: Latência máxima de 2s satisfaz o requisito de entrega sub-10s
- Positivo: Compatível com MySQL sem necessidade de infraestrutura adicional
- Negativo: Exige gerenciamento operacional de dois processos em vez de um (Dockerfile, manifests Kubernetes, CI/CD)

### Worker embutido no processo da API

- Positivo: Topologia operacional simples — um único processo para gerenciar
- Positivo: Compartilhamento direto de instância de cliente de banco de dados e configurações
- Negativo: Reinicialização da API por deploy ou falha mata o worker e interrompe entregas em andamento
- Negativo: Acoplamento de ciclos de vida incompatível com requisito de entrega contínua

### Notificação reativa via mecanismo de trigger no MySQL

- Positivo: Latência próxima de zero — worker ativado apenas quando há evento novo
- Positivo: Elimina overhead de polling em períodos de baixa atividade
- Negativo: MySQL não possui mecanismo nativo de notificação assíncrona (sem equivalente ao LISTEN/NOTIFY)
- Negativo: Qualquer workaround (chamada HTTP a partir de trigger, arquivo de sinalização) é um antipadrão operacional

## 6. Consequências

A adoção do worker como processo separado transforma a topologia operacional do sistema: o que antes era um único processo Node.js passa a ser dois processos independentes com ciclos de vida distintos. Todo artefato de deployment — Dockerfile, scripts de inicialização e pipelines de CI/CD — deve contemplar explicitamente o segundo processo.

O intervalo de polling de 2 segundos impõe uma latência mínima de 0 a 2 segundos por evento. Essa latência é aceitável dado o requisito de entrega sub-10s, mas representa um teto de performance para o design atual. A escalabilidade horizontal do worker requer particionamento do outbox por chave de negócio (ex: `order_id`) e execução de múltiplos workers — decisão explicitamente adiada e documentada como limitação conhecida do design de worker único.

[NECESSITA INFORMAÇÃO: Qual é a estratégia definida de monitoramento e alertas para o processo worker em produção — métricas de saúde, thresholds de alerta e responsável por responder a falhas?]

## 7. Referências

- `src/server.ts` — processo da API; o worker seguirá estrutura equivalente
- `src/config/database.ts` — fábrica de cliente de banco de dados; instanciada independentemente pelo worker
- `decisions.md` — Decisões #2, #3, #16, #24, #32
