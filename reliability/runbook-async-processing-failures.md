# Runbook: Async Processing Failures (orders -> worker -> notifications)

Scope:

1. Async order completion failures in local or AWS dev environments.
2. Services: `orders`, `worker`, `notifications`, plus NATS and Redis dependencies.

## Trigger Signals

1. `PulseCartWorkerProcessingErrors` alert fires.
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

3. Confirm metrics direction (errors vs processed).

```bash
kubectl port-forward -n pulsecart svc/worker 9091:9091
curl -sf http://localhost:9091/metrics | rg 'messages_processed_total|messages_errors_total|notifier_errors_total|idempotency_errors_total'
```

## Immediate Mitigation

1. Restart failed worker/notifications pods.
2. Validate `NOTIFICATIONS_URL`, `NATS_URL`, `REDIS_ADDR` env values in workload manifests.
3. Re-run cloud smoke once metrics stop worsening.

## Deep Fix Guidance

1. If notifier errors dominate, inspect notifications service response path and timeouts.
2. If idempotency errors dominate, inspect Redis connectivity/auth and duplicate-key logic.
3. If processing errors dominate with malformed payloads, inspect event schema/contract drift.

## Exit Criteria

1. Worker error rate returns to baseline.
2. Async cloud smoke passes.
3. No sustained increase in duplicate side effects.
