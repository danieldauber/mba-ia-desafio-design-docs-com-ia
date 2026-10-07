# Potential ADR: Snapshot Payload on Outbox Insertion

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Consider (Score: 90/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

When inserting a row into `webhook_outbox`, the full event payload is rendered and stored as a JSON snapshot at the moment of insertion — not computed lazily at dispatch time. This means the outbox row contains the complete order state (order_id, order_number, from_status, to_status, customer_id, total_cents) as it existed when the status change occurred.

The alternative — storing only `order_id` and rendering the payload when the worker dispatches — was evaluated and rejected. If the order is modified after the status change but before the worker dispatches (even within seconds), a lazily-rendered payload would reflect the later state, not the state at the time of the event.

This decision was made by Larissa (Tech Lead) and Diego in the post-call discussion captured in `TRANSCRICAO.md` (09:51–09:52), and is recorded as decision #12 and #36 in `decisions.md`.

## Why This Might Deserve an ADR

- **Impact**: Affects data storage volume (each outbox row carries the full payload), data consistency guarantees (snapshot vs. live query), and the worker implementation (no additional DB reads needed at dispatch time for payload construction).
- **Trade-offs**: Snapshots consume more storage per outbox row but provide strong consistency. Lazy rendering is more storage-efficient but risks stale data if orders are modified concurrently or between insertion and dispatch.
- **Complexity**: The `publishWebhookEvent(tx, order, fromStatus, toStatus)` function must receive the complete order object (not just the ID) to render the snapshot at insertion time. The snapshot format must be versioned or at least stable.
- **Team Knowledge**: Developers who later add fields to the webhook payload must ensure those fields are captured at insertion time, not at dispatch. This is a non-obvious constraint if not documented.
- **Future Implications**: If payload schema evolves, old snapshot rows may have a different structure than new ones. The worker must handle both, or a migration must update old rows.
- **Temporal Context**: This was a last-minute confirmation post-call (09:51–09:52 in TRANSCRICAO.md), decided after the main meeting had ended, indicating it was nearly overlooked.

## Evidence Found in Codebase

### Key Files
- [`decisions.md`](../../../../../decisions.md) - Decision #12 (snapshot on insertion), #36 (lazy render rejected)
- [`TRANSCRICAO.md`](../../../../../TRANSCRICAO.md) - Lines 307–314 (post-call conversation between Bruno, Diego, and Larissa)
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) - Lines 168–178
  - The `refreshed` order object at the end of `changeStatus` is the natural candidate to pass to `publishWebhookEvent`

### Code Evidence
```typescript
// src/modules/orders/order.service.ts:131–178
// After changeStatus commits, the refreshed order is returned.
// The publishWebhookEvent call will use the intermediate `order` (state at transition start)
// or the refreshed state — the decision is to snapshot at the transition moment.

// Planned outbox row payload field:
// webhook_outbox.payload: JSON — rendered at INSERT time inside publishWebhookEvent(tx, order, from, to)
// Contains: { event_id, event_type, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents }
```

```typescript
// Rejected approach (decisions.md #36):
// webhook_outbox.order_id: Char(36) — only store reference
// Worker at dispatch: const order = await prisma.order.findUnique({ where: { id: outbox.order_id } })
// Risk: order may have changed between insertion and dispatch
```

### Impact Analysis
- Introduced: 2026-06-24 (post-call clarification, same commit date)
- Modified: N/A — not yet implemented
- Affects: WEBHOOKS (publishWebhookEvent signature, outbox schema), ORDERS (must pass order state to publishWebhookEvent)
- Storage implication: each outbox row stores a JSON blob (~500–1000 bytes per event) instead of just a UUID reference

### Alternatives

| Alternative | Rejected Reason |
|---|---|
| Store only order_id, render payload at dispatch | Order may change between insertion and dispatch; snapshot inconsistency risk |

## Questions to Address in ADR (if created)

- What is the exact set of fields in the snapshot (does it include all payload fields from decisions.md #21)?
- What order state is captured — the state at the start of `changeStatus` or after the update is applied?
- Is the snapshot JSON schema versioned (e.g., `payload_version` field) to handle future schema evolution?

## Related Potential ADRs
- [Transactional Outbox Pattern for Webhook Event Dispatch](../must-document/WEBHOOKS/transactional-outbox-pattern-for-webhook-event-dispatch.md)
- [At-Least-Once Delivery with X-Event-Id Idempotency](../must-document/WEBHOOKS/at-least-once-delivery-with-event-id-idempotency.md)

## Additional Notes
This decision was nearly omitted — it was raised by Bruno after the main meeting had ended. The risk of the lazy-render approach would have been subtle and hard to detect in testing, since the window for the race condition is narrow.
