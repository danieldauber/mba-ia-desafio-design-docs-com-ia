# RFC-001: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Status** | Proposto |
| **Autor** | Larissa (Tech Lead) |
| **Data** | 2026-06-24 |
| **Revisores** | Diego (Eng. Sênior, Plataforma), Bruno (Eng. Pleno, Pedidos), Sofia (Eng. Segurança), Marcos (PM) |
| **ADRs relacionados** | [ADR-001](./adrs/ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md), [ADR-002](./adrs/ADR-002-exponential-backoff-retry-com-dead-letter-queue.md), [ADR-003](./adrs/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md), [ADR-004](./adrs/ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md), [ADR-005](./adrs/ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md) |

---

## TL;DR

Substituir o polling de `GET /orders` feito pelos clientes B2B por notificações outbound via webhook. A plataforma passa a empurrar eventos de mudança de status de pedido para URLs registradas pelos clientes, com garantia de entrega at-least-once, assinatura HMAC-SHA256 por endpoint e resiliência via Transactional Outbox + retry com backoff exponencial. Prazo: 3 sprints.

---

## Contexto e Problema

Três clientes B2B — Atlas Comercial, MaxDistribuição e Nova Cargo — integram suas operações com o Order Management System e precisam acompanhar em tempo real as mudanças de status dos pedidos. Hoje fazem isso por polling periódico no `GET /orders`, o que gera latência variável, carga desnecessária na API e fricção operacional do lado deles.

A Atlas Comercial sinalizou que a ausência de notificações em tempo real é critério de avaliação de continuidade do contrato.

O requisito de negócio é direto: qualquer mudança de status de pedido deve chegar ao sistema do cliente em menos de 10 segundos. Abaixo desse limiar, os clientes consideram a entrega "tempo real" para seus fluxos operacionais.

O sistema não possui nenhum mecanismo de notificação externa. Esta RFC propõe a arquitetura da solução e documenta as alternativas descartadas e as questões ainda em aberto.

---

## Proposta Técnica

### Visão geral

A solução tem três componentes principais:

**1. Módulo de configuração de webhooks** (`src/modules/webhooks/`)
Expõe endpoints REST para clientes B2B cadastrarem, editarem e removerem endpoints de webhook. Cada endpoint armazena a URL de destino, a secret HMAC, a lista de status de interesse e o estado ativo/inativo. Segue o padrão de módulo já estabelecido no projeto (controller / service / repository / routes / schemas).

**2. Inserção no outbox dentro da transação de pedido**
Quando `OrderService.changeStatus` é chamado, a mesma transação Prisma que atualiza o status, registra o histórico e ajusta o estoque também insere o evento na tabela `webhook_outbox` via `publishWebhookEvent(tx, order, fromStatus, toStatus)`. O payload é serializado como snapshot no momento da inserção — o evento reflete o estado exato da mudança, independente de atualizações posteriores no pedido.

**3. Worker de despacho** (`src/worker.ts`)
Processo Node.js separado que faz polling na `webhook_outbox` a cada 2 segundos, recupera eventos pendentes em batch, assina o payload com HMAC-SHA256 usando a secret do endpoint de destino e executa a chamada HTTP. Falhas entram em retry com backoff exponencial; eventos que esgotam todas as tentativas vão para `webhook_dead_letter`.

### Fluxo de ponta a ponta

```
OrderService.changeStatus()
  └─ $transaction {
       atualiza status do pedido
       registra order_status_history
       ajusta stock_quantity
       insere snapshot do evento em webhook_outbox   ← novo
     }

worker (processo separado, polling a cada 2s)
  └─ lê eventos PENDING do outbox em batch
       └─ para cada evento:
            └─ filtra webhooks ativos do customer que inscreveram aquele status
                 └─ para cada webhook:
                      └─ assina payload com HMAC-SHA256 (secret do endpoint)
                           └─ POST para URL do cliente (timeout 10s)
                                ├─ 2xx → DELIVERED
                                └─ outros → retry com backoff
                                     └─ após 5 falhas → webhook_dead_letter
```

### Contrato de entrega

- **Semântica**: at-least-once. O cliente pode receber o mesmo evento mais de uma vez em cenários de falha de rede ou restart do worker.
- **Idempotência**: cada evento carrega `X-Event-Id` (UUID gerado na inserção do outbox, estável entre retries). O cliente deduplica pelo header. Diego estimou que at-least-once com `X-Event-Id` resolve 99% dos casos (`TRANSCRICAO.md:154`); Marcos se comprometeu a documentar o contrato no portal do desenvolvedor antes do go-live (`TRANSCRICAO.md:156`).
- **Assinatura**: payload assinado com HMAC-SHA256 usando secret única do endpoint de destino, transmitida em header dedicado. Detalhes de headers e estrutura de payload no FDD.

### Resiliência

Detalhado em [ADR-002](./adrs/ADR-002-exponential-backoff-retry-com-dead-letter-queue.md). Resumo:

| Tentativa | Aguarda após falha |
|---|---|
| 1ª | 1 minuto |
| 2ª | 5 minutos |
| 3ª | 30 minutos |
| 4ª | 2 horas |
| 5ª | 12 horas |

Após a 5ª falha: evento movido para `webhook_dead_letter`. Reprocessamento manual via endpoint de replay restrito a role ADMIN, com log de auditoria de quem executou a ação. Contrato de API no FDD.

### Segurança

Detalhado em [ADR-004](./adrs/ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md). Resumo:

- HMAC-SHA256 sobre o corpo bruto da requisição, secret única por endpoint.
- Secret gerada pela plataforma no cadastro, retornada apenas na resposta do `POST` de criação.
- Rotação disponível via endpoint dedicado; secret anterior válida por 24h em paralelo (grace period para migração do cliente sem downtime).
- TLS obrigatório: URL `http` recusada com erro de validação no schema Zod.
- Revisão de segurança de 2 dias úteis com Sofia obrigatória antes de qualquer deploy em produção.

---

## Alternativas Consideradas e Descartadas

### Webhook síncrono dentro da transação de pedido

A abordagem mais simples seria executar a chamada HTTP ao endpoint do cliente diretamente dentro de `OrderService.changeStatus`, antes do commit da transação.

**Por que foi descartada**: a transação de mudança de status é composta — atualiza orders, registra histórico e ajusta estoque. Inserir uma chamada HTTP nesse fluxo expõe toda a operação à latência e indisponibilidade do sistema externo do cliente. Se o cliente estiver fora do ar ou lento, a transação fica bloqueada, degradando o throughput da API para todos os pedidos. Mais crítico: se o commit falhar após o envio HTTP, o evento foi despachado para um status que não existe — inconsistência irrecuperável. Descartado por unanimidade. (`TRANSCRICAO.md:33-37`)

### Broker de mensagens externo (Redis Streams, Kafka, RabbitMQ)

A alternativa "correta" em sistemas de maior escala seria publicar o evento em um broker externo e ter um consumer dedicado fazendo a entrega.

**Por que foi descartada**: o problema de coordenação entre o commit da transação MySQL e a publicação no broker não desaparece — a consistência between-systems ainda exige um padrão Outbox de qualquer forma, ou aceita uma janela de inconsistência. Além disso, introduz nova infraestrutura com SLA e operabilidade próprios para um time pequeno. O MySQL existente resolve o problema com menos complexidade operacional. ([ADR-001](./adrs/ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md), `TRANSCRICAO.md:50-52`)

### Trigger de banco para notificação reativa do worker

Em vez de polling, o worker seria ativado por um trigger MySQL que detecta inserções na `webhook_outbox`.

**Por que foi descartada**: MySQL não possui mecanismo nativo equivalente ao `LISTEN/NOTIFY` do PostgreSQL. O projeto usa triggers MySQL, mas sua execução é limitada a SQL — não notificam processos externos. Qualquer implementação reativa exigiria workarounds: escrita em arquivo de sinalização, chamada HTTP a partir do trigger — antipadrões operacionais com pontos de falha adicionais que Diego descreveu como "fica esquisito" (`TRANSCRICAO.md:64`). Polling a cada 2 segundos atende o requisito de entrega sub-10s com latência máxima de 2 segundos, sem complexidade adicional.

### Secret HMAC global compartilhada entre todos os endpoints

Uma única secret de plataforma assinaria todos os payloads de todos os clientes.

**Por que foi descartada**: um vazamento de secret por parte de qualquer cliente comprometeria todas as integrações simultâneamente. O time já vivenciou um incidente onde um cliente expôs uma secret em log de aplicação. Secret por endpoint limita o raio de explosão a um único cliente. ([ADR-004](./adrs/ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md), `TRANSCRICAO.md:127-128`)

### Garantia exactly-once na entrega

Eliminar a possibilidade de entrega duplicada exigiria um protocolo de acknowledgment bidirecional com cada sistema externo ou two-phase commit entre o outbox e o estado do cliente.

**Por que foi descartada**: a complexidade de implementação é desproporcional ao ganho. Diego foi direto: *"At-least-once com event_id resolve 99% dos casos"* (`TRANSCRICAO.md:154`). Exactly-once exigiria two-phase commit ou acknowledgment protocol bidirecional com cada sistema externo — nenhum dos três clientes pediu essa garantia. O padrão adotado por Stripe e GitHub para o mesmo problema é at-least-once com identificador estável.

---

## Questões em Aberto

### 1. Ferramenta de supervisão do processo worker

A decisão de rodar o worker como processo Node.js separado está fechada ([ADR-003](./adrs/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md)), mas a ferramenta de supervisão em produção não foi definida na reunião. As opções são: Docker restart policy (mais simples, suficiente se já há orquestração de containers), PM2 (comum em projetos Node.js, adiciona monitoramento e clustering), ou systemd (se o deploy for em VM bare-metal). A escolha impacta o `Dockerfile`, os scripts de CI/CD e a estratégia de health check do worker.

**Responsável por resolver**: Diego / Larissa. Necessário antes do início da Etapa de Deploy (Sprint 3).

### 2. Rate limiting de envio de webhooks por cliente

Se um cliente tiver dezenas de pedidos mudando de status em um minuto, o worker disparará uma chamada HTTP por evento por webhook registrado. Não há controle de volume de saída nesta fase. O time decidiu observar o comportamento em produção antes de implementar throttling — mas nenhum limiar de observação foi definido. Sem um critério explícito ("se passar de X chamadas/minuto para o mesmo endpoint, throttle"), a decisão de implementar pode nunca ter um gatilho claro.

**Responsável por resolver**: Diego. Ponto em aberto explicitado na reunião (`TRANSCRICAO.md:224-230`).

### 3. Archival de linhas entregues na outbox

Diego declarou na reunião: *"Linhas entregues a gente arquiva depois de 30 dias ou assim, fora do escopo dessa feature"* (`TRANSCRICAO.md:56`). O período de 30 dias está acordado, mas o mecanismo não foi definido: job agendado, rotina periódica, migração manual. Sem implementação, a tabela `webhook_outbox` cresce indefinidamente em produção.

**Responsável por resolver**: Diego / Bruno. Adiado explicitamente para fase posterior.

### 4. Notificação ao cliente em caso de falhas consecutivas

Marcos perguntou diretamente: *"Tem como avisar o cliente quando o webhook dele tá com problema? Tipo se ele falhou 3 vezes seguidas, mandar email pra ele."* A resposta de Larissa foi explícita: *"Não. Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto."* (`TRANSCRICAO.md:218-221`).

Sem esse mecanismo, um cliente pode ficar dias sem receber notificações enquanto seu endpoint estiver fora do ar, sem saber que está perdendo eventos. O limiar de 5 falhas até o DLQ foi definido (`decisions.md` #4), mas não há sinal ativo para o cliente.

**Responsável por resolver**: Marcos / Larissa. Adiado para fase posterior após medição de impacto em produção.

---

## Impacto e Riscos

### Impacto no código existente

A única modificação em código existente é em `src/modules/orders/order.service.ts`: o método `changeStatus` passa a chamar `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da transação Prisma existente. O restante da implementação é aditivo — novo módulo, novas tabelas, novo processo.

### Limitações conhecidas desta arquitetura

**Ordenação de eventos**: com worker único, eventos são processados na ordem de inserção no outbox, garantindo que um cliente receba PAID → PROCESSING → SHIPPED na sequência correta para um mesmo pedido. Se o sistema for escalado para múltiplos workers em paralelo no futuro, essa garantia se perde — escalonamento com particionamento por `order_id` está explicitamente adiado (`TRANSCRICAO.md:80-84`, `decisions.md` #28).

**Filtro na inserção, não no envio**: se nenhum webhook ativo do customer está inscrito para o status que mudou, nenhuma linha é inserida no outbox. Isso é eficiente mas irreversível — um evento de status não inscrito não pode ser entregue retroativamente. Clientes precisam configurar seus filtros antes das mudanças de status ocorrerem (`TRANSCRICAO.md:194-199`).

### Riscos principais

| Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|
| Worker para silenciosamente; eventos acumulam sem entrega | Médio | Alto | Monitoramento de fila (questão em aberto #4) |
| Cliente recebe duplicatas e não implementa idempotência | Baixo | Médio | Documentação clara no portal do desenvolvedor antes do go-live (Marcos, `TRANSCRICAO.md:156`) |
| Vazamento de secret por cliente expõe sua própria integração | Baixo | Médio | Secret por endpoint limita raio de explosão; rotação disponível |
| Crescimento descontrolado da tabela `webhook_outbox` | Médio (longo prazo) | Médio | Archival de 30 dias (questão em aberto #3) |
| Transação de pedido mais lenta com inserção na outbox | Baixo | Baixo | Inserção atômica de uma linha; impacto esperado mínimo |

### Prazo

3 sprints com revisão de segurança de Sofia incluída no Sprint 3, antes do deploy. Prazo final: fim de novembro de 2026.

---

## Decisões Relacionadas

| ADR | Decisão |
|---|---|
| [ADR-001 — Transactional Outbox para despacho de eventos webhook](./adrs/ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md) | Atomicidade entre mudança de status do pedido e criação do evento de webhook |
| [ADR-002 — Retry com backoff exponencial e Dead Letter Queue](./adrs/ADR-002-exponential-backoff-retry-com-dead-letter-queue.md) | Estratégia de resiliência para entregas com falha: 5 tentativas, DLQ, replay ADMIN |
| [ADR-003 — Processo worker separado com polling para consumo do outbox](./adrs/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md) | Isolamento do ciclo de vida do worker em relação à API; polling a cada 2 segundos |
| [ADR-004 — HMAC-SHA256 por endpoint com rotação de segredo](./adrs/ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md) | Autenticação e integridade do payload; secret única por endpoint com grace period de 24h |
| [ADR-005 — Entrega at-least-once com idempotência via X-Event-Id](./adrs/ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md) | Semântica de entrega e mecanismo de deduplicação delegado ao cliente |
