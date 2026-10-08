# TRACKER: Sistema de Webhooks de Notificação de Pedidos

Rastreabilidade cruzada de cada decisão, requisito e restrição aos documentos onde aparecem e à sua fonte primária (transcrição ou código).

**Formato de fonte:**
- `TRANSCRICAO [hh:mm] Nome` — timestamp e participante da reunião
- `CODIGO caminho/do/arquivo` — arquivo existente no repositório
- `DECISAO #N` — entrada na tabela de `decisions.md`

---

| ID | Item | Tipo | Documentos | Fonte |
|---|---|---|---|---|
| T-001 | Latência máxima de 10 segundos para entrega do evento | Requisito não funcional | RFC §Contrato de entrega, PRD §Objetivos, FDD §2 | TRANSCRICAO [09:02] Marcos |
| T-002 | Webhook exclusivamente outbound (plataforma → cliente) | Restrição de escopo | RFC §Visão geral, PRD §Contexto, FDD §1 | TRANSCRICAO [09:02] Marcos, [09:03] Sofia |
| T-003 | Padrão Transactional Outbox em MySQL | Decisão arquitetural | ADR-001, RFC §Proposta, PRD §Decisões, FDD §4 | TRANSCRICAO [09:06] Diego |
| T-004 | Descarte de webhook síncrono dentro do service de orders | Alternativa descartada | ADR-001 §Alternativas, RFC §Alternativas Descartadas | TRANSCRICAO [09:04] Bruno, [09:06] Diego |
| T-005 | Descarte de Redis Streams / broker externo | Alternativa descartada | ADR-001 §Alternativas, RFC §Alternativas Descartadas | TRANSCRICAO [09:07] Diego |
| T-006 | Tabela `webhook_outbox` como fila transacional | Componente de dados | ADR-001, FDD §4 Schema, PRD §Arquitetura | TRANSCRICAO [09:06] Diego |
| T-007 | Índices em `status` e `created_at` na outbox | Decisão de performance | ADR-001 §Consequências, FDD §10 Riscos | TRANSCRICAO [09:08] Diego |
| T-008 | Archival de linhas entregues após 30 dias (adiado) | Decisão adiada | RFC §Questões em Aberto, PRD §Fora de escopo | TRANSCRICAO [09:08] Diego |
| T-009 | Worker em polling a cada 2 segundos | Decisão arquitetural | ADR-003, RFC §Visão geral, PRD §Decisões, FDD §4 | TRANSCRICAO [09:09] Diego, [09:10] Larissa |
| T-010 | Descarte de trigger MySQL como mecanismo reativo | Alternativa descartada | ADR-003 §Alternativas, RFC §Alternativas Descartadas | TRANSCRICAO [09:09] Diego |
| T-011 | Worker como processo Node.js separado (`src/worker.ts`) | Decisão arquitetural | ADR-003, RFC §Visão geral, PRD §Arquitetura, FDD §4 | TRANSCRICAO [09:11] Diego, [09:11] Larissa |
| T-012 | Entry-point do worker baseada em `src/server.ts` existente | Padrão de implementação | ADR-003 §Contexto, FDD §8 Integração | TRANSCRICAO [09:11] Larissa |
| T-013 | Ordenação por `created_at ASC` com garantia apenas para single-worker | Limitação conhecida | ADR-003 §Consequências, RFC §Limitações, FDD §4 Fluxo worker | TRANSCRICAO [09:12] Diego, [09:13] Larissa |
| T-014 | Multi-worker com particionamento por `order_id` adiado | Decisão adiada | RFC §Limitações, PRD §Fora de escopo | TRANSCRICAO [09:13] Diego |
| T-015 | Retry com backoff exponencial, 5 tentativas | Decisão arquitetural | ADR-002, RFC §Resiliência, PRD §Decisões, FDD §6 | TRANSCRICAO [09:15] Diego, [09:16] Larissa |
| T-016 | Descarte de retry com 3 tentativas | Alternativa descartada | ADR-002 §Alternativas | TRANSCRICAO [09:16] Bruno, [09:16] Diego |
| T-017 | Progressão do backoff: 1m / 5m / 30m / 2h / 12h (~14h36m total) | Parâmetro de configuração | ADR-002 §Consequências, RFC §Resiliência, FDD §6 | TRANSCRICAO [09:17] Diego |
| T-018 | DLQ em tabela `webhook_dead_letter` separada | Decisão arquitetural | ADR-002, RFC §Resiliência, PRD §Escopo, FDD §4 Schema | TRANSCRICAO [09:18] Diego |
| T-019 | Endpoint `POST /admin/webhooks/dead-letter/:id/replay` | Requisito funcional | ADR-002 §Consequências, RFC, PRD RF-010, FDD Contrato 4 | TRANSCRICAO [09:18] Diego, [09:35] Diego |
| T-020 | Replay reinseriu na outbox com `attempt_count = 0`; DLQ preservado | Regra de negócio | ADR-002 §Consequências, FDD RF-010 §Fluxo principal | TRANSCRICAO [09:18] Diego |
| T-021 | HMAC-SHA256 sobre o corpo do request | Decisão de segurança | ADR-004, RFC §Segurança, PRD §Segurança, FDD Contrato 5 | TRANSCRICAO [09:20] Sofia |
| T-022 | Secret única por endpoint (não global) | Decisão de segurança | ADR-004, RFC §Segurança, PRD §Decisões, FDD §6 | TRANSCRICAO [09:21] Sofia |
| T-023 | Descarte de secret global compartilhada | Alternativa descartada | ADR-004 §Alternativas, RFC §Alternativas Descartadas | TRANSCRICAO [09:21] Sofia, [09:22] Diego |
| T-024 | Rotação de secret com grace period de 24 horas | Decisão de segurança | ADR-004 §Consequências, RFC §Segurança, PRD RF-008, FDD Contrato 6 | TRANSCRICAO [09:21] Sofia |
| T-025 | TLS obrigatório — URL `https` validada via Zod | Requisito de segurança | ADR-004, PRD §Segurança, FDD §6 Erros | TRANSCRICAO [09:23] Sofia |
| T-026 | Limite de 64KB por payload | Requisito não funcional | PRD §Segurança, FDD §6 Erros, FDD §9 Critérios | TRANSCRICAO [09:24] Diego |
| T-027 | Semântica at-least-once com `X-Event-Id` UUID para dedup | Decisão arquitetural | ADR-005, RFC §Contrato, PRD §Decisões, FDD Contrato 5 | TRANSCRICAO [09:25] Diego |
| T-028 | Descarte de garantia exactly-once | Alternativa descartada | ADR-005 §Alternativas, RFC §Alternativas Descartadas | TRANSCRICAO [09:25] Diego |
| T-029 | `X-Event-Id` gerado na inserção do outbox, estável entre retries | Invariante crítico | ADR-005 §Consequências, FDD §6 Invariantes | TRANSCRICAO [09:25] Diego |
| T-030 | Documentação de at-least-once no portal do desenvolvedor (Marcos) | Dependência organizacional | RFC §Contrato, PRD §Dependências, FDD §10 Riscos | TRANSCRICAO [09:26] Marcos |
| T-031 | Módulo `src/modules/webhooks` seguindo padrão existente | Padrão de implementação | ADR-001 §Contexto, PRD §Arquitetura, FDD §8 Integração | TRANSCRICAO [09:27] Bruno |
| T-032 | Separação entre entry-point `src/worker.ts` e lógica `webhook.processor.ts` | Padrão de implementação | FDD §8 Componentes | TRANSCRICAO [09:28] Bruno |
| T-033 | Prefixo `WEBHOOK_` em todos os códigos de erro do módulo | Padrão de implementação | PRD §Segurança, FDD §6 Matriz de erros | TRANSCRICAO [09:28] Bruno, [09:29] Larissa |
| T-034 | Reuso de AppError, Pino, error middleware, schemas Zod | Decisão de implementação | PRD §Arquitetura, FDD §8 Dependências | TRANSCRICAO [09:29] Bruno, [09:30] Larissa |
| T-035 | PrismaClient separado para o worker | Decisão de implementação | ADR-003 §Consequências, FDD §8 Dependências | TRANSCRICAO [09:30] Bruno |
| T-036 | Secret retornada somente na resposta do `POST` de criação; nunca no `GET` | Regra de segurança | ADR-004 §Decisão, PRD §Segurança, FDD Contrato 1 | TRANSCRICAO [09:31] Marcos, implícito na revisão de Sofia |
| T-037 | `customerId` passado no body (não extraído do JWT) | Regra de implementação | FDD Contrato 1, PRD RF-001 | TRANSCRICAO [09:32] Larissa |
| T-038 | Cadastro de webhook (`POST /api/v1/webhooks`) | Requisito funcional | PRD RF-001, FDD Contrato 1 | TRANSCRICAO [09:33] Bruno, [09:36] Sofia |
| T-038b | Edição de webhook (`PATCH /api/v1/webhooks/:id`) | Requisito funcional | PRD RF-002, FDD Contrato 2 | TRANSCRICAO [09:33] Bruno |
| T-038c | Listagem de webhooks (`GET /api/v1/webhooks?customerId=:id`) | Requisito funcional | PRD RF-003, FDD Contrato 1 | TRANSCRICAO [09:33] Bruno |
| T-038d | Remoção de webhook (`DELETE /api/v1/webhooks/:id`) | Requisito funcional | PRD RF-004, FDD Contrato 2 | TRANSCRICAO [09:33] Bruno |
| T-039 | Filtro de eventos na inserção do outbox (não no despacho) | Decisão de implementação | ADR-001 §Consequências, RFC §Limitações, PRD RF-005, FDD §4 Fluxo principal | TRANSCRICAO [09:34] Bruno, [09:34] Diego |
| T-040 | `GET /api/v1/webhooks/:id/deliveries` para histórico de entregas | Requisito funcional | PRD RF-009, FDD Contrato 3 | TRANSCRICAO [09:34] Marcos |
| T-041 | Role ADMIN obrigatória no endpoint de replay de DLQ | Requisito de segurança | ADR-002, PRD RF-010, FDD Contrato 4 | TRANSCRICAO [09:36] Sofia |
| T-042 | Log de auditoria de replay com `admin_id`, `dead_letter_id`, timestamp | Requisito de auditoria | PRD RF-010, FDD §7 Logs, FDD §9 Critérios | TRANSCRICAO [09:36] Sofia |
| T-043 | Reuso de `requireRole` existente para o endpoint de replay | Padrão de implementação | PRD §Arquitetura, FDD §8 Integração | TRANSCRICAO [09:36] Larissa |
| T-044 | Notificação por email em falhas consecutivas (adiado) | Decisão adiada | RFC §Questões em Aberto §4, PRD §Fora de escopo | TRANSCRICAO [09:37] Larissa |
| T-045 | Rate limiting de envio por cliente (observar, adiado) | Decisão adiada | RFC §Questões em Aberto §2, PRD §Fora de escopo | TRANSCRICAO [09:39] Diego, [09:39] Larissa |
| T-046 | Dashboard visual fora de escopo | Decisão descartada | PRD §Fora de escopo | TRANSCRICAO [09:40] Larissa |
| T-047 | Integração em `OrderService.changeStatus` via `publishWebhookEvent(tx, order, fromStatus, toStatus)` | Ponto de integração crítico | ADR-001 §Consequências, RFC §Fluxo, PRD §Dependências, FDD §4 e §8 | TRANSCRICAO [09:41] Bruno |
| T-048 | Função `publishWebhookEvent` recebe `tx` como primeiro argumento (não injeta repository) | Decisão de design | ADR-001, FDD §8 Integração | TRANSCRICAO [09:41] Bruno, [09:41] Diego |
| T-049 | Timeout de 10 segundos nas chamadas HTTP do worker | Requisito não funcional | PRD §Performance, FDD §5 Contrato 5, FDD §6 Erros | TRANSCRICAO [09:42] Diego |
| T-050 | Payload: `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | Contrato de payload | FDD §5 Contrato 5, PRD §Escopo | TRANSCRICAO [09:43] Diego |
| T-051 | `items` não incluídos no payload; cliente consulta `GET /orders/:id` para detalhes | Decisão de payload | FDD §5 Contrato 5 | TRANSCRICAO [09:43] Diego |
| T-052 | Headers obrigatórios: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `Content-Type: application/json` | Contrato de headers | FDD §5 Contrato 5 | TRANSCRICAO [09:44] Diego |
| T-053 | Header `X-Webhook-Id` adicionado por sugestão de Sofia | Contrato de headers | FDD §5 Contrato 5 | TRANSCRICAO [09:44] Sofia |
| T-054 | Prazo de 3 sprints até fim de novembro de 2026 | Restrição organizacional | RFC §Prazo, PRD §Resumo | TRANSCRICAO [09:46] Larissa, [09:47] Marcos |
| T-055 | Revisão de segurança de 2 dias úteis com Sofia antes do deploy | Dependência organizacional | RFC §Segurança, PRD §Dependências, PRD §Critérios | TRANSCRICAO [09:46] Sofia |
| T-056 | IDs UUID (não auto-incremental) na tabela outbox | Decisão de dados | ADR-001, FDD §4 Schema | TRANSCRICAO [09:51] Larissa |
| T-057 | Payload armazenado como snapshot na inserção do outbox | Decisão arquitetural | ADR-001 §Decisão, RFC §Visão geral, FDD §4 Fluxo principal | TRANSCRICAO [09:52] Larissa, [09:52] Diego |
| T-058 | Descarte de renderizar payload no momento do envio | Alternativa descartada | ADR-001 §Alternativas | TRANSCRICAO [09:52] Larissa |
| T-059 | `AppError` base para todos os erros do módulo webhooks | Padrão de implementação | FDD §6 Erros, PRD §Segurança | CODIGO src/shared/errors/app-error.ts |
| T-060 | `requireRole('ADMIN')` reusado sem modificação no endpoint de replay | Padrão de implementação | FDD §8 Integração, PRD RF-010 | CODIGO src/middlewares/auth.middleware.ts |
| T-061 | `errorMiddleware` trata `AppError` com prefixo `WEBHOOK_` automaticamente | Padrão de implementação | FDD §8 Integração, FDD §6 Erros | CODIGO src/middlewares/error.middleware.ts |
| T-062 | `createPrismaClient()` reutilizada pelo worker para instanciar PrismaClient separado | Padrão de implementação | ADR-003 §Referências, FDD §8 Integração | CODIGO src/config/database.ts |
| T-063 | `$transaction` Prisma em `changeStatus` como ponto de inserção do outbox | Ponto de integração | ADR-001 §Referências, FDD §8 Integração | CODIGO src/modules/orders/order.service.ts |
| T-064 | `validate()` middleware reusado para validação de body/query/params no módulo webhooks | Padrão de implementação | FDD §8 Integração | CODIGO src/middlewares/validate.middleware.ts |
| T-065 | Pino logger com redação de campos sensíveis reusado sem modificação | Padrão de implementação | FDD §7 Logs, PRD §Observabilidade | CODIGO src/shared/logger/index.ts |
| T-066 | Padrão UUID `@id @default(uuid()) @db.Char(36)` para PKs das novas tabelas | Padrão de dados | FDD §4 Schema | CODIGO prisma/schema.prisma |
| T-067 | Padrão de módulo controller/service/repository/routes/schemas seguido pelo módulo webhooks | Padrão de implementação | PRD §Arquitetura, FDD §8 Integração | CODIGO src/modules/orders/ (referência de padrão) |
| T-068 | Testes de integração contra MySQL real (sem mocks), com Vitest + Supertest | Estratégia de testes | PRD §Testes | CODIGO tests/setup.ts |

---

## Sumário de cobertura

| Categoria | Contagem |
|---|---|
| Total de itens rastreados | 71 |
| Itens com Fonte = TRANSCRICAO | 61 |
| Itens com Fonte = CODIGO | 10 |
| Decisões aplicadas rastreadas (de 24 em decisions.md) | 24 |
| Decisões adiadas rastreadas (de 4 em decisions.md) | 4 |
| Alternativas descartadas rastreadas (de 8 em decisions.md) | 6 |
| Documentos referenciados | ADR-001, ADR-002, ADR-003, ADR-004, ADR-005, RFC, PRD, FDD |

---

## Itens sem rastreabilidade direta à transcrição

Os itens abaixo têm origem em análise do codebase ou consolidação pós-reunião — não têm timestamp na transcrição, mas têm evidência no código:

| ID | Item | Justificativa |
|---|---|---|
| T-059 | `AppError` base para erros do módulo | Padrão inferido do código existente; alinhado com [09:28-09:29] Bruno sobre prefixo WEBHOOK_ |
| T-060 | `requireRole` sem modificação | Padrão confirmado em [09:36] Larissa; código verificado em `auth.middleware.ts` |
| T-061 | `errorMiddleware` automático | Padrão confirmado em [09:29] Bruno; código verificado em `error.middleware.ts` |
| T-062 | `createPrismaClient()` reusada pelo worker | Padrão confirmado em [09:30] Bruno; factory verificada em `database.ts` |
| T-063 | `$transaction` em `changeStatus` | Padrão confirmado em [09:41] Bruno; código verificado em `order.service.ts` |
| T-064 | `validate()` middleware reusado | Padrão de todos os módulos existentes; confirmado em [09:30] Larissa |
| T-065 | Pino com redação de sensíveis | Padrão confirmado em [09:29] Bruno; logger verificado em `logger/index.ts` |
| T-066 | UUID Char(36) para PKs | Decisão de [09:51] Larissa; padrão verificado em `prisma/schema.prisma` |
| T-067 | Padrão de módulo | Padrão de todos os módulos existentes; confirmado em [09:27] Bruno |
| T-068 | Testes contra MySQL real | Padrão do projeto sem mocks; verificado em `tests/setup.ts` |
