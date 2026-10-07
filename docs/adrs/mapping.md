# Codebase Architecture Mapping

## Project Overview

- **Name**: order-management-api
- **Purpose**: B2B Order Management System REST API — manages customers, products, orders, and user authentication. A webhook notification feature for outbound order-status events is in active design.
- **Type**: Backend REST API, Node.js monolith
- **Languages**: TypeScript (strict, ESNext modules, ES2022 target)
- **Primary Framework**: Express 4.x
- **Runtime**: Node.js >= 20
- **Repository age**: ~4 months (first commit: 2026-06-24)

---

## Technology Stack

| Layer | Technology | Version |
|---|---|---|
| Runtime | Node.js | >= 20 |
| Language | TypeScript | 5.6.3 |
| Web Framework | Express | 4.21.1 |
| ORM | Prisma | 5.22.0 |
| Database | MySQL | 8.0 (via Docker) |
| Validation | Zod | 3.23.8 |
| Auth | JWT (jsonwebtoken) | 9.0.2 |
| Password hashing | bcrypt | 5.1.1 |
| Logging | Pino + pino-http | 9.5.0 / 10.3.0 |
| UUIDs | uuid | 11.0.3 |
| Testing | Vitest + Supertest | 2.1.4 / 7.0.0 |
| Dev runner | tsx (watch) | 4.19.2 |
| Linting | ESLint + typescript-eslint | 8.57.1 |
| Formatting | Prettier | 3.3.3 |
| Container | Docker Compose | MySQL service only |

---

## Context Notes

**Source Files Analyzed**: `decisions.md`, `TRANSCRICAO.md`

**Key Insights**:

- **Architectural patterns documented**: Transactional Outbox pattern, Exponential Backoff Retry, Dead Letter Queue (DLQ), HMAC-SHA256 payload signing, at-least-once delivery with idempotency via `X-Event-Id`, snapshot payload on outbox insertion.
- **Business domains identified**: Orders (lifecycle + status FSM), Customers (B2B), Products (inventory), Users (internal operators/admins), Webhooks (outbound notifications — in design).
- **Module boundaries documented**: The meeting explicitly confirms `src/modules/webhooks` following the existing `controller / service / repository / routes / schemas` convention. A `src/worker.ts` entry point is planned for a separate process.
- **Technologies documented vs. discovered**: MySQL outbox chosen explicitly over Redis Streams / broker (Kafka/RabbitMQ) — rejected as overengineering for team size. No Redis or external broker is present in the current codebase.
- **Discrepancies**: The `src/modules/webhooks` module, `src/worker.ts`, and tables `webhook_outbox` / `webhook_dead_letter` documented in `decisions.md` do not yet exist in the codebase. They represent planned but unimplemented work.
- **Deferred decisions**: archival of delivered outbox rows (30 days), email fallback on consecutive failures, rate limiting of outbound webhook sends, multi-worker partitioned scaling, visual dashboard.
- **Discarded options (recorded)**: synchronous webhook inside order service, Redis Streams / external broker, MySQL trigger-based reactive notification, exactly-once delivery guarantee, single global HMAC secret.

---

## System Modules

### Module Index

| ID | Name | Description |
|---|---|---|
| AUTH | Authentication | Login, registration, JWT issuance and middleware verification |
| USERS | User Management | Internal operator/admin CRUD |
| CUSTOMERS | Customer Management | B2B customer CRUD |
| PRODUCTS | Product Catalog | Product CRUD and inventory (stock quantity) |
| ORDERS | Order Lifecycle | Order creation, status FSM, stock debit/replenish, status history |
| WEBHOOKS | Webhook Notifications | Outbound webhook configuration CRUD, outbox, worker, DLQ, replay (PLANNED) |
| SHARED | Shared Kernel | AppError hierarchy, HTTP response helpers, Pino logger |
| INFRA | Infrastructure | Express bootstrap, middleware stack, Prisma client, env config, Docker |
| DATA | Data Layer | Prisma schema, migrations, seed — MySQL 8.0 |
| TEST | Test Suite | Integration tests, Vitest configuration, test factories |

---

### AUTH: Authentication

**Purpose**: Issues JWT access tokens on login; provides `authenticate` and `requireRole` middleware used by all protected routes.

**Location**: `src/modules/auth/*`, `src/middlewares/auth.middleware.ts`

**Key Components**:
- `AuthService` — bcrypt password comparison, JWT signing
- `AuthController` — `POST /auth/login`, `POST /auth/register`
- `authenticate` middleware — Bearer token extraction and `jwt.verify`
- `requireRole(...roles)` middleware — role-based guard (ADMIN | OPERATOR)

**Technologies**: jsonwebtoken 9.0.2, bcrypt 5.1.1, Zod (schema validation)

**Dependencies**:
- Internal: USERS module (UserRepository, UserService)
- External: JWT_SECRET, JWT_EXPIRES_IN env vars

**Patterns**: Stateless JWT (no token revocation / refresh token), role-based access control (RBAC) with two roles

**Key Files**:
- `src/modules/auth/auth.service.ts`
- `src/middlewares/auth.middleware.ts`
- `src/modules/auth/auth.schemas.ts`

**Scope**: Small — 4 files

---

### USERS: User Management

**Purpose**: Manages internal platform users (ADMIN and OPERATOR roles). Provides user creation with bcrypt-hashed passwords.

**Location**: `src/modules/users/*`

**Key Components**:
- `UserRepository` — Prisma-backed CRUD
- `UserService` — password hashing (bcrypt, 10 rounds), `toPublic()` projection
- `UserController` — REST handlers

**Technologies**: bcrypt, Prisma, Zod

**Dependencies**:
- Internal: SHARED (AppError, logger), DATA (Prisma schema `users` table)
- External: none

**Patterns**: Repository pattern, service layer projection (`PublicUser` strips `passwordHash`)

**Key Files**:
- `src/modules/users/user.service.ts`
- `src/modules/users/user.repository.ts`

**Scope**: Small — 5 files

---

### CUSTOMERS: Customer Management

**Purpose**: CRUD for B2B customer records. Stores name, email, phone, document, and a JSON address field.

**Location**: `src/modules/customers/*`

**Key Components**:
- `CustomerRepository`, `CustomerService`, `CustomerController`
- Zod schemas for input validation

**Technologies**: Prisma, Zod

**Dependencies**:
- Internal: SHARED, DATA (`customers` table, `document` index)
- External: none

**Patterns**: Repository pattern, controller/service/repository layering

**Key Files**:
- `src/modules/customers/customer.repository.ts`
- `src/modules/customers/customer.schemas.ts`

**Scope**: Small — 5 files

---

### PRODUCTS: Product Catalog

**Purpose**: Product CRUD including SKU, pricing in cents, stock quantity, and active flag.

**Location**: `src/modules/products/*`

**Key Components**:
- `ProductRepository`, `ProductService`, `ProductController`
- Stock quantity tracked as integer; incremented/decremented by ORDERS module within transactions

**Technologies**: Prisma, Zod

**Dependencies**:
- Internal: SHARED, DATA (`products` table), ORDERS (stock mutation via Prisma transactions)
- External: none

**Patterns**: Repository pattern; prices stored as integer cents (no floating point)

**Key Files**:
- `src/modules/products/product.repository.ts`
- `src/modules/products/product.schemas.ts`

**Scope**: Small — 5 files

---

### ORDERS: Order Lifecycle

**Purpose**: Core business domain. Creates orders, enforces a strict status FSM, manages stock debit/replenish, maintains status history with audit trail, and generates sequential order numbers (ORD-XXXXXX).

**Location**: `src/modules/orders/*`

**Key Components**:
- `OrderService` — all business logic, Prisma interactive transactions
- `OrderRepository` — paginated list with filters, find with relations
- `order.status.ts` — FSM transition table (`canTransition`, `isTerminal`, `shouldDebitStock`, `shouldReplenishStock`)
- `OrderController` — REST handlers (list, get, create, changeStatus, delete)

**Technologies**: Prisma transactions (`$transaction`), Zod

**Dependencies**:
- Internal: SHARED, DATA (`orders`, `order_items`, `order_status_history`, `order_number_sequence` tables), PRODUCTS (stock mutation)
- External: none
- Planned dependency: WEBHOOKS (`publishWebhookEvent(tx, order, fromStatus, toStatus)` to be called inside `changeStatus` transaction)

**Patterns**:
- Finite State Machine for order status
- Interactive Prisma transactions for multi-step atomicity
- Sequential order number via `OrderNumberSequence` upsert inside transaction (optimistic counter)
- Prices in integer cents

**Key Files**:
- `src/modules/orders/order.service.ts`
- `src/modules/orders/order.status.ts`
- `src/modules/orders/order.repository.ts`

**Scope**: Medium — 5 files, ~255 lines in service

---

### WEBHOOKS: Webhook Notifications (PLANNED)

**Purpose**: Outbound webhook notification system — allows B2B customers to subscribe to order status change events. Delivers signed HTTP POST requests with at-least-once semantics.

**Location**: `src/modules/webhooks/*` (DOES NOT EXIST YET), `src/worker.ts` (DOES NOT EXIST YET)

**Planned Key Components**:
- `WebhookRepository` — CRUD on `webhooks` configuration table
- `WebhookService` — registration, secret generation/rotation with 24h grace period
- `WebhookController` — `POST/PATCH/DELETE/GET /webhooks`, `GET /webhooks/:id/deliveries`
- `publishWebhookEvent(tx, order, fromStatus, toStatus)` — inserts into `webhook_outbox` inside the ORDERS `changeStatus` transaction
- `WebhookWorker / WebhookProcessor` — polls `webhook_outbox` every 2 seconds, dispatches HTTP POST, manages retry with exponential backoff (1m/5m/30m/2h/12h, 5 attempts)
- Dead Letter Queue — `webhook_dead_letter` table; replay via `POST /admin/webhooks/dead-letter/:id/replay` (ADMIN role)

**Planned Tables**: `webhooks`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`

**Technologies**: Node.js `fetch` / `http`, HMAC-SHA256 (Node.js `crypto`), UUID, Prisma (separate PrismaClient instance for worker process)

**Dependencies**:
- Internal: SHARED (AppError with `WEBHOOK_` prefix, Pino, error middleware), AUTH (requireRole ADMIN for replay), ORDERS (integration point in `changeStatus`), DATA
- External: Customer endpoint URLs (HTTPS only, validated by Zod)

**Patterns**:
- Transactional Outbox
- Polling worker (separate Node.js process)
- Exponential backoff retry
- Dead Letter Queue
- HMAC-SHA256 payload signing, per-endpoint secret
- Secret rotation with grace period
- At-least-once delivery with `X-Event-Id` for client-side deduplication
- Snapshot payload on outbox insertion

**Scope**: Large (planned) — estimated 3 sprints

---

### SHARED: Shared Kernel

**Purpose**: Cross-cutting utilities reused by all modules: error hierarchy, HTTP response helpers, structured logger.

**Location**: `src/shared/*`

**Key Components**:
- `AppError` — base error class (`statusCode`, `errorCode`, `details`)
- `src/shared/errors/http-errors.ts` — typed subclasses: `NotFoundError`, `UnauthorizedError`, `ForbiddenError`, `ConflictError`, `ValidationError`, `UnprocessableEntityError`, `InsufficientStockError`, `InvalidStatusTransitionError`
- `src/shared/http/response.ts` — `paginated()`, `PaginatedResponse<T>`, `buildPagination()`
- `src/shared/logger/index.ts` — Pino logger with redaction of sensitive paths (`authorization`, `password`, `passwordHash`, `token`, `accessToken`)

**Technologies**: Pino 9.5.0, pino-pretty (dev), TypeScript

**Dependencies**: Internal only (no external module imports)

**Patterns**: Error-code-per-class pattern, structured logging with field redaction

**Key Files**:
- `src/shared/errors/app-error.ts`
- `src/shared/logger/index.ts`

**Scope**: Small — 5 files

---

### INFRA: Infrastructure

**Purpose**: Application bootstrap, middleware pipeline, environment validation, Prisma client singleton, Docker Compose service definition.

**Location**: `src/app.ts`, `src/server.ts`, `src/config/*`, `src/middlewares/*`, `docker-compose.yml`

**Key Components**:
- `buildApp(deps)` — composes Express app: disables `x-powered-by`, JSON body parser (1MB limit), request logger, API router, 404 handler, error middleware
- `buildControllers(prisma)` — DI composition root; constructs all module repositories, services, and controllers
- `src/config/env.ts` — Zod-validated environment schema (fail-fast on startup if invalid)
- `src/config/database.ts` — `createPrismaClient()` factory and singleton `prisma` export
- `src/middlewares/validate.middleware.ts` — generic Zod-backed request validation middleware (body / query / params)
- `src/middlewares/request-logger.middleware.ts` — pino-http request/response logging
- `src/middlewares/error.middleware.ts` — centralized error handler: `AppError`, `ZodError`, `PrismaClientKnownRequestError` (P2002 conflict, P2025 not-found)
- `docker-compose.yml` — MySQL 8.0 service with healthcheck, utf8mb4, persistent volume

**Technologies**: Express, Prisma, Zod, Pino, Docker Compose

**Dependencies**: All modules (composition root)

**Patterns**: Explicit dependency injection (constructor injection at `buildControllers`), fail-fast env validation, centralized error mapping

**Key Files**:
- `src/app.ts`
- `src/server.ts`
- `src/config/env.ts`
- `src/middlewares/error.middleware.ts`
- `docker-compose.yml`

**Scope**: Small — 7 files

---

### DATA: Data Layer

**Purpose**: Prisma schema definition, migrations, and seed data. Defines all database models and relationships.

**Location**: `prisma/*`

**Key Components**:
- `prisma/schema.prisma` — datasource (MySQL), models: `User`, `Customer`, `Product`, `Order`, `OrderItem`, `OrderStatusHistory`, `OrderNumberSequence`
- `prisma/migrations/` — migration lock (Prisma Migrate)
- `prisma/seed.ts` — 10 customers, 20 products, 26 orders across all statuses

**Technologies**: Prisma 5.22.0, MySQL 8.0, bcrypt (seed)

**Dependencies**: External — MySQL 8.0 (Docker)

**Patterns**:
- UUID primary keys (`@id @default(uuid()) @db.Char(36)`) on all entities
- Monetary values stored as integer cents (`priceCents`, `totalCents`, `discountCents`)
- JSON type for `Customer.address`
- Cascade deletes on `OrderItem` and `OrderStatusHistory` via `Order`
- Strategic indexes: `status`, `createdAt`, `customerId`, `createdById`, `orderId`, `productId`, `document`
- `OrderNumberSequence` table for atomic sequential number generation

**Key Files**:
- `prisma/schema.prisma`
- `prisma/seed.ts`

**Scope**: Small — 3 files + migrations

---

### TEST: Test Suite

**Purpose**: Integration tests running against a real MySQL database. Covers authentication flows and order lifecycle flows.

**Location**: `tests/*`

**Key Components**:
- `tests/auth.test.ts` — auth endpoint tests
- `tests/orders.test.ts` — order CRUD and status transition tests
- `tests/setup.ts` — global Vitest hooks: `$connect`, full table truncation before each test, `$disconnect` after all
- `tests/helpers/factories.ts` — test data factories

**Technologies**: Vitest 2.1.4, Supertest 7.0.0, Prisma (real DB, no mocking)

**Dependencies**:
- External: running MySQL instance (requires Docker)
- Internal: all modules (tests exercise full HTTP stack)

**Patterns**:
- Integration tests only (no unit tests or mocks)
- `fileParallelism: false`, `singleFork: true` — serial execution to avoid DB state conflicts
- Full table wipe before each test (not transactions / rollback)

**Key Files**:
- `vitest.config.ts`
- `tests/setup.ts`

**Scope**: Small — 4 files

---

## Cross-Cutting Concerns

### Authentication and Authorization
- JWT stateless auth via `authenticate` middleware applied to all `/api/v1` routes except `POST /auth/login` and `POST /auth/register`
- Two roles: `ADMIN`, `OPERATOR`; `requireRole()` middleware enforces role guards
- Sensitive fields redacted from logs by Pino (`authorization`, `password`, `passwordHash`, `token`, `accessToken`)
- Planned: ADMIN role required for `POST /admin/webhooks/dead-letter/:id/replay`

### Data Layer
- Single Prisma client singleton for API process
- Separate PrismaClient instance planned for worker process (same `DATABASE_URL`, different Node.js process)
- Interactive transactions (`$transaction`) used for multi-step operations in ORDERS module
- All primary keys: UUID (Char 36)
- All monetary values: integer cents

### API Layer
- Base path: `/api/v1`
- Health check at `GET /health` (unauthenticated)
- Consistent JSON error envelope: `{ error: { code, message, details? } }`
- Centralized error middleware handles `AppError`, `ZodError`, and Prisma known errors
- Generic `validate()` middleware for Zod-backed request validation

### Logging
- Pino structured JSON logger
- Request/response logged by pino-http middleware
- Sensitive field redaction at logger level
- Development: pino-pretty with colorized output; production: raw JSON

### Planned Infrastructure (Webhook Feature)
- `webhook_outbox` table for transactional event queueing (Transactional Outbox pattern)
- `webhook_dead_letter` table for failed deliveries after 5 retry attempts
- `webhook_deliveries` table for delivery history visible to clients
- Worker process polling outbox every 2 seconds (separate Node.js process, `npm run worker`)
- HMAC-SHA256 signing with per-endpoint secrets

### Module Convention
All business modules follow the pattern: `{name}.controller.ts`, `{name}.service.ts`, `{name}.repository.ts`, `{name}.routes.ts`, `{name}.schemas.ts` under `src/modules/{name}/`.
