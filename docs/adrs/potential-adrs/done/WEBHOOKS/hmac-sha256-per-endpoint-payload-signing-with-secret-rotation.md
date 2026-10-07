# Potential ADR: HMAC-SHA256 Per-Endpoint Payload Signing with Secret Rotation

**Module**: WEBHOOKS
**Category**: Security
**Priority**: Must Document (Score: 130/150)
**Date Identified**: 2026-10-06

---

## What Was Identified

The team chose HMAC-SHA256 as the payload signing mechanism for outbound webhooks. Each registered webhook endpoint has its own unique secret (not a global platform secret). The signature is transmitted in the `X-Signature` request header. Secrets are rotatable: when a customer rotates their secret via the API, the old secret remains valid for a 24-hour grace period, after which it is invalidated.

This design came from Sofia (Security Engineer), who drove the discussion in the technical meeting (TRANSCRICAO.md, 09:19–09:22). The per-endpoint secret model was explicitly chosen over a global secret because a single leak would otherwise compromise all customer integrations. The 24-hour grace period was motivated by a real incident: a client previously leaked their secret in application logs and needed time to rotate without service interruption.

The `X-Timestamp` header was also specified to enable customers to detect replay attacks on their end.

## Why This Might Deserve an ADR

- **Impact**: Defines the security model for all outbound webhook deliveries. Every B2B customer integrating with the webhook system must implement HMAC-SHA256 verification. Changing the signing algorithm or key model would require coordinated migration with all active customers.
- **Trade-offs**: Per-endpoint secrets increase key management complexity (secrets must be generated, stored securely, and rotatable per webhook row). A global secret would be simpler but creates a single point of compromise.
- **Complexity**: The `webhook_outbox` row or the delivery worker must access the correct per-endpoint secret at dispatch time. Secret rotation requires the worker to attempt verification with both old and new secrets during the grace period. Secure storage of secrets in the `webhooks` table must be considered (encryption at rest, not returned in GET responses).
- **Team Knowledge**: Any developer implementing the delivery worker or the secret rotation endpoint must understand the grace period logic. Security reviewers (Sofia explicitly reserved 2 days for code review) need this documented.
- **Future Implications**: If the system ever needs to support webhook consumers that cannot implement HMAC (legacy systems), this decision creates friction. The per-endpoint model also complicates multi-tenancy scenarios where a customer has many webhooks.
- **Temporal Context**: Decided at project inception (2026-06-24). A global-secret alternative was explicitly evaluated and rejected based on a prior incident.

## Evidence Found in Codebase

### Key Files
- [`decisions.md`](../../../../../decisions.md) - Decision #7 (HMAC-SHA256 + per-endpoint secret), #8 (rotation + 24h grace), #9 (TLS mandatory), #20 (request headers), #33 (global secret rejected), #18 (10s HTTP timeout)
- [`TRANSCRICAO.md`](../../../../../TRANSCRICAO.md) - Lines 119–134 (Sofia's security rationale, grace period motivation)
- [`src/config/env.ts`](../../../../../src/config/env.ts) - Zod-validated env schema; `WEBHOOK_SIGNING_SECRET` or equivalent will need to be added, or secrets stored per-row in DB

### Code Evidence
```typescript
// Planned outbound request headers (decisions.md #20):
// X-Event-Id: <uuid>          — unique per event, for client-side dedup
// X-Signature: <hmac-sha256>  — HMAC-SHA256(body, endpoint_secret)
// X-Timestamp: <ISO-8601>     — dispatch timestamp; enables replay attack detection
// X-Webhook-Id: <uuid>        — registered webhook endpoint ID
// Content-Type: application/json

// Planned signing (Node.js crypto):
// import { createHmac } from 'node:crypto';
// const signature = createHmac('sha256', endpointSecret).update(bodyString).digest('hex');
```

```typescript
// Planned secret rotation logic (decisions.md #8):
// webhooks table: secret (current), previous_secret (nullable), secret_rotated_at (nullable)
// During grace period (< 24h after rotation): worker sends X-Signature computed from current_secret
// Customer may verify against either current or previous_secret
// After 24h: previous_secret is nulled out
```

```typescript
// TLS enforcement — Zod schema validation (decisions.md #9):
// webhookUrlSchema = z.string().url().startsWith('https://', { message: 'Webhook URL must use HTTPS' })
// http:// URLs rejected at registration time with a validation error
```

### Impact Analysis
- Introduced: 2026-06-24 (repository init, via meeting records)
- Modified: N/A — not yet implemented
- Affects: WEBHOOKS (secret generation, storage, rotation, signing), DATA (webhooks table schema — secret columns), INFRA (env config may need no change if secrets are per-row)
- Security review: Sofia explicitly reserved 2 business days for pre-deploy code review of HMAC implementation and secret generation

### Alternatives

| Alternative | Rejected Reason |
|---|---|
| Global platform-wide HMAC secret | Single leak compromises all customer integrations simultaneously |
| No signing (plain HTTP POST) | No integrity guarantee; customer cannot validate origin |
| Asymmetric signing (RSA/Ed25519) | Significantly more complex; HMAC-SHA256 is the industry standard (Stripe, GitHub) |

## Questions to Address in ADR (if created)

- How are per-endpoint secrets generated (length, entropy source — `crypto.randomBytes`)?
- Are secrets stored in plaintext or encrypted in the `webhooks` table?
- Are secrets ever returned after creation (only on POST, never on GET)?
- How does the worker retrieve the secret at dispatch time — single DB read per event, or cached?
- What is the exact grace period state machine for dual-secret verification?

## Related Potential ADRs
- [At-Least-Once Delivery with X-Event-Id Idempotency](./at-least-once-delivery-with-event-id-idempotency.md)
- [Transactional Outbox Pattern for Webhook Event Dispatch](./transactional-outbox-pattern-for-webhook-event-dispatch.md)

## Additional Notes
Sofia explicitly reserved two business days for security review of the HMAC implementation and secret generation code before any production deploy. This review requirement should be captured in the ADR status or process notes.
