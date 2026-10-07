# ADR-001: Padrão Transactional Outbox para Despacho de Eventos Webhook

**Status:** Aceito
**Data:** 24-06-2026
**Used by:**
- [ADR-002: Estratégia de Retry com Backoff Exponencial e Fila de Mensagens Mortas para Webhooks](./ADR-002-exponential-backoff-retry-com-dead-letter-queue.md)
- [ADR-003: Processo Worker Separado com Polling para Consumo do Outbox](./needs-input/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md)
- [ADR-005: Entrega Pelo-Menos-Uma-Vez com Idempotência via X-Event-Id](./ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md)

---

## Contexto e Problema

O sistema precisa notificar clientes externos via webhook sempre que o status de um pedido muda. A questão central é: como garantir que o evento de notificação seja criado se e somente se a mudança de status for persistida com sucesso no banco de dados?

O módulo de pedidos executa uma sequência de operações em uma única transação de banco de dados: atualização de status, registro de histórico e débito ou reposição de estoque. Qualquer mecanismo de despacho de webhook deve participar dessa mesma fronteira transacional para evitar dois cenários problemáticos: o envio de um evento cujo status foi revertido por rollback, ou a persistência de um status sem a criação do evento correspondente.

A decisão foi tomada na reunião técnica de 24 de junho de 2026 (registrada em `TRANSCRICAO.md`), antes da implementação do módulo. A equipe avaliou explicitamente quatro abordagens antes de selecionar o padrão Transactional Outbox.

## Condutores da Decisão

- Garantia de atomicidade: o evento de webhook deve existir se e somente se a transação de negócio for confirmada.
- Isolamento de falhas: falhas na entrega ao endpoint do cliente não devem afetar a transação de pedido.
- Adequação ao tamanho da equipe: a solução deve evitar a introdução de nova infraestrutura desnecessária para o porte atual do projeto.
- Consistência operacional: o outbox deve ser a única fonte de verdade para eventos pendentes, simplificando rastreabilidade e reprocessamento.
- Acoplamento controlado: o módulo de pedidos não deve depender diretamente do repositório de webhooks.

## Opções Consideradas

1. **Transactional Outbox com tabela `webhook_outbox` no MySQL**
2. **Chamada HTTP síncrona dentro da transação de pedido**
3. **Broker de mensagens externo (Redis Streams, Kafka ou RabbitMQ)**

## Resultado da Decisão

Opção escolhida: **Transactional Outbox com tabela `webhook_outbox` no MySQL**, porque vincula o ciclo de vida do evento ao resultado da transação de banco de dados sem introduzir dependências de infraestrutura adicionais.

Uma função dedicada recebe o cliente de transação Prisma ativo como parâmetro e insere a linha na `webhook_outbox` dentro da mesma transação do `changeStatus`. Um processo worker separado realiza o polling da tabela e executa as chamadas HTTP de saída de forma assíncrona.

**Limitação conhecida e adiada:** a política de retenção e arquivamento das linhas na `webhook_outbox` após entrega bem-sucedida está explicitamente fora do escopo desta fase (ver `decisions.md` #25). A equipe acordou na reunião de 24-06-2026 que linhas entregues serão arquivadas após 30 dias, mas o mecanismo de arquivamento não será implementado neste ciclo.

## Prós e Contras das Opções

### Transactional Outbox com tabela `webhook_outbox` no MySQL

- Positivo: Atomicidade garantida — rollback da transação de pedido impede criação de evento órfão.
- Positivo: Falhas na entrega HTTP são isoladas do fluxo de negócio principal.
- Positivo: Sem nova dependência de infraestrutura; aproveita o MySQL já existente.
- Negativo: Acopla a garantia de entrega de eventos ao MySQL; migração de banco exige revisitar esta decisão.

### Chamada HTTP síncrona dentro da transação de pedido

- Positivo: Implementação simples, sem tabela auxiliar ou worker.
- Negativo: Cliente offline ou lento causa rollback da transação de pedido, bloqueando a operação de negócio.
- Negativo: Latência de rede impacta diretamente o tempo de resposta da API de pedidos.

### Broker de mensagens externo (Redis Streams, Kafka ou RabbitMQ)

- Positivo: Desacoplamento total entre produção e consumo de eventos.
- Negativo: Introduz nova infraestrutura (novo componente, novo SLA, novo ponto de falha).
- Negativo: Coordenação two-phase commit entre MySQL e broker exige padrão outbox de qualquer forma, ou aceita janela de inconsistência.
- Negativo: Considerado sobreengenharia para o porte atual da equipe.

## Consequências

O padrão impõe que qualquer alteração de status de pedido passe pela função de inserção no outbox dentro da mesma transação. Engenheiros que modificarem transições de estado no módulo ORDERS precisam compreender esse contrato; remover ou mover a inserção para fora da transação quebra a atomicidade silenciosamente.

O worker de polling passa a ser um componente de infraestrutura necessário para a operação do sistema. Decisões de retry, fila de mensagens mortas (DLQ) e semântica de entrega (at-least-once) são consequências diretas desta escolha e devem ser documentadas em ADRs complementares.

**Limitação conhecida:** a garantia de ordenação dos eventos é implícita por `order_id` enquanto o sistema operar com worker único — o worker processa eventos na ordem de `created_at` do outbox. Com múltiplos workers em paralelo, a ordenação global não é garantida. Escalonamento multi-worker com particionamento por `order_id` está explicitamente adiado (`decisions.md` #28).

## Referências

- `src/modules/orders/order.service.ts:126`
- `decisions.md` (Decisões #1, #15, #30, #31, #32, #36)
- `TRANSCRICAO.md:44`
