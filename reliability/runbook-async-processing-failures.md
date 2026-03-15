# Runbook: Async Processing Failures (orders -> worker -> notifications)

Scope:

1. Async order completion failures in local or AWS dev environments.
2. Services: `orders`, `worker`, `notifications`, plus NATS and Redis dependencies.

## Trigger Signals

1. `PulseCartWorkerProcessingErrors` alert fires.
2. `PulseCartOrdersOutboxRelayErrors` alert fires.
3. `PulseCartOrdersOutboxBacklog` alert fires.
2. Worker error metrics increase:
   - `triad_worker_messages_errors_total`
   - `triad_worker_notifier_errors_total`
   - `triad_worker_idempotency_errors_total`
3. Cloud/local smoke reports async completion failure.

## Triage Sequence

1. Confirm worker and notifications health.

```bash
kubectl get pods -n pulsecart
kubectl logs -n pulsecart deployment/worker --tail=200
kubectl logs -n pulsecart deployment/notifications --tail=200
```

2. Confirm NATS and Redis connectivity paths.

```bash
kubectl get pods -n nats
kubectl logs -n pulsecart deployment/orders --tail=120
kubectl logs -n pulsecart deployment/worker --tail=120
```

3. Confirm metrics direction (orders outbox backlog vs worker processing).

```bash
kubectl port-forward -n pulsecart svc/orders 18081:8081
kubectl port-forward -n pulsecart svc/worker 19091:9091
curl -sf http://localhost:18081/metrics | grep -E 'outbox_pending_events|outbox_events_published_total|outbox_events_failed_total'
curl -sf http://localhost:19091/metrics | grep -E 'messages_processed_total|messages_errors_total|notifier_errors_total|idempotency_errors_total'
```

## Immediate Mitigation

1. Restart failed worker/notifications pods.
2. If outbox backlog is growing, inspect the `orders` logs before restarting anything else.
3. Validate `NOTIFICATIONS_URL`, `NATS_URL`, `REDIS_ADDR` env values in workload manifests.
4. Re-run cloud smoke once backlog and worker error metrics stop worsening.

## Deep Fix Guidance

1. If outbox relay errors dominate, inspect NATS connectivity, relay logs, and whether pending events are draining after recovery.
2. If notifier errors dominate, inspect notifications service response path and timeouts.
3. If idempotency errors dominate, inspect Redis connectivity/auth and duplicate-key logic.
4. If processing errors dominate with malformed payloads, inspect event schema/contract drift.

Troubleshooting note from the March 15 validation:

1. if order create continues returning `201` while NATS is down but `triad_orders_outbox_pending_events` stays at `0`, suspect delivery-confirmation logic in the relay rather than Prometheus or Alertmanager
2. the first implementation incorrectly treated `nats.Conn.Publish()` success as broker delivery
3. the permanent fix requires a successful NATS flush before the outbox row is marked published

## Exit Criteria

1. `triad_orders_outbox_pending_events` returns to `0`.
2. Worker error rate returns to baseline.
3. Async cloud smoke passes.
4. No sustained increase in duplicate side effects.
