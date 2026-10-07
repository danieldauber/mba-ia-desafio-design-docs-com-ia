# Decisões Técnicas — Sistema de Webhooks de Notificação de Pedidos

**Fonte:** Transcrição de reunião técnica (09:00 – 09:53)
**Participantes:** Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Sênior), Sofia (Eng. Segurança)

---

## Aplicadas

| # | Decisão | Responsável | Justificativa |
|---|---|---|---|
| 1 | Padrão **Transactional Outbox** em MySQL para disparo de webhooks | Diego / Bruno | Atomicidade com a transação de mudança de status; evita inconsistência entre orders e eventos |
| 2 | Worker como **processo separado** (`src/worker.ts` + `npm run worker`) | Bruno | Isolamento de ciclo de vida; reinício da API não mata o worker |
| 3 | Worker em **polling a cada 2 segundos** | Diego | MySQL sem LISTEN/NOTIFY nativo; atende requisito < 10s de latência |
| 4 | **Retry com backoff exponencial**: 5 tentativas — 1m / 5m / 30m / 2h / 12h | Diego | Cobre janelas de manutenção planejada (~2h); evita evento pendurado indefinidamente |
| 5 | **DLQ em tabela separada** `webhook_dead_letter` com payload, motivo e timestamp | Diego | Legibilidade da outbox principal; evidência para debug e reprocessamento |
| 6 | Endpoint `POST /admin/webhooks/dead-letter/:id/replay` com role **ADMIN** + log de auditoria | Sofia / Larissa | Operação sensível; reaproveita middleware `requireRole` existente |
| 7 | **HMAC-SHA256** sobre o corpo do request, secret **por endpoint** | Sofia | Padrão de mercado; vazamento de uma secret não compromete todas |
| 8 | **Rotação de secret** com grace period de **24 horas** para a secret antiga | Sofia | Histórico de cliente que vazou secret em log; janela para migração sem downtime |
| 9 | **TLS obrigatório** — URL `https` validada no schema Zod, `http` recusado | Sofia | Validação de entrada; requisito não funcional |
| 10 | Garantia **at-least-once** com `X-Event-Id` (UUID) para dedup pelo cliente | Diego | Exactly-once exigiria coordenação bidirecional; padrão Stripe/GitHub |
| 11 | **Filtro de eventos na inserção** da outbox por lista de status configurada por webhook | Bruno / Diego | Evita linhas desnecessárias na tabela |
| 12 | **Payload renderizado como snapshot** na inserção do outbox | Larissa / Diego | Preserva estado exato do momento da mudança; evita inconsistências por updates posteriores |
| 13 | **Estrutura de módulo** `src/modules/webhooks` seguindo padrão existente (controller, service, repository, routes, schemas) | Bruno | Consistência com arquitetura atual do projeto |
| 14 | Reuso de **AppError, Pino, error middleware, schemas Zod** — prefixo `WEBHOOK_` nos códigos de erro | Bruno | Zero nova infra; coerência com padrões estabelecidos |
| 15 | Integração no `OrderService.changeStatus` via função `publishWebhookEvent(tx, order, fromStatus, toStatus)` dentro da mesma transação | Bruno | Garante atomicidade outbox + mudança de status; sem injeção de repository completo |
| 16 | **PrismaClient separado** para o worker (mesmo `DATABASE_URL`, instância nova) | Bruno | PrismaClient é por processo Node |
| 17 | **IDs UUID** na tabela outbox (padrão do projeto) | Larissa | Confirmação pós-call; segue padrão uniforme do repositório |
| 18 | Timeout de **10 segundos** nas chamadas HTTP do worker | Diego / Sofia | Cliente que não responde entra no fluxo de retry |
| 19 | Limite de **64KB** por payload; erro se ultrapassar | Diego | Requisito não funcional; nenhum evento real deve atingir esse teto |
| 20 | Headers: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type: application/json` | Diego / Sofia | Identificação, verificação de integridade e detecção de replay attack |
| 21 | Payload com campos: `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` — **sem items** | Diego | Payload enxuto; cliente busca detalhes via `GET /orders/:id` |
| 22 | Endpoints CRUD de webhooks: `POST`, `PATCH`, `DELETE`, `GET` por customer; `GET /webhooks/:id/deliveries` | Bruno / Marcos | Requisito funcional core dos clientes B2B |
| 23 | **Webhook somente outbound** (plataforma → cliente) | Sofia / Marcos | Escopo definido explicitamente; clientes querem receber, não enviar |
| 24 | Ordenação garantida **somente por order_id** com single-worker (limitação conhecida documentada) | Diego | Escalonamento multi-worker quebraria ordering; tratado como problema futuro |

---

## Adiadas

| # | Decisão | Responsável | Justificativa |
|---|---|---|---|
| 25 | Archival de linhas entregues na outbox após 30 dias | Diego | Explicitamente declarado fora do escopo desta feature |
| 26 | Notificação por **email** em caso de falhas consecutivas no webhook | Marcos / Larissa | Fora de escopo desta fase; reavaliar após medição de impacto |
| 27 | **Rate limiting** de envio de webhooks por cliente | Diego / Larissa | Ponto em aberto; implementar somente se virar problema observado |
| 28 | Escalabilidade para **múltiplos workers** com particionamento por `order_id` | Diego | Complexidade desnecessária agora; documentada como limitação conhecida |

---

## Descartadas

| # | Decisão | Responsável | Justificativa |
|---|---|---|---|
| 29 | **Dashboard visual** para o cliente acompanhar webhooks | Larissa / Marcos | Projeto separado do time de frontend; fora de escopo desta entrega |
| 30 | **Webhook síncrono** dentro do service de orders | Larissa / Bruno / Diego | Transação de mudança de status já é pesada; cliente offline causaria rollback em cascata |
| 31 | **Redis Streams** ou broker externo como alternativa ao outbox | Diego | Overengineering para time pequeno; outbox no MySQL existente resolve |
| 32 | **Trigger de banco** para notificar o worker reativamente | Diego | MySQL não possui NOTIFY/LISTEN nativo; solução workaround seria antipadrão |
| 33 | **Secret global única** compartilhada entre todos os endpoints | Sofia | Vazamento de uma compromete todas as integrações |
| 34 | Garantia **exactly-once** na entrega | Diego | Exige coordenação bidirecional; at-least-once + `X-Event-Id` é o padrão de mercado |
| 35 | Retry com **3 tentativas** | Diego | Insuficiente para cobrir janelas de manutenção de ~2h do cliente |
| 36 | Armazenar na outbox apenas `order_id` e **renderizar payload no envio** | Larissa / Diego | Risco de inconsistência se o pedido mudar após a inserção |
