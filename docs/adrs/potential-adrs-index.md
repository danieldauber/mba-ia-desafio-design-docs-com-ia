# Potential ADRs Index

## Analysis Progress

### Analyzed Modules
- **WEBHOOKS**: Webhook Notifications — 2026-10-06 — 4 high priority, 2 medium priority ADRs

### Pending Analysis
- **AUTH**: Authentication
- **USERS**: User Management
- **CUSTOMERS**: Customer Management
- **PRODUCTS**: Product Catalog
- **ORDERS**: Order Lifecycle
- **SHARED**: Shared Kernel
- **INFRA**: Infrastructure
- **DATA**: Data Layer
- **TEST**: Test Suite

---

## High Priority ADRs (must-document/)

### Module: WEBHOOKS

| Title | Category | Score | File |
|---|---|---|---|
| Transactional Outbox Pattern for Webhook Event Dispatch | Architecture | 145/150 | [Link](./potential-adrs/must-document/WEBHOOKS/transactional-outbox-pattern-for-webhook-event-dispatch.md) |
| HMAC-SHA256 Per-Endpoint Payload Signing with Secret Rotation | Security | 130/150 | [Link](./potential-adrs/must-document/WEBHOOKS/hmac-sha256-per-endpoint-payload-signing-with-secret-rotation.md) |
| Separate Worker Process with Polling for Outbox Consumption | Architecture | 130/150 | [Link](./potential-adrs/must-document/WEBHOOKS/separate-worker-process-with-polling-for-outbox-consumption.md) |
| Exponential Backoff Retry with Dead Letter Queue | Architecture | 125/150 | [Link](./potential-adrs/must-document/WEBHOOKS/exponential-backoff-retry-with-dead-letter-queue.md) |
| At-Least-Once Delivery with X-Event-Id Idempotency | Architecture | 110/150 | [Link](./potential-adrs/must-document/WEBHOOKS/at-least-once-delivery-with-event-id-idempotency.md) |

---

## Medium Priority ADRs (consider/)

### Module: WEBHOOKS

| Title | Category | Score | File |
|---|---|---|---|
| Snapshot Payload on Outbox Insertion | Architecture | 90/150 | [Link](./potential-adrs/consider/WEBHOOKS/snapshot-payload-on-outbox-insertion.md) |
| Event Filtering at Outbox Insertion by Configured Status List | Architecture | 80/150 | [Link](./potential-adrs/consider/WEBHOOKS/event-filtering-at-outbox-insertion-by-configured-status-list.md) |

---

## Summary

- High Priority (must-document/): 5 ADRs
- Medium Priority (consider/): 2 ADRs
- Total: 7 ADRs
- Modules Analyzed: 1 of 10
