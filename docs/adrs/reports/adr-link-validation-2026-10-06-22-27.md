# ADR Link Validation Report
Generated: 2026-10-06 22:27

## Summary

- **Links checked:** 14 (7 bidirectional pairs)
- **Valid:** 14 (100%)
- **Broken:** 0
- **Orphaned:** 0
- **Bidirectionality violations:** 0
- **Status consistency violations:** 0
- **Circular dependencies:** 0

---

## Link Validation Detail

| Source ADR | Relationship | Target ADR | Relative Path | Target Exists | Bidirectional |
|-----------|-------------|-----------|--------------|--------------|--------------|
| ADR-001 | Used by | ADR-002 | `./ADR-002-exponential-backoff-retry-com-dead-letter-queue.md` | OK | OK |
| ADR-001 | Used by | ADR-003 | `./needs-input/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md` | OK | OK |
| ADR-001 | Used by | ADR-005 | `./ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md` | OK | OK |
| ADR-002 | Depends on | ADR-001 | `./ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md` | OK | OK |
| ADR-002 | Related to | ADR-003 | `./needs-input/ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md` | OK | OK |
| ADR-002 | Related to | ADR-005 | `./ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md` | OK | OK |
| ADR-003 | Depends on | ADR-001 | `../ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md` | OK | OK |
| ADR-003 | Used by | ADR-004 | `./ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md` | OK | OK |
| ADR-003 | Related to | ADR-002 | `../ADR-002-exponential-backoff-retry-com-dead-letter-queue.md` | OK | OK |
| ADR-004 | Depends on | ADR-003 | `./ADR-003-processo-worker-separado-com-polling-para-consumo-do-outbox.md` | OK | OK |
| ADR-004 | Related to | ADR-005 | `../ADR-005-entrega-pelo-menos-uma-vez-com-idempotencia-x-event-id.md` | OK | OK |
| ADR-005 | Depends on | ADR-001 | `./ADR-001-transactional-outbox-para-despacho-de-eventos-webhook.md` | OK | OK |
| ADR-005 | Related to | ADR-002 | `./ADR-002-exponential-backoff-retry-com-dead-letter-queue.md` | OK | OK |
| ADR-005 | Related to | ADR-004 | `./needs-input/ADR-004-hmac-sha256-assinatura-payload-por-endpoint-com-rotacao-de-segredo.md` | OK | OK |

---

## Status Consistency

| ADR | Status | Has "Superseded by" | Consistent |
|-----|--------|-------------------|-----------|
| ADR-001 | Aceito | No | OK |
| ADR-002 | Aceito | No | OK |
| ADR-003 | Aceito | No | OK |
| ADR-004 | Aceito | No | OK |
| ADR-005 | Aceito | No | OK |

---

## Validation: PASSED
