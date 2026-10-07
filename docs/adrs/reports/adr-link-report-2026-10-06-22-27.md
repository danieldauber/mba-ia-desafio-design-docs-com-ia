# ADR Relationship Analysis Report
Generated: 2026-10-06 22:27

## Scan Summary

- **Path scanned:** `docs/adrs/generated/`
- **Module:** WEBHOOKS
- **ADRs processed:** 5
- **ADRs updated:** 5
- **Relationships detected:** 7 pairs (13 link updates)
- **Git analysis:** No ADR-specific commits found; all ADRs share date 2026-06-24

---

## Relationship Breakdown

| Type | Pairs | Link updates |
|------|-------|-------------|
| Supersedes / Superseded by | 0 | 0 |
| Depends on / Used by | 4 | 8 |
| Related to | 3 | 6 |
| Amends / Amended by | 0 | 0 |
| **Total** | **7** | **14** |

---

## Detected Relationships

### Depends on / Used by (4 pairs)

| ADR | Depends on | Confidence | Evidence |
|-----|-----------|-----------|---------|
| ADR-002 | ADR-001 | 0.92 | Explicit mention of `webhook_outbox` table from ADR-001; DLQ preserves outbox readability |
| ADR-003 | ADR-001 | 0.95 | Decision outcome explicitly states "consume eventos registrados na tabela `webhook_outbox`" |
| ADR-004 | ADR-003 | 0.90 | Explicit: "O worker de entrega deve recuperar o segredo correto do endpoint no momento do despacho" |
| ADR-005 | ADR-001 | 0.88 | Explicit: "`event_id` UUID deve ser gerado na inserção do outbox" ties to ADR-001's outbox table |

### Related to (3 pairs)

| ADR A | ADR B | Confidence | Evidence |
|-------|-------|-----------|---------|
| ADR-002 | ADR-003 | 0.80 | Retry logic executes within the worker process; same operational domain |
| ADR-002 | ADR-005 | 0.75 | At-least-once semantics (ADR-005) are the root cause for duplicate deliveries handled by retry (ADR-002) |
| ADR-004 | ADR-005 | 0.70 | Both define per-delivery HTTP headers (X-Signature/X-Timestamp vs X-Event-Id); both reference Stripe/GitHub patterns |

---

## Dependency Graph

```
ADR-001 (Transactional Outbox)
  └─[Used by]──> ADR-002 (Retry + DLQ)
  │                └─[Related to]──> ADR-003 (Worker)
  │                └─[Related to]──> ADR-005 (At-least-once)
  └─[Used by]──> ADR-003 (Worker)
  │                └─[Used by]──> ADR-004 (HMAC-SHA256)
  │                                  └─[Related to]──> ADR-005 (At-least-once)
  └─[Used by]──> ADR-005 (At-least-once)
```

**No cycles detected.**

---

## Key Architectural Chains

1. **Core delivery pipeline**: ADR-001 (outbox) → ADR-003 (worker) → ADR-004 (security signing)
2. **Reliability contract**: ADR-001 (outbox) → ADR-002 (retry/DLQ) + ADR-005 (idempotency)
3. **Delivery semantics cluster**: ADR-002 + ADR-005 + ADR-004 form the delivery contract surface

---

## Foundational ADR Exclusion Applied

No foundational ADRs detected in this module. All ADRs are domain-specific webhook decisions.

---

## Supersession Analysis

All 5 ADRs share the same date (2026-06-24). No temporal gap detected. No version indicators ("v2", "migration", "replacement") found in titles. No file renames detected in git history.

**Result:** 0 supersession relationships.

---

## Warnings

None.

## Errors

None.
