# Potential ADR: Prisma ORM as Data Access Layer

**Module**: DATA
**Category**: Technology
**Priority**: Must Document (Score: 145)
**Date Identified**: 2026-10-06

---

## What Was Identified

Prisma 5.22.0 was chosen as the exclusive data access layer for the entire application. Every module accesses MySQL through Prisma-generated client types — no raw SQL queries, no query builders (Knex, Kysely), and no alternative ORM (TypeORM, Sequelize, MikroORM) coexists in the dependency tree. The decision is present from the initial commit (2026-06-24) with no prior ORM or no-ORM evidence.

Prisma operates in three distinct roles within this project: schema definition language (PSL in `schema.prisma`), migration management tool (Prisma Migrate with `migration_lock.toml`), and generated type-safe client (`@prisma/client`). The `createPrismaClient()` factory in `src/config/database.ts` constructs a singleton instance for the API process, with a separate instance planned for the webhook worker process.

The ORDERS module uses Prisma interactive transactions (`prisma.$transaction(async (tx) => {...})`) for multi-step atomic operations — stock debit, order creation, and sequence number reservation. The planned webhook feature will invoke `publishWebhookEvent(tx, ...)` inside the same transaction boundary, relying on Prisma transaction propagation semantics. This makes Prisma's transaction API a structural coupling point across module boundaries.

## Why This Might Deserve an ADR

- **Impact**: All 6 business modules (AUTH, USERS, CUSTOMERS, PRODUCTS, ORDERS, WEBHOOKS) interact with the database exclusively through Prisma client. Repository classes in each module receive a `PrismaClient` instance via constructor injection.
- **Trade-offs**: Prisma's schema-first approach with generated client provides strong TypeScript type safety but introduces a code generation step in the development and CI workflow. Schema changes require `prisma generate` before TypeScript compilation. The PSL is not portable — switching ORMs means rewriting all repositories and regenerating the client.
- **Complexity**: Prisma Migrate's `shadowDatabaseUrl` requirement adds operational complexity: a separate "shadow" database must exist during migration development. This is a non-trivial CI/CD constraint.
- **Team Knowledge**: Every engineer touching data access must understand Prisma's filtering API, relation queries, transaction semantics, and the distinction between `$transaction` (interactive vs. batch). The `$transaction` pattern used in ORDERS is critical for correctness.
- **Future Implications**: The planned dual-process architecture (API + worker) requires two separate `PrismaClient` instances with independent connection pools — a pattern that must be explicitly documented to avoid connection exhaustion.

## Evidence Found in Codebase

### Key Files
- [`prisma/schema.prisma`](../../../../prisma/schema.prisma) - Full file
  - Schema-first model definitions in PSL, `generator client { provider = "prisma-client-js" }`
- [`src/config/database.ts`](../../../../src/config/database.ts) - Lines 1-10
  - `createPrismaClient()` factory and singleton export
- [`prisma/migrations/migration_lock.toml`](../../../../prisma/migrations/migration_lock.toml)
  - Prisma Migrate lock file confirming Migrate as the migration tool
- [`prisma/seed.ts`](../../../../prisma/seed.ts) - Lines 1-5, 165-175
  - PrismaClient usage in seed script including `$transaction` for sequence number reservation

### Code Evidence

```typescript
// src/config/database.ts:1-10
import { PrismaClient } from '@prisma/client';
import { env } from './env.js';

export function createPrismaClient(): PrismaClient {
  return new PrismaClient({
    log: env.NODE_ENV === 'development' ? ['warn', 'error'] : ['error'],
  });
}

export const prisma: PrismaClient = createPrismaClient();
```

```typescript
// prisma/seed.ts:165-175 — $transaction usage pattern
async function reserveOrderNumber(): Promise<string> {
  return prisma.$transaction(async (tx) => {
    const seq = await tx.orderNumberSequence.upsert({
      where: { id: 1 },
      create: { id: 1, nextValue: 2 },
      update: { nextValue: { increment: 1 } },
      select: { nextValue: true },
    });
    const current = seq.nextValue - 1;
    return `ORD-${String(current).padStart(6, '0')}`;
  });
}
```

```prisma
// prisma/schema.prisma:1-3
generator client {
  provider = "prisma-client-js"
}
```

### Impact Analysis
- Introduced: 2026-06-24 ("init repository")
- Modified: 1 commit — single-point introduction, no ORM evolution observed
- Affects: all 6 business modules via Repository pattern + INFRA module (database.ts singleton)
- Transaction pattern: used in ORDERS module, referenced in planned WEBHOOKS module
- Recent themes: Initial setup — no Prisma version upgrade commits observed

### Alternatives (if observable)

No alternatives are documented for the ORM choice. The selection of Prisma over TypeORM, MikroORM, Sequelize, or raw query builders (Knex, Kysely) is not recorded in context documents. This gap is itself a reason to create an ADR — the rationale is currently implicit.

## Questions to Address in ADR (if created)

- Why Prisma over TypeORM or MikroORM, which are more established in the Node.js/TypeScript ecosystem?
- How is `prisma generate` integrated into the CI pipeline and local development workflow?
- What is the connection pool sizing strategy for the API process and the planned worker process?
- How will `shadowDatabaseUrl` be managed in CI (separate schema on same server, separate Docker container)?
- Is Prisma Migrate the long-term migration strategy, or will raw SQL migrations be introduced for complex operations?

## Related Potential ADRs
- [MySQL 8.0 as Primary Database](./mysql-8-as-primary-database.md)

## Additional Notes

The `shadowDatabaseUrl` in `schema.prisma` is evidence that Prisma Migrate (not `prisma db push`) is the chosen migration workflow. This distinction matters for production deployments and deserves explicit documentation alongside the ORM choice itself.
