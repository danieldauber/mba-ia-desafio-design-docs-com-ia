# Potential ADR: At-Least-Once Delivery with X-Event-Id Idempotency

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 110/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

The team explicitly chose at-least-once delivery semantics for webhook notifications, acknowledging that a customer endpoint may receive the same event more than once (due to network failures, worker restarts, or retry logic). To allow clients to deduplicate, each outbox event is assigned a UUID at insertion time (`event_id`), transmitted as the `X-Event-Id` request header. The responsibility for idempotent handling is delegated to the client.

This decision was driven by the practical infeasibility of exactly-once delivery: guaranteeing exactly-once would require bidirectional coordination with each customer's system. The team cited Stripe and GitHub as precedents for the at-least-once + `X-Event-Id` pattern. The Product Manager (Marcos) committed to documenting this in the developer portal so customers are aware.

## Why This Might Deserve an ADR

- **Impact**: Defines the delivery contract for every B2B customer integrating via webhooks. Customers who do not implement idempotent handling will process duplicate events, causing data inconsistencies on their side. This is an external-facing API contract decision.
- **Trade-offs**: Exactly-once would eliminate client-side dedup burden but requires two-phase commit or acknowledgment protocols that add significant complexity. At-least-once + event ID is the established industry standard and accepted by all parties.
- **Complexity**: The `event_id` UUID must be generated and stored on the `webhook_outbox` row at insertion, not at dispatch time. If the worker retries a failed delivery, it must reuse the same `event_id` (not generate a new one), so clients can correctly deduplicate.
- **Team Knowledge**: Developers implementing the worker retry path must understand that `X-Event-Id` must be stable across retries. Developers writing client integration documentation must communicate this contract clearly.
- **Future Implications**: If any customer integration cannot handle duplicate events, this becomes a support escalation. The decision to not pursue exactly-once is a permanent constraint unless the delivery infrastructure is significantly rearchitected.
- **Temporal Context**: Decided at project inception (2026-06-24). The exactly-once alternative was evaluated and discarded.

## Evidence Found in Codebase

### Key Files
- [`decisions.md`](../../../../../decisions.md) - Decision #10 (at-least-once + X-Event-Id), #34 (exactly-once rejected), #20 (full header list), #21 (payload structure with event_id)
- [`TRANSCRICAO.md`](../../../../../TRANSCRICAO.md) - Lines 147–158 (Diego's rationale, Stripe/GitHub reference, Marcos's documentation commitment)

### Code Evidence
```typescript
// Planned payload structure (decisions.md #21):
// {
//   event_id: string;       // UUID — stable across retries, generated at outbox insertion
//   event_type: string;     // e.g. "order.status_changed"
//   timestamp: string;      // ISO 8601 — time of status change
//   order_id: string;
//   order_number: string;
//   from_status: string;
//   to_status: string;
//   customer_id: string;
//   total_cents: number;
//   // NOTE: no order items — client calls GET /orders/:id for full detail
// }

// Planned request header (decisions.md #20):
// X-Event-Id: <uuid>  — same UUID on every retry attempt for this event
```

```typescript
// Planned outbox schema (inferred from decisions.md + TRANSCRICAO.md):
// webhook_outbox.event_id: Char(36) UUID — generated at INSERT, reused across retries
// NOT regenerated per delivery attempt
```

### Impact Analysis
- Introduced: 2026-06-24 (repository init, via meeting records)
- Modified: N/A — not yet implemented
- Affects: WEBHOOKS (outbox insert, worker dispatch), external customer integrations (idempotency contract), product documentation (developer portal)
- External-facing: this is part of the API contract with B2B customers Atlas Comercial, MaxDistribuição, Nova Cargo

### Alternatives

| Alternative | Rejected Reason |
|---|---|
| Exactly-once delivery | Requires bidirectional coordination; prohibitively complex for this system size |
| At-most-once delivery (fire-and-forget) | Risks silent event loss; unacceptable for order status notifications |

## Questions to Address in ADR (if created)

- Where exactly is `event_id` generated — inside `publishWebhookEvent(tx, ...)` at outbox insertion?
- Is `event_id` the same as the outbox row's primary key, or a separate field?
- Is the event deduplication window defined anywhere (e.g., "deduplicate within 7 days")?
- How is the at-least-once guarantee documented in the public developer portal?

## Related Potential ADRs
- [Transactional Outbox Pattern for Webhook Event Dispatch](./transactional-outbox-pattern-for-webhook-event-dispatch.md)
- [Exponential Backoff Retry with Dead Letter Queue](./exponential-backoff-retry-with-dead-letter-queue.md)
- [HMAC-SHA256 Per-Endpoint Payload Signing with Secret Rotation](./hmac-sha256-per-endpoint-payload-signing-with-secret-rotation.md)

## Additional Notes
The decision to omit order items from the payload (decisions.md #21, "payload enxuto") is a deliberate size-reduction choice that complements the at-least-once model — clients who receive duplicates fetch fresh detail from `GET /orders/:id` rather than re-processing a potentially stale embedded item list.
