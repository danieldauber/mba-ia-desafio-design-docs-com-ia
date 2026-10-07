---
document_type: FDD
feature: Sistema de Webhooks de Notificação de Pedidos
version: "1.1"
date: "2026-06-24"
author: "Larissa (Tech Lead)"
reviewers:
  - "Diego (Eng. Sênior)"
  - "Bruno (Eng. Pleno)"
  - "Sofia (Eng. Segurança)"
status: Rascunho
rfc: docs/RFC.md
adrs:
  - docs/adrs/ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md
  - docs/adrs/ADR-002-exponential-backoff-retry-com-dead-letter-queue.md
  - docs/adrs/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md
  - docs/adrs/ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md
  - docs/adrs/ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md
---

# FDD: Sistema de Webhooks de Notificação de Pedidos

---

### 1. Contexto e motivação técnica

O Order Management System expõe atualmente apenas uma interface de consulta pull: clientes B2B precisam fazer polling periódico no `GET /api/v1/orders` para detectar mudanças de status de pedidos. Isso impõe latência variável, carga desnecessária na API e acoplamento operacional do lado do cliente.

Três clientes (Atlas Comercial, MaxDistribuição e Nova Cargo) solicitaram formalmente notificações em tempo real. O requisito funcional é que qualquer mudança de status chegue ao endpoint do cliente em menos de 10 segundos.

O sistema não possui nenhum mecanismo de notificação externa. A feature preenche esse vazio com um sistema de webhooks outbound baseado em três componentes: módulo de configuração de webhooks (`src/modules/webhooks/`), inserção atômica no outbox dentro da transação de pedido (`src/modules/orders/order.service.ts`), e worker de despacho como processo Node.js separado (`src/worker.ts`).

A arquitetura adota o padrão Transactional Outbox para garantir que nenhum evento seja criado sem o commit da transação de negócio correspondente, e nunca o contrário.

**Atores:**
- Operador/Admin: cadastra e gerencia configurações de webhook via API
- Worker: processo interno que consome o outbox e despacha chamadas HTTP
- Cliente B2B: sistema externo que recebe e processa os eventos

**Premissas:**
- Webhooks são exclusivamente outbound (plataforma para cliente)
- MySQL 8.0 é o banco de dados; não há broker de mensagens externo
- O cliente é responsável pela deduplicação via `X-Event-Id`
- TLS obrigatório em todos os endpoints de destino

---

### 2. Objetivos técnicos

- Entregar eventos de mudança de status ao cliente em menos de 10 segundos a partir do commit da transação, com latência máxima de polling de 2 segundos (`TRANSCRICAO.md:60-68`)
- Garantir atomicidade entre mudança de status do pedido e criação do evento: se a transação fizer rollback, nenhum evento é criado; se a transação commitar, o evento existe (`TRANSCRICAO.md:44-48`)
- Entregar com semântica at-least-once e `X-Event-Id` UUID estável entre retries para que o cliente possa deduplicar (`TRANSCRICAO.md:149-156`)
- Garantir autenticidade e integridade do payload via HMAC-SHA256, com secret única por endpoint de webhook (`TRANSCRICAO.md:119-124`)
- Cobrir indisponibilidades de cliente de até 15 horas via backoff exponencial com 5 tentativas (1m / 5m / 30m / 2h / 12h) antes de mover para DLQ (`TRANSCRICAO.md:95-108`)
- Não introduzir nova dependência de infraestrutura: outbox persiste no MySQL existente (`TRANSCRICAO.md:50-52`)

---

### 3. Escopo e exclusões

**Incluído**
- CRUD de configuração de webhooks: `POST`, `GET`, `PATCH`, `DELETE /api/v1/webhooks`
- Filtro de eventos por lista de status configurada por webhook
- Geração e rotação de secret HMAC com grace period de 24h
- Tabela `webhook_outbox` e inserção atômica em `OrderService.changeStatus`
- Worker de polling com retry exponencial e DLQ (`webhook_dead_letter`)
- Endpoint `GET /api/v1/webhooks/:id/deliveries` para histórico de entregas
- Endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role ADMIN)
- Validação TLS obrigatório (URL `https` no schema Zod)
- Limite de 64KB por payload; erro se ultrapassar
- Timeout de 10 segundos nas chamadas HTTP do worker

**Excluído**
- Webhooks inbound (recebimento de eventos de sistemas externos)
- Dashboard visual para o cliente acompanhar histórico de webhooks
- Notificação por email em caso de falhas consecutivas (adiado, `decisions.md` #26)
- Rate limiting de envio por cliente (adiado, `decisions.md` #27)
- Escalabilidade multi-worker com particionamento por `order_id` (adiado, `decisions.md` #28)
- Archival automático de linhas entregues na outbox (adiado, `decisions.md` #25)

---

### 4. Fluxos detalhados e diagramas

**Fluxo principal: mudança de status com despacho de webhook**

1. Cliente da API chama `PATCH /api/v1/orders/:id/status` com `{ toStatus: "SHIPPED" }`
2. `OrderController` valida o body via middleware `validate()` (`src/middlewares/validate.middleware.ts`)
3. `OrderService.changeStatus` abre transação Prisma (`$transaction`)
4. Dentro da transação:
   a. Busca o pedido e valida a transição via `canTransition(from, to)` (`src/modules/orders/order.status.ts`)
   b. Atualiza `orders.status`
   c. Insere em `order_status_history`
   d. Debita ou restitui `stock_quantity` conforme `shouldDebitStock` / `shouldReplenishStock`
   e. Chama `publishWebhookEvent(tx, order, fromStatus, toStatus)` — importada de `src/modules/webhooks/webhook.publisher.ts`:
      - Dentro do mesmo `tx`: busca webhooks **ativos** do customer que têm `toStatus` na lista de eventos (`webhooks.events`)
      - **Filtra na inserção, não no despacho** (`TRANSCRICAO.md:194-200`): se nenhum webhook do customer tem o `toStatus` inscrito, nenhuma linha é inserida
      - Para cada webhook ativo encontrado: insere uma linha em `webhook_outbox` com o payload já serializado como snapshot, `webhook_id` da configuração, `status = PENDING` e `attempt_count = 0`
5. Transação commita (ou faz rollback; nesse caso nenhuma linha do outbox persiste)
6. API responde `200 OK` com o pedido atualizado

**Fluxo do worker: consumo e despacho**

1. Worker acorda a cada 2 segundos
2. Busca batch de eventos `PENDING` em `webhook_outbox` ordenados por `created_at ASC` — garante que eventos do mesmo pedido sejam processados na sequência de inserção (ex: PAID → PROCESSING → SHIPPED). Propriedade válida somente com single worker; se escalado para múltiplos workers, a garantia de ordering por pedido se perde (`TRANSCRICAO.md:80-84`, `decisions.md` #28 adiado)
3. Marca cada evento como `PROCESSING` (evita processamento duplo em caso de crash)
4. Para cada evento (a linha da outbox já contém `webhook_id`, `payload` serializado como snapshot e `attempt_count`):
   a. Busca a configuração do webhook pelo `webhook_id` da linha (URL, `secret` atual, `previous_secret` se `previous_secret_expires_at > now`)
   b. Verifica tamanho do payload já armazenado: se > 64KB, move direto para `webhook_dead_letter` com motivo `PAYLOAD_TOO_LARGE`, sem tentativa de envio
   c. Gera assinatura: `HMAC-SHA256(rawBodyString, secret)`
   d. Executa `POST` para a URL do webhook com timeout de 10s e headers obrigatórios
   e. Resposta `2xx`: marca evento como `DELIVERED`, registra em `webhook_deliveries`
   f. Resposta não-`2xx` ou timeout: incrementa `attempt_count`, calcula próximo `retry_at` pelo backoff, marca como `PENDING` novamente
   g. Se `attempt_count >= 5`: move para `webhook_dead_letter` com payload, motivo e timestamp

**Fluxo alternativo: rotação de secret**

1. Operador chama `POST /api/v1/webhooks/:id/rotate-secret`
2. Sistema gera nova secret via `crypto.randomBytes(32).toString('hex')` com prefixo `whsec_`, armazena em `webhooks.secret`, move a anterior para `webhooks.previous_secret`, registra `previous_secret_expires_at = now + 24h`
3. Worker passa a assinar com `secret` atual; ao buscar a configuração do webhook, inclui `previous_secret` na resposta se `previous_secret_expires_at > now`
4. Cliente pode verificar com qualquer uma das duas durante o grace period
5. Após 24h, `previous_secret` é invalidado — verificação inline no worker no passo 4a do fluxo de despacho

**Fluxo alternativo: replay de DLQ**

1. Admin chama `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
2. Middleware `requireRole('ADMIN')` verifica role (`src/middlewares/auth.middleware.ts`)
3. Sistema cria nova linha em `webhook_outbox` com `status = PENDING` e `attempt_count = 0`
4. Histórico original em `webhook_dead_letter` é preservado como trilha de auditoria
5. Log estruturado registra `admin_id`, `dead_letter_id` e timestamp da ação

**Diagrama de estados do evento na outbox**

```
PENDING
  └─ worker processa ──► PROCESSING
                              ├─ 2xx ──────────────────────────► DELIVERED
                              ├─ falha + attempt < 5 ──────────► PENDING (retry_at agendado)
                              └─ falha + attempt >= 5 ──────────► (move para webhook_dead_letter)
```

**Schema das tabelas principais**

`webhooks` — configuração de cada endpoint de cliente:
```
id                       CHAR(36) PK UUID
customer_id              CHAR(36) FK → customers.id
url                      VARCHAR(2048) NOT NULL  -- HTTPS obrigatório
events                   JSON NOT NULL           -- lista de status: ["SHIPPED","DELIVERED"]
active                   BOOLEAN DEFAULT true
secret                   VARCHAR(255) NOT NULL   -- secret atual (nunca exposta via GET)
previous_secret          VARCHAR(255) NULL       -- secret anterior durante grace period
previous_secret_expires_at DATETIME NULL         -- NULL = sem rotação ativa
created_at               DATETIME
updated_at               DATETIME
```

`webhook_outbox` — fila de eventos a despachar:
```
id                CHAR(36) PK UUID
webhook_id        CHAR(36) FK → webhooks.id     -- endereçado na inserção, não no despacho
order_id          CHAR(36) FK → orders.id
payload           JSON NOT NULL                  -- snapshot do estado no momento da inserção
status            ENUM('PENDING','PROCESSING','DELIVERED')
attempt_count     INT DEFAULT 0
retry_at          DATETIME NULL                  -- NULL = disponível imediatamente
created_at        DATETIME
updated_at        DATETIME
INDEX(status, retry_at)   -- leitura do worker: WHERE status='PENDING' AND (retry_at IS NULL OR retry_at <= NOW())
INDEX(order_id)
```

`webhook_dead_letter` — eventos que esgotaram todas as tentativas:
```
id                CHAR(36) PK UUID
webhook_id        CHAR(36) FK → webhooks.id
order_id          CHAR(36) FK → orders.id
outbox_id         CHAR(36)                       -- referência ao id original na outbox (sem FK para permitir archival)
payload           JSON NOT NULL
reason            VARCHAR(500) NOT NULL           -- ex: "HTTP 503 after 5 attempts", "PAYLOAD_TOO_LARGE"
failed_at         DATETIME NOT NULL
replayed_at       DATETIME NULL                  -- preenchido se replay foi executado
replayed_by       CHAR(36) NULL                  -- user_id do admin que executou o replay
```

`webhook_deliveries` — histórico de tentativas (bem-sucedidas e falhas):
```
id                CHAR(36) PK UUID
outbox_id         CHAR(36) FK → webhook_outbox.id
webhook_id        CHAR(36) FK → webhooks.id
attempt           INT NOT NULL
http_status       INT NULL                       -- NULL se timeout ou erro de rede
duration_ms       INT NULL
response_body     TEXT NULL                      -- primeiros 1KB da resposta para debug
attempted_at      DATETIME NOT NULL
```

---

### 5. Contratos públicos (assinaturas, endpoints, headers, exemplos)

---

**Contrato 1: Cadastrar webhook**

- Tipo: `http_endpoint`
- Rota: `POST /api/v1/webhooks`
- Auth: Bearer JWT (qualquer role autenticada)
- Semântica de status:
  - `201 Created` — webhook criado; secret retornada apenas nesta resposta
  - `400 Bad Request` — URL inválida, lista de eventos vazia, corpo malformado (`WEBHOOK_INVALID_URL`, `WEBHOOK_EVENTS_REQUIRED`)
  - `401 Unauthorized` — token ausente ou inválido
  - `409 Conflict` — URL já cadastrada para o mesmo `customerId` (`WEBHOOK_DUPLICATE_URL`)

**Exemplo de requisição**
```json
POST /api/v1/webhooks
Authorization: Bearer <token>
Content-Type: application/json

{
  "customerId": "cus_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "url": "https://integracoes.atlascomercial.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Exemplo de resposta**
```json
HTTP/1.1 201 Created

{
  "id": "wh_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "customerId": "cus_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "url": "https://integracoes.atlascomercial.com.br/webhooks/oms",
  "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "secret": "whsec_4f3a9b2c1d8e7f6a5b4c3d2e1f0a9b8c",
  "createdAt": "2026-06-24T09:00:00.000Z"
}
```

> A `secret` é retornada **apenas** nesta resposta. Chamadas `GET` subsequentes nunca a expõem.

---

**Contrato 2: Listar webhooks do customer**

- Tipo: `http_endpoint`
- Rota: `GET /api/v1/webhooks?customerId=:id`
- Auth: Bearer JWT (qualquer role autenticada)
- Semântica de status:
  - `200 OK` — lista paginada de webhooks (sem o campo `secret`)
  - `400 Bad Request` — `customerId` ausente (`WEBHOOK_CUSTOMER_REQUIRED`)
  - `401 Unauthorized` — token ausente ou inválido

**Exemplo de resposta**
```json
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "wh_01J2K3M4N5P6Q7R8S9T0U1V2W3",
      "customerId": "cus_01J2K3M4N5P6Q7R8S9T0U1V2W3",
      "url": "https://integracoes.atlascomercial.com.br/webhooks/oms",
      "events": ["SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-06-24T09:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

---

**Contrato 3: Histórico de entregas**

- Tipo: `http_endpoint`
- Rota: `GET /api/v1/webhooks/:id/deliveries`
- Auth: Bearer JWT (qualquer role autenticada)
- Semântica de status:
  - `200 OK` — lista paginada de tentativas de entrega
  - `401 Unauthorized` — token ausente ou inválido
  - `404 Not Found` — webhook não encontrado (`WEBHOOK_NOT_FOUND`)

**Exemplo de resposta**
```json
HTTP/1.1 200 OK

{
  "data": [
    {
      "id": "del_01J2K3M4N5P6Q7R8S9T0U1V2W3",
      "eventId": "evt_01J2K3M4N5P6Q7R8S9T0U1V2W3",
      "eventType": "order.status_changed",
      "status": "DELIVERED",
      "httpStatus": 200,
      "attemptCount": 1,
      "durationMs": 312,
      "deliveredAt": "2026-06-24T09:00:04.312Z"
    },
    {
      "id": "del_02J2K3M4N5P6Q7R8S9T0U1V2W4",
      "eventId": "evt_02J2K3M4N5P6Q7R8S9T0U1V2W4",
      "eventType": "order.status_changed",
      "status": "FAILED",
      "httpStatus": 503,
      "attemptCount": 5,
      "lastErrorAt": "2026-06-24T21:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 2, "totalPages": 1 }
}
```

---

**Contrato 4: Replay de evento na DLQ**

- Tipo: `http_endpoint`
- Rota: `POST /api/v1/admin/webhooks/dead-letter/:id/replay`
- Auth: Bearer JWT, role `ADMIN` obrigatória (verificada por `requireRole('ADMIN')` em `src/middlewares/auth.middleware.ts`)
- Semântica de status:
  - `200 OK` — evento reinserido no outbox com `status = PENDING`
  - `401 Unauthorized` — token ausente ou inválido
  - `403 Forbidden` — role insuficiente (`WEBHOOK_FORBIDDEN`)
  - `404 Not Found` — ID não encontrado na DLQ (`WEBHOOK_DEAD_LETTER_NOT_FOUND`)

**Exemplo de requisição**
```json
POST /api/v1/admin/webhooks/dead-letter/dl_01J2K3M4N5P6Q7R8S9T0U1V2W3/replay
Authorization: Bearer <admin_token>
```

**Exemplo de resposta**
```json
HTTP/1.1 200 OK

{
  "outboxId": "evt_03J2K3M4N5P6Q7R8S9T0U1V2W5",
  "deadLetterId": "dl_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "status": "PENDING",
  "replayedAt": "2026-06-25T08:00:00.000Z",
  "replayedBy": "usr_admin_01"
}
```

---

**Contrato 5: Payload entregue ao endpoint do cliente (outbound)**

- Tipo: `http_endpoint` (outbound, executado pelo worker)
- Rota: URL configurada pelo cliente (HTTPS obrigatório)
- Método: `POST`
- Timeout: 10 segundos
- Tamanho máximo do body: 64KB
- Headers enviados pelo worker:

| Header | Valor |
|---|---|
| `X-Event-Id` | UUID do evento, gerado na inserção do outbox, estável entre retries |
| `X-Webhook-Id` | ID do endpoint webhook de destino |
| `X-Signature` | `sha256=<HMAC-SHA256(rawBody, secret)>` |
| `X-Timestamp` | Timestamp ISO 8601 do momento do envio |
| `Content-Type` | `application/json` |

**Exemplo de body entregue ao cliente**
```json
{
  "event_id": "evt_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "event_type": "order.status_changed",
  "timestamp": "2026-06-24T09:00:02.000Z",
  "order_id": "ord_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "order_number": "ORD-000042",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "cus_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "total_cents": 149900
}
```

> `items` não são incluídos no payload. Cliente que precisar de detalhes consulta `GET /api/v1/orders/:id`.

---

**Contrato 6: Rotação de secret**

- Tipo: `http_endpoint`
- Rota: `POST /api/v1/webhooks/:id/rotate-secret`
- Auth: Bearer JWT (qualquer role autenticada)
- Semântica de status:
  - `200 OK` — nova secret gerada; retorna apenas a nova secret
  - `401 Unauthorized` — token ausente ou inválido
  - `404 Not Found` — webhook não encontrado (`WEBHOOK_NOT_FOUND`)

**Exemplo de resposta**
```json
HTTP/1.1 200 OK

{
  "secret": "whsec_9d8e7f6a5b4c3d2e1f0a9b8c7d6e5f4a",
  "previousSecretExpiresAt": "2026-06-25T09:00:00.000Z"
}
```

---

### 6. Erros, exceções e fallback

**Matriz de erros**

| Código | HTTP | Condição | Tratamento |
|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | ID de webhook inexistente | Responde 404; nenhuma ação |
| `WEBHOOK_INVALID_URL` | 400 | URL não é HTTPS ou malformada | Rejeita no schema Zod antes de persistir |
| `WEBHOOK_EVENTS_REQUIRED` | 400 | Lista de eventos vazia ou ausente | Rejeita no schema Zod |
| `WEBHOOK_DUPLICATE_URL` | 409 | URL já cadastrada para o mesmo `customerId` | Prisma P2002; error middleware mapeia para 409 |
| `WEBHOOK_CUSTOMER_REQUIRED` | 400 | `customerId` ausente na query | Rejeita no schema Zod |
| `WEBHOOK_SECRET_REQUIRED` | 400 | Tentativa de criar webhook sem secret gerada | Erro de invariante interno |
| `WEBHOOK_FORBIDDEN` | 403 | Role insuficiente para operação ADMIN | `requireRole` em `src/middlewares/auth.middleware.ts` lança `ForbiddenError` |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | ID não encontrado em `webhook_dead_letter` | Responde 404 |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — | Payload > 64KB no worker | Worker move evento direto para DLQ sem tentativa de envio |
| `WEBHOOK_DELIVERY_TIMEOUT` | — | Cliente não responde em 10s | Worker trata como falha e agenda retry |

Todos os erros seguem o envelope padrão do projeto (definido em `src/middlewares/error.middleware.ts`):
```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found",
    "details": {}
  }
}
```

**Estratégias de resiliência**

- **Timeout**: 10 segundos por chamada HTTP do worker. Expiração tratada como falha; incrementa `attempt_count`.
- **Retry com backoff exponencial**: 5 tentativas com intervalos 1m / 5m / 30m / 2h / 12h. Janela total de aproximadamente 14h36m.
- **DLQ**: após 5 falhas, evento persiste em `webhook_dead_letter` com payload, motivo e timestamp. Não é descartado.
- **Idempotência do worker**: evento marcado como `PROCESSING` antes do envio. Em caso de crash do worker, evento fica preso em `PROCESSING`; job de recuperação (a implementar) deve resetar eventos `PROCESSING` com `updated_at` antigo.

**Invariantes críticos**

- `event_id` é gerado na inserção do outbox e nunca muda entre tentativas de retry.
- A inserção em `webhook_outbox` ocorre sempre dentro da transação Prisma de `changeStatus`. Inserção fora da transação quebra a atomicidade silenciosamente.
- Se nenhum webhook ativo do customer tem o `toStatus` na lista de eventos, nenhuma linha é inserida no outbox. Esse filtro é irreversível: o evento não pode ser entregue retroativamente.

---

### 7. Observabilidade

**Métricas**

- `webhook_outbox_pending_count` (gauge): número de eventos com `status = PENDING` no outbox. Alerta se acima de threshold configurável por 5 minutos (indica worker parado).
- `webhook_delivery_duration_ms` (histogram): tempo de resposta HTTP por entrega, por `webhook_id` e `http_status`.
- `webhook_delivery_success_total` (counter): entregas com `2xx`, por `webhook_id`.
- `webhook_delivery_failure_total` (counter): falhas por tentativa, por `webhook_id` e `http_status`.
- `webhook_dead_letter_total` (counter): eventos movidos para DLQ, por `webhook_id`.
- `webhook_worker_poll_duration_ms` (histogram): tempo de cada ciclo de polling do worker.

**Logs**

Formato: JSON estruturado via Pino (`src/shared/logger/index.ts`). Campos obrigatórios em cada linha de log do worker:

```json
{
  "level": "info",
  "time": "2026-06-24T09:00:04.312Z",
  "event_id": "evt_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "webhook_id": "wh_01J2K3M4N5P6Q7R8S9T0U1V2W3",
  "attempt": 1,
  "http_status": 200,
  "duration_ms": 312,
  "msg": "webhook delivered"
}
```

Campos sensíveis nunca logados (herdado da configuração de redação do Pino em `src/shared/logger/index.ts`): `secret`, `previous_secret`, `authorization`, `X-Signature`.

Eventos de log obrigatórios:
- `webhook delivered` — entrega com `2xx`
- `webhook delivery failed` — falha com motivo e `attempt_count`
- `webhook moved to dead letter` — após 5 falhas
- `webhook replayed` — replay ADMIN com `admin_id` e `dead_letter_id`
- `worker poll cycle` — a cada ciclo, com contagem de eventos processados

**Tracing**

Spans principais por ciclo do worker:
- `worker.poll` — duração do SELECT no outbox
- `worker.dispatch` — por evento: inclui `event_id`, `webhook_id`, `attempt`
- `worker.http_call` — chamada HTTP ao cliente, inclui URL (sem secret), `http_status`, `duration_ms`

Taxa de amostragem e backend de tracing a definir pela equipe de plataforma antes do go-live. Prioritariamente, spans de eventos que entram na DLQ e de erros HTTP `>= 500` devem ser retidos para diagnóstico.

Dados sensíveis excluídos de todos os spans: `X-Signature`, `secret`, `previous_secret` e body do payload nunca aparecem em atributos de span.

**Dashboards e alertas mínimos**

- Painel: `webhook_outbox_pending_count` por tempo — detecta acúmulo por worker parado
- Painel: taxa de sucesso de entregas por `webhook_id` — detecta endpoints de clientes degradados
- Alerta: `webhook_dead_letter_total` > 0 nos últimos 5 minutos — requer ação operacional
- Alerta: `webhook_outbox_pending_count` > 100 por mais de 5 minutos — worker provavelmente parado

---

### 8. Dependências e compatibilidade

| Componente | Versão mínima | Observações |
|---|---|---|
| Node.js | 20 | Runtime do worker e da API |
| TypeScript | 5.6.3 | Tipagem estrita; `ESNext` + `ES2022` target |
| Prisma | 5.22.0 | ORM; worker instancia `PrismaClient` separado |
| MySQL | 8.0 | Banco principal; tabelas `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries`, `webhooks` |
| Express | 4.21.1 | Framework HTTP da API; worker não usa Express |
| Zod | 3.23.8 | Validação de schemas; usado em `webhook.schemas.ts` |
| `crypto` (Node built-in) | 20 | HMAC-SHA256; sem dependência externa |
| `uuid` | 11.0.3 | Geração de `event_id` e secrets |

**Integração com o sistema existente**

Esta é a única seção do FDD que descreve pontos de toque com código já existente no repositório.

1. **`src/modules/orders/order.service.ts` — método `changeStatus` (linha ~126)**
   Ponto de integração central. A função `publishWebhookEvent(tx, order, fromStatus, toStatus)` é importada de `src/modules/webhooks/webhook.publisher.ts` e chamada dentro do bloco `this.prisma.$transaction(async (tx) => { ... })` existente, após as operações de status e estoque. Recebe o `tx` (client de transação Prisma) como primeiro argumento para garantir atomicidade. A função é pura no sentido de não depender de `this` do `OrderService` — não exige injeção de `WebhookRepository` no construtor de `OrderService`; é importação estática de módulo (`TRANSCRICAO.md:242-244`).

2. **`src/middlewares/auth.middleware.ts` — funções `authenticate` e `requireRole`**
   O endpoint de replay de DLQ (`POST /api/v1/admin/webhooks/dead-letter/:id/replay`) aplica `requireRole('ADMIN')` via o middleware existente. Nenhuma alteração neste arquivo; apenas uso na definição das rotas do módulo webhooks.

3. **`src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts`**
   Todos os erros do módulo webhooks estendem `AppError` seguindo o padrão do projeto. Novos erros específicos (ex: `WebhookNotFoundError`, `WebhookInvalidUrlError`) são adicionados a `http-errors.ts` com códigos prefixados `WEBHOOK_`. O `errorMiddleware` em `src/middlewares/error.middleware.ts` já trata `AppError` sem modificação.

4. **`src/config/database.ts` — `createPrismaClient()`**
   O worker instancia seu próprio `PrismaClient` usando a factory `createPrismaClient()` existente, apontando para a mesma `DATABASE_URL`. PrismaClient não é compartilhado entre processos; cada processo Node.js mantém sua própria instância e pool de conexão.

5. **`src/middlewares/validate.middleware.ts`**
   O módulo webhooks usa o middleware `validate()` existente para validação de body, query e params com schemas Zod — sem modificação no middleware.

**Garantias de compatibilidade**

- A inserção em `webhook_outbox` dentro de `changeStatus` não altera a semântica da resposta da API de pedidos. O cliente da API continua recebendo o mesmo payload de resposta.
- Nenhuma coluna existente nas tabelas `orders`, `order_status_history` ou `products` é alterada.
- O módulo webhooks é aditivo: novas tabelas, novo módulo, novo processo. Rollback é possível removendo as tabelas e o módulo sem impacto no restante do sistema.

---

### 9. Critérios de aceite técnicos

- Mudança de status de pedido resulta em evento entregue ao endpoint do cliente em menos de 10 segundos em condições normais de rede
- Rollback da transação de `changeStatus` não cria nenhuma linha em `webhook_outbox`
- Se nenhum webhook do customer tem o `toStatus` inscrito, nenhuma linha é inserida em `webhook_outbox` — o filtro ocorre na inserção, não no despacho
- Evento com `2xx` do cliente é marcado como `DELIVERED` e não é reenviado
- Evento com falha segue o backoff exato: 1m / 5m / 30m / 2h / 12h
- Após 5 falhas consecutivas, evento está em `webhook_dead_letter` com payload, motivo e timestamp corretos
- Replay de DLQ por usuário ADMIN cria nova linha em `webhook_outbox` com `attempt_count = 0`; registro original em `webhook_dead_letter` é preservado com `replayed_at` e `replayed_by` preenchidos
- Log de replay contém `admin_id`, `dead_letter_id` e timestamp
- `X-Event-Id` do mesmo evento é idêntico em todas as tentativas de retry
- Endpoint com URL `http://` é recusado com `WEBHOOK_INVALID_URL` na criação
- Payload com mais de 64KB não é enviado ao cliente; evento vai direto para DLQ com motivo `WEBHOOK_PAYLOAD_TOO_LARGE`
- Secret nunca aparece em resposta de `GET /api/v1/webhooks` nem em nenhum log
- Durante grace period de 24h após rotação, o worker aceita verificação de assinatura gerada tanto com `secret` atual quanto com `previous_secret` — ambas são válidas até `previous_secret_expires_at`
- O worker, quando reiniciado após crash, reseta eventos presos em `PROCESSING` com `updated_at` mais antigo que 30 segundos para `PENDING` antes de iniciar o ciclo normal
- Worker e API são processos Node.js independentes: restart ou crash da API não interrompe o ciclo de polling do worker

---

### 10. Riscos e mitigação

### Worker para silenciosamente sem alerta

- **Probabilidade:** média
- **Impacto:** eventos acumulam em `webhook_outbox` sem entrega; clientes ficam desatualizados por tempo indeterminado
- **Mitigação:**
  - Alerta em `webhook_outbox_pending_count` > 100 por mais de 5 minutos
  - Supervisão de processo configurada antes do go-live (systemd, Docker restart policy ou PM2)
  - Log de ciclo de polling a cada iteração do worker
- **Plano de contingência:** restart manual do worker; eventos em `PROCESSING` com `updated_at` antigo são resetados para `PENDING` por job de recuperação

### Eventos presos em PROCESSING por crash do worker

- **Probabilidade:** baixa
- **Impacto:** eventos nunca processados, nunca chegam à DLQ; ficam invisíveis para o sistema
- **Mitigação:**
  - Job periódico (ou verificação inline no início de cada ciclo) que reseta eventos em `PROCESSING` com `updated_at > 30s` para `PENDING`
- **Plano de contingência:** reset manual via SQL; evento volta ao fluxo normal de retry

### Cliente não implementa idempotência e processa duplicatas

- **Probabilidade:** alta (para clientes que não leram a documentação)
- **Impacto:** duplicação de operações no sistema do cliente (ex: confirmar pedido duas vezes)
- **Mitigação:**
  - Documentação clara no portal do desenvolvedor com exemplos de deduplicação por `X-Event-Id` (Marcos, `TRANSCRICAO.md:156`)
  - `X-Event-Id` estável entre retries é invariante verificável em testes de integração
- **Plano de contingência:** suporte ativo na fase de onboarding dos três clientes iniciais

### Vazamento de secret por cliente

- **Probabilidade:** baixa (já ocorreu com um cliente anterior, `TRANSCRICAO.md:122`)
- **Impacto:** ator malicioso pode forjar eventos para aquele cliente específico
- **Mitigação:**
  - Secret por endpoint: vazamento limita o raio de explosão a um único webhook
  - Rotação disponível imediatamente via `POST /api/v1/webhooks/:id/rotate-secret`
  - Grace period de 24h permite migração sem downtime
- **Plano de contingência:** revogar secret imediatamente via rotação; cliente atualiza verificação antes do fim do grace period

### Crescimento descontrolado da tabela `webhook_outbox`

- **Probabilidade:** média (sem archival implementado)
- **Impacto:** degradação de performance das queries do worker ao longo do tempo
- **Mitigação:**
  - Índices em `status` e `created_at` na tabela `webhook_outbox`
  - Archival de linhas `DELIVERED` com mais de 30 dias (adiado, implementar na fase seguinte)
- **Plano de contingência:** script de limpeza manual executado por DBA antes que o volume impacte performance
