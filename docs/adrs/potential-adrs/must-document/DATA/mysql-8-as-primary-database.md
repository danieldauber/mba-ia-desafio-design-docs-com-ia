# Potential ADR: MySQL 8.0 as Primary Relational Database

**Module**: DATA
**Category**: Technology
**Priority**: Must Document (Score: 150)
**Date Identified**: 2026-10-06

---

## What Was Identified

MySQL 8.0 was selected as the sole relational database engine for all persistence needs of the order-management-api. The decision is embedded from the very first commit (2026-06-24, "init repository") — no migration history exists, indicating the choice was made before the repository was initialized, not discovered through iteration.

The datasource block in `prisma/schema.prisma` explicitly declares `provider = "mysql"`, and the `migration_lock.toml` locks the migration engine to `provider = "mysql"`, making any future change to a different database require full migration regeneration. The Docker Compose service provisions MySQL 8.0 with utf8mb4 character set, a persistent named volume, and a healthcheck, confirming operational intent beyond development convenience.

Context documents (`decisions.md`) record that Redis Streams and external brokers (Kafka, RabbitMQ) were explicitly rejected for the planned webhook outbox feature in favor of a MySQL-backed outbox table (`webhook_outbox`). This confirms MySQL was evaluated not only as a storage engine but as a messaging substrate — a significantly broader role than typical RDBMS usage.

## Why This Might Deserve an ADR

- **Impact**: Every module (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS, planned WEBHOOKS) depends on this database. All schema definitions, query patterns, transaction semantics, and operational runbooks are MySQL-specific.
- **Trade-offs**: MySQL 8.0 with Prisma ORM locks the team into MySQL-compatible SQL dialect, utf8mb4 encoding behavior, and InnoDB storage engine semantics. JSON column support (used for `Customer.address`) and the transactional outbox pattern both depend on InnoDB transaction guarantees.
- **Complexity**: The choice to use MySQL as the outbox backing store (instead of Redis or a broker) means the database must sustain both transactional OLTP load and worker polling load — an operational trade-off requiring documented rationale.
- **Team Knowledge**: Any engineer working on schema changes, migrations, index tuning, or the webhook worker must understand MySQL-specific behavior (e.g., ENUM types, CHAR(36) UUID storage, datetime(3) precision, cascading FK constraints).
- **Future Implications**: Scaling decisions (read replicas, connection pooling, partitioning for outbox tables), backup/restore strategy, and cloud vendor selection (RDS MySQL vs. Aurora MySQL vs. PlanetScale) all flow from this choice.

## Evidence Found in Codebase

### Key Files
- [`prisma/schema.prisma`](../../../../prisma/schema.prisma) - Lines 5-9
  - Datasource declaration with `provider = "mysql"` and dual URL env vars (including `shadowDatabaseUrl` for Prisma Migrate)
- [`prisma/migrations/migration_lock.toml`](../../../../prisma/migrations/migration_lock.toml) - Line 3
  - Migration engine locked to MySQL provider
- [`prisma/migrations/20260519182739_init/migration.sql`](../../../../prisma/migrations/20260519182739_init/migration.sql) - Lines 1-126
  - Full DDL using MySQL syntax: `CHAR(36)`, `ENUM(...)`, `DATETIME(3)`, `JSON`, `utf8mb4_unicode_ci`
- [`docker-compose.yml`](../../../../docker-compose.yml) - MySQL 8.0 service definition

### Code Evidence

```prisma
// prisma/schema.prisma:5-9
datasource db {
  provider          = "mysql"
  url               = env("DATABASE_URL")
  shadowDatabaseUrl = env("SHADOW_DATABASE_URL")
}
```

```sql
-- prisma/migrations/20260519182739_init/migration.sql:2-13
CREATE TABLE `users` (
    `id` CHAR(36) NOT NULL,
    `email` VARCHAR(255) NOT NULL,
    `passwordHash` VARCHAR(255) NOT NULL,
    `name` VARCHAR(150) NOT NULL,
    `role` ENUM('ADMIN', 'OPERATOR') NOT NULL DEFAULT 'OPERATOR',
    `createdAt` DATETIME(3) NOT NULL DEFAULT CURRENT_TIMESTAMP(3),
    `updatedAt` DATETIME(3) NOT NULL,
    UNIQUE INDEX `users_email_key`(`email`),
    PRIMARY KEY (`id`)
) DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
```

### Impact Analysis
- Introduced: 2026-06-24 ("init repository")
- Modified: 1 commit — stable since inception, no schema evolution commits observed
- Affects: all 7 current tables across 6 modules
- Planned extension: `webhook_outbox`, `webhook_dead_letter`, `webhook_deliveries` tables
- Recent themes: Initial setup — no operational tuning commits yet

### Alternatives (if observable)

Context documentation (`decisions.md`) explicitly records rejected alternatives for the webhook outbox backing store:
- Redis Streams — rejected as overengineering for team size
- Kafka / RabbitMQ — rejected as overengineering for team size
- MySQL trigger-based reactive notification — rejected

No alternative RDBMS (PostgreSQL, SQLite) was mentioned; MySQL appears to have been a pre-existing constraint or implicit team preference.

## Questions to Address in ADR (if created)

- Was MySQL selected over PostgreSQL, and if so, why (team familiarity, hosting constraints, existing infrastructure)?
- Is there a cloud hosting target (AWS RDS, GCP Cloud SQL, PlanetScale) that influenced this choice?
- How will the `SHADOW_DATABASE_URL` requirement for Prisma Migrate be managed in CI/CD and production?
- What is the connection pooling strategy for the planned dual-process architecture (API process + worker process)?
- What backup, point-in-time recovery, and failover strategy is planned?

## Related Potential ADRs
- [Prisma ORM as Data Access Layer](./prisma-orm-as-data-access-layer.md)

## Additional Notes

The presence of `shadowDatabaseUrl` in the datasource indicates Prisma Migrate is the migration management tool, not a raw SQL workflow. This is an implicit sub-decision within the MySQL choice that is worth documenting.
