# Potential ADR: Exponential Backoff Retry with Dead Letter Queue

**Module**: WEBHOOKS
**Category**: Architecture
**Priority**: Must Document (Score: 125/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

The team defined a 5-attempt exponential backoff retry schedule for failed webhook deliveries: 1 minute, 5 minutes, 30 minutes, 2 hours, 12 hours. After the fifth failure the event is moved from `webhook_outbox` to a separate `webhook_dead_letter` table. A DLQ replay endpoint (`POST /admin/webhooks/dead-letter/:id/replay`) allows an ADMIN user to requeue a dead-lettered event manually.

The specific backoff schedule was debated — a 3-attempt variant was proposed and rejected. The 5-attempt schedule covers a client maintenance window of approximately 2 hours, which matches a known outage pattern from the team's existing client base. The total time window from first failure to last retry is approximately 15 hours.

The DLQ was placed in a separate table (not as a status flag on `webhook_outbox`) to keep the main outbox readable and to provide an explicit audit trail with payload, failure reason, and timestamp.

## Why This Might Deserve an ADR

- **Impact**: Defines the resilience contract of the webhook system. Clients integrating with the API will rely on this behavior — it is part of the implicit SLA. The 5-attempt / 15-hour window is a business-level commitment, not just an implementation detail.
- **Trade-offs**: 5 attempts over 15 hours means a client who had a brief outage might receive aged notifications. Fewer attempts (3) were rejected as insufficient for planned maintenance windows. Indefinite retry was rejected because it leaves events perpetually pending.
- **Complexity**: The worker must track `retry_count`, `next_retry_at`, and `last_error` on outbox rows. DLQ replay must atomically move rows back to `webhook_outbox` with a reset `retry_count`. The ADMIN-only replay endpoint extends the auth model into the webhook management surface.
- **Team Knowledge**: Engineers must understand the retry state machine to correctly implement the worker. The 5-attempt schedule and backoff intervals are not derivable from the code alone without this decision being documented.
- **Future Implications**: The deferred email notification feature (decisions.md #26) will hook into the DLQ — when an event reaches dead-letter status, an email alert was proposed. This decision defines that trigger point.
- **Temporal Context**: Decided at project inception (2026-06-24). The 3-attempt alternative was explicitly discussed and rejected based on real historical client outage data.

## Evidence Found in Codebase

### Key Files
- [`decisions.md`](../../../../../decisions.md) - Decision #4 (retry schedule), #5 (DLQ table), #6 (replay endpoint with ADMIN role), #35 (3-attempt variant rejected), Deferred #26 (email on consecutive failures)
- [`TRANSCRICAO.md`](../../../../../TRANSCRICAO.md) - Lines 92–118 (retry debate, DLQ design)
- [`src/middlewares/auth.middleware.ts`](../../../../../src/middlewares/auth.middleware.ts) - `requireRole` middleware that will be reused for the replay endpoint

### Code Evidence
```typescript
// Planned retry schedule (decisions.md #4):
const RETRY_INTERVALS_MS = [
  1 * 60 * 1000,    // 1m
  5 * 60 * 1000,    // 5m
  30 * 60 * 1000,   // 30m
  2 * 60 * 60 * 1000,  // 2h
  12 * 60 * 60 * 1000, // 12h
];
const MAX_ATTEMPTS = 5;

// Planned DLQ table: webhook_dead_letter
// Fields: id (UUID), webhook_id, outbox_id, payload (snapshot), failure_reason, failed_at, replayed_at?
```

```typescript
// Replay endpoint (decisions.md #6) — reuses existing auth middleware:
// POST /admin/webhooks/dead-letter/:id/replay
// requireRole('ADMIN') — reuses src/middlewares/auth.middleware.ts
// Moves dead-letter row back to webhook_outbox with status = PENDING, retry_count = 0
// Logs audit: who replayed, when
```

### Impact Analysis
- Introduced: 2026-06-24 (repository init, via meeting records)
- Modified: N/A — not yet implemented
- Affects: WEBHOOKS (worker retry logic, DLQ table), DATA (new Prisma models for dead_letter), AUTH (requireRole reuse for replay endpoint), INFRA (error middleware must handle DLQ-specific errors)
- Business rationale: client maintenance windows historically reach ~2 hours; 5 attempts at 1m/5m/30m/2h/12h covers this window with margin

### Alternatives

| Alternative | Rejected Reason |
|---|---|
| 3 retry attempts | Insufficient to cover 2-hour client maintenance windows |
| Indefinite retry with backoff | Events would hang forever for permanently-offline clients |
| DLQ as status flag in webhook_outbox | Reduces readability of the main table; less clean audit trail |

## Questions to Address in ADR (if created)

- What is the exact SQL/Prisma schema for retry state on `webhook_outbox` rows (`retry_count`, `next_retry_at`, `last_error`)?
- Does the replay endpoint reset the delivery history or append a new delivery attempt?
- What audit fields are written on replay (who, when, from which DLQ row)?
- Is there a maximum number of replays allowed per dead-letter row?

## Related Potential ADRs
- [Transactional Outbox Pattern for Webhook Event Dispatch](./transactional-outbox-pattern-for-webhook-event-dispatch.md)
- [Separate Worker Process with Polling for Outbox Consumption](./separate-worker-process-with-polling-for-outbox-consumption.md)
- [At-Least-Once Delivery with X-Event-Id Idempotency](./at-least-once-delivery-with-event-id-idempotency.md)

## Additional Notes
The email-on-failure feature (deferred, decisions.md #26) is a direct downstream consumer of the DLQ trigger point. When that feature is implemented, it will need to reference this ADR to understand the threshold (5 failures) that defines "consecutive failures".
