# Memo: Orders Outbox Delivery Gap After Initial Rollout

Date: 2026-03-15
Status: Closed

## Summary

The first deployed version of the orders transactional outbox incorrectly treated `nats.Conn.Publish()` success as proof of broker delivery.

During live validation, `POST /v1/orders` continued returning `201` after NATS was intentionally removed, but:

1. `triad_orders_outbox_pending_events` stayed at `0`
2. `triad_orders_outbox_events_published_total` did not reflect a real broker outage

That meant the relay could mark outbox rows as published even when NATS was unavailable.

The fix was to require a successful NATS flush before treating the publish as successful.

## Trigger

The intent of the outbox design was:

1. synchronous order acceptance succeeds after durable order + outbox write
2. async completion can lag if NATS is unavailable
3. backlog metrics and alerts reflect that lag

The validation drill was:

```bash
kubectl scale deployment nats -n nats --replicas=0
kubectl get all -n nats
curl -i https://pulsecart-dev.cloudevopsguru.com/v1/orders \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: outbox-drill-check-1" \
  -d '{"user_id":"u1","items":[{"sku":"s1","qty":1,"price_cents":100}],"currency":"USD"}'
```

Observed result:

1. order create still returned `201`
2. `nats` namespace had no running resources
3. outbox backlog did not increase

## Troubleshooting Sequence

### 1. Confirm the synchronous path still succeeded

```bash
curl -i https://pulsecart-dev.cloudevopsguru.com/v1/orders \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: outbox-drill-check-1" \
  -d '{"user_id":"u1","items":[{"sku":"s1","qty":1,"price_cents":100}],"currency":"USD"}'
```

Result:

1. request returned `201`

### 2. Confirm NATS was actually absent

```bash
kubectl get all -n nats
```

Result:

1. `No resources found in nats namespace.`

### 3. Inspect orders metrics directly

```bash
kubectl port-forward -n pulsecart svc/orders 18081:8081
curl -sf http://localhost:18081/metrics | grep -E 'outbox_pending_events|outbox_events_published_total|outbox_events_failed_total|create_order_success_total'
```

Result:

1. order success metric increased
2. outbox backlog remained `0`
3. relay failure metric did not reflect the outage

### 4. Inspect orders logs

```bash
kubectl logs -n pulsecart deployment/orders --tail=200
```

Result:

1. no meaningful relay-failure evidence appeared

## Root Cause

The relay used:

1. `nats.Conn.Publish()`

as the success condition for delivery.

That only proves the NATS client accepted the message into its local buffer. It does not prove the broker accepted it.

With the broker unavailable, the client could still return `nil` from `Publish()` before a real network flush failure was observed.

## Fix

The NATS publisher in `triad-app` now requires:

1. `Publish()`
2. `FlushTimeout(3 * time.Second)`
3. clean `LastError()`

before the relay treats the outbox event as delivered and marks `published_at`.

This makes the outbox semantics align with the intended reliability boundary:

1. accepted order stays durable
2. backlog remains pending when the broker is unavailable
3. alerts and metrics can now surface real async lag

## What This Proved

1. A transactional outbox is only as correct as its delivery confirmation rule.
2. Client-library enqueue success is not the same as broker acceptance.
3. Live drills are necessary to validate reliability assumptions that look correct in code review.

## Follow-Up

1. Re-run the NATS outage drill after rollout of the flush-based fix.
2. Verify `triad_orders_outbox_pending_events` rises above `0` during broker outage.
3. Verify `PulseCartOrdersOutboxRelayErrors` and, if sustained long enough, `PulseCartOrdersOutboxBacklog`.
