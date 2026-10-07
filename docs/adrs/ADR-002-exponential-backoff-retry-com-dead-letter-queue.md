# ADR-002: Estratégia de Retry com Backoff Exponencial e Fila de Mensagens Mortas para Webhooks

**Status:** Aceito
**Data:** 24-06-2026
**Depends on:** [ADR-001: Padrão Transactional Outbox para Despacho de Eventos Webhook](./ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md)
**Related to:**
- [ADR-003: Processo Worker Separado com Polling para Consumo do Outbox](./needs-input/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md)
- [ADR-005: Entrega Pelo-Menos-Uma-Vez com Idempotência via X-Event-Id](./ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md)

---

## 1. Contexto e Declaração do Problema

O sistema de webhooks precisa de uma estratégia de resiliência para entregas com falha. Clientes que integram com a API dependem do comportamento de retry como parte do contrato implícito de entrega — a janela de retry representa um compromisso de nível de negócio, não apenas um detalhe de implementação.

A base de clientes existente apresenta padrões de indisponibilidade de até 2 horas (janelas de manutenção planejadas). A estratégia de retry precisa cobrir esse padrão sem manter eventos pendentes indefinidamente para clientes permanentemente offline.

Eventos que esgotam todas as tentativas precisam de um destino explícito com trilha de auditoria, e deve existir um mecanismo de reprocessamento manual para recuperação operacional.

## 2. Fatores de Decisão

- Janelas de manutenção de clientes atingem historicamente cerca de 2 horas; a estratégia de retry deve cobrir esse intervalo com margem.
- Eventos pendentes indefinidamente degradam a legibilidade da fila principal e mascaram falhas permanentes.
- A trilha de auditoria de falhas (payload, motivo, timestamp) é necessária para suporte e diagnóstico.
- O reprocessamento manual de eventos mortos deve ser restrito a usuários ADMIN para controle de acesso.
- A estratégia de retry define o ponto de gatilho para funcionalidades futuras dependentes de falha consecutiva (notificação por email, decisões.md #26).
- O estado de retry (`contagem de tentativas`, `próxima tentativa em`, `último erro`) deve ser rastreável por evento no outbox.

## 3. Opções Consideradas

- **Opção A**: Backoff exponencial com 5 tentativas e tabela DLQ separada
- **Opção B**: 3 tentativas de retry com backoff exponencial
- **Opção C**: Retry indefinido com backoff exponencial

## 4. Decisão

Opção escolhida: **Opção A — Backoff exponencial com 5 tentativas e tabela DLQ separada**, porque cobre a janela de manutenção de 2 horas com margem (janela total de ~15 horas), fornece um destino explícito e auditável para eventos não entregues, e mantém a fila principal (`webhook_outbox`) legível sem contaminação de eventos mortos.

O comportamento do endpoint de replay (`POST /admin/webhooks/dead-letter/:id/replay`) é reinserção na `webhook_outbox` como evento pendente — sem reset do histórico de tentativas anteriores, que permanece na tabela `webhook_dead_letter` como trilha de auditoria.

## 5. Prós e Contras das Opções

### Opção A: Backoff exponencial com 5 tentativas e tabela DLQ separada

**Prós:**
- Cobre janelas de manutenção de ~2 horas com margem (última tentativa em ~15 horas após a primeira falha).
- Tabela DLQ separada preserva legibilidade do outbox principal e fornece trilha de auditoria limpa com payload, motivo de falha e timestamp.
- Endpoint de replay restrito a ADMIN reutiliza infraestrutura de autorização existente.

**Contras:**
- Clientes com breve indisponibilidade podem receber notificações com até 15 horas de atraso.
- A separação em duas tabelas aumenta a complexidade operacional do schema de dados.

### Opção B: 3 tentativas de retry com backoff exponencial

**Prós:**
- Menor janela de atraso máximo para entrega de eventos.
- Schema de estado mais simples com menos transições.

**Contras:**
- Insuficiente para cobrir janelas de manutenção de 2 horas — explicitamente rejeitado com base em dados históricos de indisponibilidade de clientes.
- Aumenta a taxa de eventos que alcançam o DLQ prematuramente.

### Opção C: Retry indefinido com backoff exponencial

**Prós:**
- Garante entrega eventual para clientes que ficam offline temporariamente por períodos longos.

**Contras:**
- Eventos de clientes permanentemente offline ficam pendentes indefinidamente, degradando a fila principal.
- Sem ponto de gatilho definido para alertas de falha ou ações de recuperação.
- Explicitamente rejeitado por tornar o estado do sistema opaco.

## 6. Consequências

A adoção desta estratégia estabelece uma janela de entrega de até ~15 horas após a primeira falha, com 5 tentativas em intervalos progressivos definidos na reunião de 24-06-2026:

| Tentativa | Intervalo após falha anterior |
|-----------|-------------------------------|
| 1ª        | 1 minuto                      |
| 2ª        | 5 minutos                     |
| 3ª        | 30 minutos                    |
| 4ª        | 2 horas                       |
| 5ª        | 12 horas                      |

Tempo total entre primeira falha e última tentativa: aproximadamente 14h36m. Esse comportamento passa a ser parte do contrato implícito de SLA para integradores da API de webhooks.

O worker de processamento do outbox deve rastrear o estado de retry por evento e calcular o próximo horário de tentativa. A tabela `webhook_dead_letter` torna-se a fonte de verdade para eventos não entregues, com campos de auditoria obrigatórios. O endpoint de replay restrito a ADMIN representa uma extensão do modelo de autorização para a superfície de gerenciamento de webhooks.

Não há limite definido de replays manuais por evento nesta fase. Essa restrição operacional deve ser avaliada após medição de uso em produção.

A funcionalidade de notificação por email em caso de falhas consecutivas (decisions.md #26, atualmente adiada) depende diretamente desta decisão: o limiar de 5 falhas define o gatilho dessa notificação. Qualquer alteração futura no número de tentativas deve considerar esse acoplamento.

## 7. Referências

- `decisions.md:4` — Decisão sobre o cronograma de retry (5 tentativas)
- `decisions.md:5` — Decisão sobre a tabela DLQ separada
- `decisions.md:6` — Decisão sobre o endpoint de replay com papel ADMIN
- `decisions.md:35` — Rejeição formal da variante de 3 tentativas
- `src/middlewares/auth.middleware.ts` — Middleware `requireRole` reutilizado pelo endpoint de replay
