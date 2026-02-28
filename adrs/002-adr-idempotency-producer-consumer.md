# ADR-002: End-to-End Idempotency Across Producer and Consumer

Status: Accepted
Date: 2026-02-27

## Context

PulseCart processes order creation through synchronous and asynchronous boundaries:

1. Client -> API Gateway -> Orders (`POST /v1/orders`)
2. Orders -> NATS (`orders.created.v1`)
3. Worker consumes event and calls Notifications

Network retries, client retries, and message replay can all create duplicate processing unless idempotency is enforced at multiple layers.

## Decision

Use layered idempotency:

1. Producer-side idempotency in `orders`
   - Require `Idempotency-Key` on create-order requests.
   - Reserve key in Redis (`SETNX` + TTL) before side effects.
   - Return `409` for duplicate requests.

2. Consumer-side idempotency in `worker`
   - Use event `order_id` as consumer idempotency key.
   - Reserve key in Redis (`SETNX` + TTL).
   - Skip duplicate side effects when event is replayed.

3. Validate behavior through E2E
   - First request returns `201`.
   - Duplicate request returns `409`.
   - Notifications side effect executes exactly once.

## Alternatives

1. Producer-only idempotency
   - Reject duplicate HTTP requests but trust event stream delivery.
2. Consumer-only idempotency
   - Accept duplicate producer writes and suppress only downstream effects.
3. Exactly-once delivery infrastructure
   - Rely on transport semantics only.

## Tradeoffs

### Benefits of layered approach

- Protects both synchronous write path and async side effects.
- Works with at-least-once delivery and replay semantics.
- Keeps implementation simple and understandable for a single-team codebase.

### Costs and limitations

- Redis key management and TTL tuning required.
- Possible key leakage if TTL strategy is poor.
- Current design is eventual consistency; DB write + publish is not yet outbox-atomic.

## Consequences

### Immediate effects

- Duplicate client retries do not create duplicate order effects.
- Duplicate event deliveries do not repeat notifications.
- Reliability behavior is observable via duplicate/error metrics.

### Follow-up work

1. Introduce transactional outbox in `orders` for DB + event atomicity.
2. Define service-specific idempotency key retention policy.
3. Add replay load tests and chaos scenarios for Redis/NATS degradation.

This decision prioritizes practical reliability now while keeping a clear upgrade path to stronger delivery guarantees.
