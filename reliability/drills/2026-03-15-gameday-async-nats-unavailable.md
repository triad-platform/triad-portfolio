# AWS Gameday: Async NATS Unavailable

Date:

1. March 15, 2026

Scope:

1. AWS dev async dependency failure
2. path under test: `orders -> outbox relay -> messaging/triad-nats -> worker`

## Goal

Prove that when the real in-cluster NATS dependency is unavailable:

1. `orders` still accepts requests
2. async delivery intent is retained in the outbox
3. backlog and relay-failure signals reflect the outage
4. the outage can be induced under GitOps ownership without Argo immediately undoing the drill

## Preconditions

1. NATS drill activation was switched through `apps/platform/nats.yaml`
2. Argo reconciled `nats` against `platform/nats/drills/unavailable`
3. `messaging/triad-nats` had:
   - Deployment `0/0`
   - no ready pod
   - no endpoint subsets

## Commands Used

Verify drill state:

```bash
kubectl get app nats -n argocd
kubectl get deploy,svc,pods -n messaging | grep nats
kubectl get endpoints -n messaging triad-nats -o yaml
```

Send test orders:

```bash
curl -i -X POST https://pulsecart-dev.cloudevopsguru.com/v1/orders \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: maturity-async-7' \
  --data-binary @/tmp/pulsecart-order.json

curl -i -X POST https://pulsecart-dev.cloudevopsguru.com/v1/orders \
  -H 'Content-Type: application/json' \
  -H 'Idempotency-Key: maturity-async-8' \
  --data-binary @/tmp/pulsecart-order.json
```

Check decisive metrics:

```bash
curl -sf http://localhost:18081/metrics | grep outbox_pending_events
curl -sf http://localhost:18081/metrics | grep outbox_events_failed_total
curl -sf http://localhost:18081/metrics | grep outbox_events_published_total
curl -sf http://localhost:18081/metrics | grep create_order_success_total
```

## Observed Result

Public request path:

1. both order-create requests returned `201`

Orders metrics:

1. `triad_orders_outbox_pending_events 2`
2. `triad_orders_outbox_events_failed_total 4`
3. `triad_orders_outbox_events_published_total 1`
4. `triad_orders_create_order_success_total 3`

## What This Proves

1. the Git-native NATS-unavailable drill path works under Argo ownership
2. `orders` continues accepting requests while NATS is unavailable
3. async delivery is no longer falsely treated as successful during the outage
4. backlog and relay-failure metrics now reflect the real dependency failure

## Why This Matters

This closes the main remaining AWS dev maturity gap.

The unresolved question was no longer bootstrap, policy, or operator access. It was whether the async dependency path behaved correctly when the real GitOps-managed NATS runtime was unavailable.

The answer is now yes, when the drill is executed through the correct owner boundary.

## Follow-Up

1. restore `apps/platform/nats.yaml` back to `platform/nats`
2. verify backlog drains after NATS returns
3. treat AWS dev as materially closer to the maturity exit gate for the next platform phase
