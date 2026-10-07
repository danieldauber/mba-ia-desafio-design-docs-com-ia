# Potential ADR: Transactional Outbox Pattern for Webhook Event Dispatch

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 145/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

The team explicitly chose the Transactional Outbox pattern as the mechanism for dispatching webhook notifications when order statuses change. Rather than calling customer endpoints synchronously inside `OrderService.changeStatus`, the design inserts a row into a `webhook_outbox` table within the same Prisma `$transaction` that updates the order status and stock. A separate worker process then polls the outbox and performs the outbound HTTP calls.

This decision was the central architectural choice in the technical meeting documented in `TRANSCRICAO.md` (recorded 2026-06-24 in the initial repository commit). The team explicitly compared this approach against synchronous dispatch and against external message brokers (Redis Streams, Kafka, RabbitMQ) before committing to the outbox. The pattern ties the lifecycle of the webhook event to the database transaction: if `changeStatus` rolls back, no orphan event exists; if it commits, the event is guaranteed present in the outbox.

The integration point is codified as `publishWebhookEvent(tx, order, fromStatus, toStatus)` — a function that receives the active `Prisma.TransactionClient` so it participates in the parent transaction without injecting a full repository into `OrderService`.

## Why This Might Deserve an ADR

- **Impact**: This is the foundational delivery guarantee for the entire webhook feature. Every downstream decision — retry strategy, DLQ, worker process isolation, ordering semantics — depends on the outbox being the single source of truth for pending events.
- **Trade-offs**: The outbox couples event dispatch atomicity to MySQL. If the team ever moves to a different primary database or introduces a message broker, this decision must be revisited. The alternative (external broker) was explicitly discarded as overengineering for the current team size.
- **Complexity**: The pattern requires a `webhook_outbox` table, a polling worker, and a specific function signature (`publishWebhookEvent(tx, ...)`) injected into `OrderService.changeStatus`. Anyone modifying order status transitions must be aware of this contract.
- **Team Knowledge**: Every engineer touching the ORDERS or WEBHOOKS module must understand why the outbox insert is inside the same `$transaction` block. Removing or moving it would silently break atomicity.
- **Future Implications**: Ordering guarantees, multi-worker scaling, and archival policy are all direct consequences of this pattern choice. Future rate limiting or batching decisions will also build on it.
- **Temporal Context**: Decided at project inception (2026-06-24); no implementation exists yet — this is a pre-implementation architectural commitment documented through meeting records.

## Evidence Found in Codebase

### Key Files
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) - Lines 126–178
  - `changeStatus` method uses `this.prisma.$transaction(async (tx) => { ... })`. The planned `publishWebhookEvent(tx, order, fromStatus, toStatus)` call will be inserted inside this same lambda, after the status update and history insert.
- [`decisions.md`](../../../../../decisions.md) - Decision #1, #15, #30, #31, #32, #36
  - Decision #1 records the outbox choice with justification. Decision #15 specifies the integration point. Decisions #30–32, #36 document what was explicitly rejected.
- [`TRANSCRICAO.md`](../../../../../TRANSCRICAO.md) - Lines 44–58, 238–242
  - Diego's explanation of the pattern (09:06–09:08) and Bruno's description of the integration into `changeStatus` (09:40–09:41).

### Code Evidence
```typescript
// src/modules/orders/order.service.ts:131
async changeStatus(id: string, input: UpdateOrderStatusInput, userId: string): Promise<OrderWithRelations> {
  return this.prisma.$transaction(async (tx) => {
    // ... order lookup, FSM validation, stock debit/replenish ...
    await tx.order.update({ where: { id }, data: { status: to } });
    await tx.orderStatusHistory.create({ data: { ... } });
    // PLANNED: await publishWebhookEvent(tx, order, from, to);
    const refreshed = await tx.order.findUnique({ ... });
    return refreshed!;
  });
}
```

```typescript
// Planned function signature (from decisions.md #15 + TRANSCRICAO.md):
// publishWebhookEvent(tx: Prisma.TransactionClient, order: Order, fromStatus: OrderStatus, toStatus: OrderStatus): Promise<void>
// Inserts into webhook_outbox within the active transaction
```

### Impact Analysis
- Introduced: 2026-06-24 (repository init, via meeting records)
- Modified: N/A — not yet implemented in code
- Affects: ORDERS module (`order.service.ts`), WEBHOOKS module (outbox table, worker), DATA layer (new Prisma models), INFRA (new worker entry point `src/worker.ts`)
- Recent themes from decisions: "atomicity", "consistency", "rollback safety", "team size", "overengineering rejection"

### Alternatives
The following were explicitly evaluated and discarded:

| Alternative | Rejected Reason |
|---|---|
| Synchronous HTTP call in `OrderService.changeStatus` | Client offline would cause transaction rollback on order status change |
| Redis Streams | Overengineering for small team; adds new infrastructure dependency |
| External message broker (Kafka/RabbitMQ) | Same as Redis Streams |
| MySQL trigger-based notification | MySQL has no LISTEN/NOTIFY; workaround would be an antipattern |

## Questions to Address in ADR (if created)

- What delivery guarantee does the outbox provide vs. a message broker?
- How does `publishWebhookEvent` receive the transaction client without coupling `OrderService` to `WebhookRepository`?
- What happens to outbox rows after successful delivery (archival policy, retention period)?
- What is the ordering guarantee across concurrent order status changes?

## Related Potential ADRs
- [Separate Worker Process with Polling for Outbox Consumption](./separate-worker-process-with-polling-for-outbox-consumption.md)
- [Exponential Backoff Retry with Dead Letter Queue](./exponential-backoff-retry-with-dead-letter-queue.md)
- [At-Least-Once Delivery with X-Event-Id Idempotency](./at-least-once-delivery-with-event-id-idempotency.md)
- [HMAC-SHA256 Per-Endpoint Payload Signing with Secret Rotation](./hmac-sha256-per-endpoint-payload-signing-with-secret-rotation.md)

## Additional Notes
The module does not exist in code yet. Evidence is sourced entirely from `decisions.md` (structured decision log) and `TRANSCRICAO.md` (verbatim meeting transcript), both committed on 2026-06-24. Git history confirms this is the original design intent.
