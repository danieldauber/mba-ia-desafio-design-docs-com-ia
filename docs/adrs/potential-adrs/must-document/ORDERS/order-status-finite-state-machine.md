# Potential ADR: Order Status Finite State Machine

**Module**: ORDERS
**Category**: Architecture
**Priority**: Must Document (Score: 135/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

The order lifecycle in this system is governed by an explicit Finite State Machine (FSM) implemented in `src/modules/orders/order.status.ts`. The FSM defines a declarative, immutable transition table mapping each `OrderStatus` value to the set of statuses it may legally transition into. This table is the single source of truth for all status changes across the system.

The pattern was established in the initial repository commit on 2026-06-24 and has not been modified since — indicating a stable, intentional design that was settled before development began. All order mutation operations (`create`, `changeStatus`, `delete`) consult the FSM, and the planned WEBHOOKS module is architecturally coupled to FSM transitions as its event source.

Beyond pure transition enforcement, the FSM module exports named predicate functions (`shouldDebitStock`, `shouldReplenishStock`) that determine when inventory side effects must fire. This means the FSM is not just a validation layer — it is the authoritative coordinator of cross-module side effects triggered by status changes.

## Why This Might Deserve an ADR

- **Impact**: Every order mutation in the system passes through the FSM. The PRODUCTS module stock levels change based on FSM predicates. The planned WEBHOOKS module will insert into the outbox based on FSM transition events. The `InvalidStatusTransitionError` in SHARED is a direct FSM contract artifact.
- **Trade-offs**: An explicit static transition table makes valid paths immediately auditable and prevents ad-hoc status assignments scattered across service code. The trade-off is rigidity — adding a new status (e.g., `PARTIALLY_SHIPPED`) requires updating the transition table, the FSM predicates, the Prisma enum, migrations, and all consumers.
- **Complexity**: The FSM bridges three concerns: transition validation, terminal state detection, and stock side-effect coordination. An engineer unfamiliar with this pattern could easily add a direct `order.update({ status: newStatus })` call that bypasses the FSM entirely, introducing silent inconsistencies.
- **Team Knowledge**: Any engineer touching orders, products stock, or the webhook integration must understand which transitions are valid, which are terminal, and which transitions trigger stock mutations. Without documented intent, the `STOCK_DEBIT_TRANSITION` constant and the `shouldReplenishStock` conditions read as magic.
- **Future Implications**: The planned WEBHOOKS feature passes `fromStatus` and `toStatus` into `publishWebhookEvent(tx, order, fromStatus, toStatus)` — meaning external customers will receive event payloads that reflect FSM transitions. If the FSM is modified, the event contract changes. Undocumented, this creates a future breaking-change risk.
- **Temporal Context**: Stable for 4 months with zero modifications. The design was fixed before any feature work commenced.

## Evidence Found in Codebase

### Key Files

- [`src/modules/orders/order.status.ts`](../../../../../src/modules/orders/order.status.ts) - Lines 1-37
  - Contains the complete FSM: transition table, `canTransition`, `allowedTransitions`, `isTerminal`, `STOCK_DEBIT_TRANSITION`, `shouldDebitStock`, `shouldReplenishStock`
- [`src/modules/orders/order.service.ts`](../../../../../src/modules/orders/order.service.ts) - Lines 147-157
  - Consumption of `canTransition`, `shouldDebitStock`, `shouldReplenishStock` inside the `changeStatus` transaction
- [`src/shared/errors/index.js`](../../../../../src/shared/errors/index.js) - `InvalidStatusTransitionError`
  - A dedicated error type exists specifically to represent FSM violation — evidence that the FSM is a first-class architectural contract

### Code Evidence

```typescript
// src/modules/orders/order.status.ts:3-10 — FSM transition table
const transitions: Readonly<Record<OrderStatus, ReadonlyArray<OrderStatus>>> = {
  [OrderStatus.PENDING]: [OrderStatus.PAID, OrderStatus.CANCELLED],
  [OrderStatus.PAID]: [OrderStatus.PROCESSING, OrderStatus.CANCELLED],
  [OrderStatus.PROCESSING]: [OrderStatus.SHIPPED, OrderStatus.CANCELLED],
  [OrderStatus.SHIPPED]: [OrderStatus.DELIVERED],
  [OrderStatus.DELIVERED]: [],
  [OrderStatus.CANCELLED]: [],
};
```

```typescript
// src/modules/orders/order.status.ts:24-37 — Stock side-effect predicates
export const STOCK_DEBIT_TRANSITION = {
  from: OrderStatus.PENDING,
  to: OrderStatus.PAID,
} as const;

export function shouldDebitStock(from: OrderStatus, to: OrderStatus): boolean {
  return from === STOCK_DEBIT_TRANSITION.from && to === STOCK_DEBIT_TRANSITION.to;
}

export function shouldReplenishStock(from: OrderStatus, to: OrderStatus): boolean {
  return (
    to === OrderStatus.CANCELLED && (from === OrderStatus.PAID || from === OrderStatus.PROCESSING)
  );
}
```

```typescript
// src/modules/orders/order.service.ts:147-157 — FSM enforcement in changeStatus
if (!canTransition(from, to)) {
  throw new InvalidStatusTransitionError(from, to);
}

if (shouldDebitStock(from, to)) {
  await this.debitStock(tx, order.items);
}
if (shouldReplenishStock(from, to)) {
  await this.replenishStock(tx, order.items);
}
```

### Impact Analysis

- Introduced: 2026-06-24 ("init repository")
- Modified: 0 commits after introduction — no evolution history
- Last change: 2026-06-24
- Affects: 3 files directly (order.status.ts, order.service.ts, shared errors), 4 modules by dependency (ORDERS, PRODUCTS, SHARED, planned WEBHOOKS)
- Recent themes: No modifications — design appears settled from inception

### Alternatives (if observable)

No explicit alternatives are recorded in code comments. However, the context file `decisions.md` documents that synchronous webhook delivery inside the order service was considered and rejected — which implies that the FSM-driven event coordination approach (triggering side effects from transition predicates) was a deliberate choice over ad-hoc inline coupling. The separation of transition logic into a standalone module (rather than inline conditionals in OrderService) is itself an architectural choice not documented in code comments.

## Questions to Address in ADR (if created)

- Why was the FSM implemented as a standalone module rather than inline conditionals in OrderService?
- What criteria determine whether a transition should trigger a stock side effect (e.g., why PENDING→PAID debits and not PAID→PROCESSING)?
- What is the policy for evolving the FSM — specifically, can statuses be added, or is the enum considered stable?
- How should the WEBHOOKS module consume FSM events — by subscribing to the same predicates, or by receiving all `changeStatus` events and filtering internally?
- What happens to in-flight orders if the FSM rules change in production (e.g., an order currently in PROCESSING that has a transition removed)?

## Related Potential ADRs

- No other ORDERS potential ADRs identified in this analysis.
- Relates to DATA module patterns: prices in integer cents, `OrderStatusHistory` schema, `OrderNumberSequence` table — candidates for DATA module ADR identification.
- Relates to planned WEBHOOKS module: the FSM transition event is the proposed trigger for `publishWebhookEvent`.

## Additional Notes

The `isTerminal` function (transitions table entry is an empty array) enables the system to determine when no further transitions are possible without hardcoding terminal state names. This is a future-proof design detail that should be captured in the ADR — it means terminal states are derived, not declared.

The `delete` operation in OrderService uses a separate guard (`order.status !== PENDING && order.status !== CANCELLED`) rather than calling `isTerminal` — a minor inconsistency that may indicate the FSM contract is not yet fully unified across all order operations.
