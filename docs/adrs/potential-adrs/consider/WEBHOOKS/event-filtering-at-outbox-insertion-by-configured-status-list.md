# Potential ADR: Event Filtering at Outbox Insertion by Configured Status List

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Consider (Score: 80/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

When the `publishWebhookEvent` function is called inside `OrderService.changeStatus`, it does not unconditionally insert a row into `webhook_outbox`. Instead, it first checks each registered webhook for the affected customer and only inserts outbox rows for endpoints whose configured `event_filter` list includes the `toStatus` value. If no webhook for that customer has subscribed to the new status, no outbox row is created at all.

This is a pre-insertion filter — it operates within the transaction boundary, before any rows are written. The alternative (inserting for all events and filtering at dispatch time) was discussed and rejected because inserting unnecessary rows wastes storage and pollutes the outbox table.

## Why This Might Deserve an ADR

- **Impact**: Determines the relationship between the webhook configuration table and the outbox table. Engineers adding new order statuses to the FSM must be aware that status values used in `event_filter` lists are coupled to the `OrderStatus` enum. Removing or renaming a status value could silently break existing filter configurations.
- **Trade-offs**: Filter-at-insertion reduces outbox volume and simplifies the worker (no need to apply business rules at dispatch time). However, it requires a DB read inside the transaction (`SELECT webhooks WHERE customer_id = ? AND event_filter CONTAINS ?`), adding latency to every `changeStatus` call.
- **Complexity**: The filter logic must correctly handle JSON array storage of status values in the `webhooks` table and perform the lookup efficiently within the existing transaction.
- **Team Knowledge**: Developers adding new `OrderStatus` values must update documentation for the webhook event filter to list the new status as a subscribable option. This coupling is not visible from the order module alone.
- **Future Implications**: If the order status FSM expands (new statuses), the `event_filter` vocabulary must expand in sync. There is no schema enforcement linking FSM status values to filter configuration.
- **Temporal Context**: Decided at project inception (2026-06-24) — decisions.md #11.

## Evidence Found in Codebase

### Key Files
- [`decisions.md`](../../../../../decisions.md) - Decision #11 (filter at insertion, not dispatch)
- [`TRANSCRICAO.md`](../../../../../TRANSCRICAO.md) - Lines 193–200 (Bruno and Diego discussing filter placement)
- [`src/modules/orders/order.status.ts`](../../../../../src/modules/orders/order.status.ts)
  - Defines `OrderStatus` enum values — these are the vocabulary for `event_filter` configuration

### Code Evidence
```typescript
// src/modules/orders/order.status.ts
// OrderStatus values: PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED
// These are the subscribable events in the webhook event_filter

// Planned publishWebhookEvent behavior:
// 1. Query: SELECT * FROM webhooks WHERE customer_id = order.customerId AND active = true
// 2. For each webhook where toStatus IN webhook.event_filter:
//    INSERT INTO webhook_outbox (id, webhook_id, event_id, payload, status, created_at)
// 3. If no matching webhooks: no insert, no-op
```

```typescript
// Rejected approach (filter at dispatch time):
// INSERT INTO webhook_outbox for ALL active webhooks for customer
// Worker reads event, checks filter, skips non-matching
// Downside: unnecessary rows, worker still does business logic
```

### Impact Analysis
- Introduced: 2026-06-24 (repository init, via meeting records)
- Modified: N/A — not yet implemented
- Affects: WEBHOOKS (publishWebhookEvent implementation, webhooks table schema — event_filter column), ORDERS (changeStatus transaction adds a SELECT query)
- Coupling: OrderStatus enum values are now referenced in webhook configuration data; renaming a status requires a data migration

### Alternatives

| Alternative | Rejected Reason |
|---|---|
| Filter at dispatch in the worker | Unnecessary outbox rows; business filtering logic in worker is less clean |
| No filtering (deliver all events to all webhooks) | Customers receive noise for events they did not subscribe to |

## Questions to Address in ADR (if created)

- What is the column type for `event_filter` in the `webhooks` table (JSON array, comma-separated varchar, relation table)?
- How is the filter validated on webhook registration — against the current `OrderStatus` enum values?
- What happens when an order status is deprecated — are existing `event_filter` configs migrated?

## Related Potential ADRs
- [Transactional Outbox Pattern for Webhook Event Dispatch](../must-document/WEBHOOKS/transactional-outbox-pattern-for-webhook-event-dispatch.md)
- [Snapshot Payload on Outbox Insertion](./snapshot-payload-on-outbox-insertion.md)

## Additional Notes
This decision creates a tight coupling between the order domain's FSM vocabulary and the webhook configuration domain. This coupling is worth capturing so future engineers understand why adding a new `OrderStatus` value requires updating webhook documentation and potentially customer-facing API schemas.
