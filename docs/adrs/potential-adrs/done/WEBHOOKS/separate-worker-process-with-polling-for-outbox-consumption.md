# Potential ADR: Separate Worker Process with Polling for Outbox Consumption

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 130/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

The team decided the webhook delivery worker will run as a completely separate Node.js process (`src/worker.ts`, started via `npm run worker`), independent of the API process (`src/server.ts`). This worker polls the `webhook_outbox` table every 2 seconds for pending events, processes them (HTTP dispatch, retry logic), and writes results back to the database.

This was a deliberate decision against embedding the worker in the same process as the API. The team also evaluated reactive alternatives (MySQL triggers, LISTEN/NOTIFY-style notification) and rejected them because MySQL lacks a native notification mechanism.

The 2-second polling interval was chosen to satisfy the business requirement of delivering notifications in under 10 seconds (stated by clients Atlas Comercial, MaxDistribuição, and Nova Cargo). The worker uses its own `PrismaClient` instance because `PrismaClient` is scoped per Node.js process.

## Why This Might Deserve an ADR

- **Impact**: Determines the operational topology of the system. The team must deploy and supervise two Node.js processes instead of one. Process restart behavior, health monitoring, and scaling differ for each. Anyone writing a Dockerfile, Kubernetes manifest, or CI/CD pipeline must account for this second process.
- **Trade-offs**: Polling introduces a minimum 0–2s latency per event (the worker may have just started a sleep cycle). This is accepted as satisfying the sub-10s requirement. A reactive approach would reduce latency but requires infrastructure MySQL cannot provide natively without workarounds.
- **Complexity**: The worker needs its own `PrismaClient`, its own logger configuration, and its own process lifecycle (graceful shutdown). It must share `DATABASE_URL` but be restarted independently from the API.
- **Team Knowledge**: Developers unfamiliar with the separate process will attempt to add webhook logic to the API process or be confused by the two entry points. This decision must be documented for onboarding.
- **Future Implications**: Any attempt to scale webhook throughput requires partitioning the outbox by `order_id` and running multiple workers — a problem explicitly deferred. The single-worker constraint is a known limitation of this design.
- **Temporal Context**: Decided at project inception (2026-06-24); the `src/worker.ts` file does not yet exist in the codebase.

## Evidence Found in Codebase

### Key Files
- [`src/server.ts`](../../../../../src/server.ts) - Existing API process entry point; the worker will mirror this structure
- [`src/config/database.ts`](../../../../../src/config/database.ts) - `createPrismaClient()` factory; worker will call this independently to instantiate its own client
- [`decisions.md`](../../../../../decisions.md) - Decision #2 (separate process), #3 (2s polling), #16 (separate PrismaClient), #32 (MySQL trigger rejected), #24 (single-worker ordering limitation)
- [`TRANSCRICAO.md`](../../../../../TRANSCRICAO.md) - Lines 70–73, 176–179

### Code Evidence
```typescript
// src/server.ts — existing entry point pattern the worker will follow
import { buildApp } from './app.js';
import { createPrismaClient } from './config/database.js';
import { env } from './config/env.js';

// Planned src/worker.ts will follow same structure:
// import { createPrismaClient } from './config/database.js';
// const prisma = createPrismaClient(); // separate instance, same DATABASE_URL
// const processor = new WebhookProcessor(prisma, logger);
// processor.start(); // polling loop every 2s
```

```typescript
// Planned polling loop (from decisions.md #3):
// setInterval or while loop — poll webhook_outbox WHERE status = 'PENDING'
// ORDER BY createdAt ASC — preserves ordering within single-worker constraint
// Process batch, mark as DELIVERED or increment retry_count
```

### Impact Analysis
- Introduced: 2026-06-24 (repository init, via meeting records)
- Modified: N/A — not yet implemented
- Affects: INFRA (new entry point, process management), WEBHOOKS (worker logic), DATA (outbox polling queries), deployment configuration
- Single-worker ordering constraint: documented in decisions.md #24 — ordering guaranteed only per `order_id` while single-worker; explicitly deferred multi-worker partitioning

### Alternatives

| Alternative | Rejected Reason |
|---|---|
| Worker embedded in API process | API restart kills the worker; lifecycle coupling is undesirable |
| MySQL trigger for reactive notification | MySQL has no LISTEN/NOTIFY native mechanism; any workaround (file, endpoint call) is an antipattern |
| Longer polling interval (>2s) | Would not meet the sub-10s delivery requirement |

## Questions to Address in ADR (if created)

- How is the worker process supervised in production (systemd, Docker restart policy, PM2)?
- What is the graceful shutdown sequence for the worker (drain in-flight events)?
- How is the worker's health monitored and alerting configured?
- What triggers a decision to move to multi-worker with partitioned polling?

## Related Potential ADRs
- [Transactional Outbox Pattern for Webhook Event Dispatch](./transactional-outbox-pattern-for-webhook-event-dispatch.md)
- [Exponential Backoff Retry with Dead Letter Queue](./exponential-backoff-retry-with-dead-letter-queue.md)

## Additional Notes
The separate process model introduces a new operational concern for a project that currently ships as a single API process. This is the most significant infrastructure topology change introduced by the webhook feature.
