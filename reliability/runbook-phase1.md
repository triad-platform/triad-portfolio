# PulseCart Phase 1 Runbook

Scope:
- Local/dev reliability incidents for the Phase 1 vertical slice.
- Services: `api-gateway`, `orders`, `worker`, `notifications`.
- Dependencies: Postgres, Redis, NATS.

## Quick Triage

1. Validate dependencies:
```bash
cd /Users/lseino/triad-platform/triad-app
make smoke
```

2. Validate service health:
```bash
curl -sf http://localhost:8080/healthz
curl -sf http://localhost:8081/healthz
curl -sf http://localhost:8082/healthz
curl -sf http://localhost:9091/metrics >/dev/null
```

3. Reproduce core flow:
```bash
make e2e
```

4. Inspect recent logs:
```bash
tail -n 120 .tmp/e2e-logs/api-gateway.log
tail -n 120 .tmp/e2e-logs/orders.log
tail -n 120 .tmp/e2e-logs/worker.log
tail -n 120 .tmp/e2e-logs/notifications.log
```

## Alert Playbooks

### PulseCartGatewayHigh5xxRate

Signal:
- `rate(triad_api_gateway_http_response_status_500_total[5m]) > 0.1`

Likely causes:
- Upstream orders failures or unexpected handler panics.
- Invalid upstream URL/network issue.

Checks:
```bash
curl -i http://localhost:8080/v1/orders \
  -X POST \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: debug-1" \
  -d '{"user_id":"u1","items":[{"sku":"s1","qty":1,"price_cents":100}],"currency":"USD"}'
```

Immediate mitigation:
1. Restart local services with `make e2e` (self-cleans stale processes).
2. Verify `ORDERS_URL` and orders health endpoint.
3. If persistent, inspect `orders` logs and metrics first.

### PulseCartGatewayTimeouts

Signal:
- `rate(triad_api_gateway_orders_forward_timeouts_total[5m]) > 0`

Likely causes:
- Orders latency increased (DB/NATS issues).
- Gateway timeout too low for current load.

Checks:
```bash
curl -sf http://localhost:8080/metrics | rg orders_forward_timeouts_total
curl -sf http://localhost:8081/metrics | rg create_order_duration
```

Immediate mitigation:
1. Confirm Postgres and NATS health.
2. Reduce local load/retry storm.
3. Temporarily raise gateway timeout for debugging only.

### PulseCartOrdersPersistenceErrors

Signal:
- `rate(triad_orders_create_order_persistence_errors_total[5m]) > 0`

Likely causes:
- Postgres unavailable.
- Schema missing/corrupted.
- DB connection or transaction failure.

Checks:
```bash
docker exec pulsecart-postgres pg_isready -U pulsecart
docker exec pulsecart-postgres psql -U pulsecart -d pulsecart -c '\dt'
curl -sf http://localhost:8081/metrics | rg persistence_errors
```

Immediate mitigation:
1. Restart Postgres container.
2. Restart `orders` service to re-run schema bootstrap.
3. Re-run `make e2e` to verify recovery.

### PulseCartOrdersPublishErrors

Signal:
- `rate(triad_orders_create_order_publish_errors_total[5m]) > 0`

Likely causes:
- NATS unavailable/disconnected.
- Event publish path failure.

Checks:
```bash
curl -sf http://localhost:8222/healthz
curl -sf http://localhost:8081/metrics | rg publish_errors
tail -n 120 .tmp/e2e-logs/orders.log
```

Immediate mitigation:
1. Restart NATS container.
2. Restart orders service to re-establish NATS connection.
3. Validate worker subscription log appears.

### PulseCartWorkerProcessingErrors

Signal:
- `rate(triad_worker_messages_errors_total[5m]) > 0`

Likely causes:
- Bad event payload.
- Idempotency store (Redis) issues.
- Notifications call failures.

Checks:
```bash
curl -sf http://localhost:9091/metrics | rg 'messages_errors_total|notifier_errors_total|idempotency_errors_total'
tail -n 120 .tmp/e2e-logs/worker.log
curl -sf http://localhost:8082/healthz
```

Immediate mitigation:
1. Confirm Redis and notifications health.
2. Inspect malformed payloads in worker logs.
3. Re-run local E2E with clean process state.

### PulseCartWorkerDuplicateSpike

Signal:
- `rate(triad_worker_messages_duplicates_total[5m]) > 0.5`

Likely causes:
- Producer retries or replay behavior.
- Repeated client requests with reused idempotency keys.

Checks:
```bash
curl -sf http://localhost:9091/metrics | rg messages_duplicates_total
curl -sf http://localhost:8081/metrics | rg create_order_duplicates_total
```

Immediate mitigation:
1. Verify this is expected replay safety behavior first.
2. If unexpected, inspect client/gateway retry pattern.
3. Confirm duplicate handling remains side-effect safe (`notifications` count remains stable).

## Escalation Criteria

Escalate and open an incident note when either is true:
1. Any critical alert persists > 15 minutes after mitigation.
2. `make e2e` fails repeatedly after dependency restarts.

Capture:
- alert name
- first seen timestamp
- impacted path
- mitigation attempts
- final resolution and follow-up action
