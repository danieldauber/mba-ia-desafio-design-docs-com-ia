# Potential ADR: MySQL 8.0 via Docker Compose as Primary Database

**Module**: INFRA
**Category**: Technology
**Priority**: Must Document (Score: 150/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

MySQL 8.0 is the sole persistence engine for all application data. The service is defined in `docker-compose.yml` and was present from the initial commit on 2026-06-24. It is the only infrastructure service in the compose file, with no secondary databases, caches, or message brokers present.

The compose service is configured with `utf8mb4` character set and `utf8mb4_unicode_ci` collation, `mysql_native_password` authentication plugin, a named persistent volume, and a health check using `mysqladmin ping` with 20 retries and a 30-second start period. The Prisma datasource in `prisma/schema.prisma` targets this MySQL instance exclusively via the `DATABASE_URL` environment variable.

The `decisions.md` context file explicitly records that MySQL was chosen over Redis Streams and external message brokers (Kafka, RabbitMQ) for the Transactional Outbox pattern, citing team size and the desire to avoid additional infrastructure dependencies. This positions the MySQL instance not only as the primary data store but also as the planned event queue backbone for the webhook feature.

## Why This Might Deserve an ADR

- **Impact**: Every read and write operation in the system flows through this MySQL instance. The schema, migration strategy, and all Prisma model definitions are MySQL-specific.
- **Trade-offs**: MySQL 8.0 was chosen over PostgreSQL (which Prisma also supports natively). Specific MySQL behaviors — JSON column support for `Customer.address`, `utf8mb4` for full Unicode support, `mysql_native_password` for compatibility — are structural choices that affect schema design.
- **Complexity**: The database serves a dual role: primary application store AND planned outbox queue for webhook delivery. This dual-purpose usage is architectural and deserves documentation.
- **Team Knowledge**: All engineers adding features must know the database type to correctly write migrations, understand index behavior (e.g., MySQL's handling of UUID primary keys vs. sequential integers), and work with Prisma's MySQL adapter.
- **Future Implications**: The planned `webhook_outbox` and `webhook_dead_letter` tables will extend this same MySQL instance, reinforcing the decision to use MySQL as the event queue rather than introducing a broker. Scaling this to multi-worker scenarios (noted as deferred in `decisions.md`) will require MySQL-specific solutions (row locking, `SKIP LOCKED`).

## Evidence Found in Codebase

### Key Files
- [`docker-compose.yml`](../../../../docker-compose.yml) - Lines 1-28 — full MySQL 8.0 service definition with healthcheck and volume
- [`src/config/database.ts`](../../../../src/config/database.ts) - Lines 1-10 — PrismaClient singleton targeting MySQL
- [`prisma/schema.prisma`](../../../../prisma/schema.prisma) — datasource provider = "mysql"

### Code Evidence
```yaml
# docker-compose.yml:1-27
services:
  mysql:
    image: mysql:8.0
    container_name: oms-mysql
    restart: unless-stopped
    environment:
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-root_password}
      MYSQL_DATABASE: ${MYSQL_DATABASE:-oms}
      MYSQL_USER: ${MYSQL_USER:-oms_user}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-oms_password}
    ports:
      - '3306:3306'
    volumes:
      - oms_mysql_data:/var/lib/mysql
    healthcheck:
      test: ['CMD', 'mysqladmin', 'ping', '-h', 'localhost', ...]
      interval: 5s
      timeout: 5s
      retries: 20
      start_period: 30s
    command:
      - --default-authentication-plugin=mysql_native_password
      - --character-set-server=utf8mb4
      - --collation-server=utf8mb4_unicode_ci
```

```typescript
// src/config/database.ts:4-10
export function createPrismaClient(): PrismaClient {
  return new PrismaClient({
    log: env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error'],
  });
}

export const prisma: PrismaClient = createPrismaClient();
```

### Impact Analysis
- Introduced: 2026-06-24 (initial commit)
- Modified: 1 commit (stable since introduction)
- Affects: all DATA module models, all repository implementations, migration strategy, planned webhook outbox tables
- The `decisions.md` context file explicitly rejects Redis Streams and external brokers in favour of MySQL as the outbox store

### Alternatives (if observable)
The `decisions.md` context file explicitly documents rejected alternatives:
- **Redis Streams** — rejected as overengineering for the team size
- **Kafka / RabbitMQ** — rejected as overengineering for the team size
- **PostgreSQL** — not mentioned; MySQL chosen without documented comparison

## Questions to Address in ADR (if created)

- Why MySQL 8.0 over PostgreSQL given Prisma supports both equally?
- What is the strategy for production deployment (managed RDS vs. self-hosted Docker)?
- Does the dual-purpose role of MySQL (data store + outbox queue) hold as the webhook feature scales?
- How will `SKIP LOCKED` or equivalent row-locking be used when multiple webhook workers poll the outbox?
- Is the `mysql_native_password` plugin choice a temporary compatibility measure or a long-term constraint?

## Related Potential ADRs
- [`express-as-primary-web-framework.md`](./express-as-primary-web-framework.md)

## Additional Notes
The compose file does not define a test database service. Integration tests (TEST module) use the same MySQL instance, controlled by a `DATABASE_URL` environment variable switch. This means no test isolation at the infrastructure level — worth noting in the ADR context.
