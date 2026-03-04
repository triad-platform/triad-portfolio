# Phase 4 SLO And Error Budget (PulseCart)

Date: 2026-03-04
Status: Active

## Scope

Primary user journey:

1. `POST /v1/orders` accepted by `api-gateway` and persisted by `orders`.
2. Async completion path processes event through `worker` and calls `notifications`.

Environment:

1. AWS dev baseline (`triad-aws-eks-dev`).
2. Measurement windows are for Phase 4 readiness, not production SLA commitments.

## SLI Definitions

### SLI-1: Order API availability

Definition:

1. Ratio of successful non-5xx responses for `POST /v1/orders`.
2. Source metric: `triad_api_gateway_http_response_status_500_total` and request totals.

PromQL shape:

```promql
1 - (
  sum(rate(triad_api_gateway_http_response_status_500_total[5m]))
  /
  sum(rate(triad_api_gateway_http_requests_total[5m]))
)
```

### SLI-2: Order API latency

Definition:

1. P95 request latency for `POST /v1/orders` at `api-gateway`.
2. Source metric: `triad_api_gateway_http_request_duration_seconds_bucket`.

PromQL shape:

```promql
histogram_quantile(
  0.95,
  sum by (le) (rate(triad_api_gateway_http_request_duration_seconds_bucket[5m]))
)
```

### SLI-3: Async completion success

Definition:

1. Ratio of worker processed-success outcomes to total worker message processing attempts.
2. Source metrics: `triad_worker_messages_processed_total`, `triad_worker_messages_errors_total`.

PromQL shape:

```promql
sum(rate(triad_worker_messages_processed_total[5m]))
/
(
  sum(rate(triad_worker_messages_processed_total[5m]))
  +
  sum(rate(triad_worker_messages_errors_total[5m]))
)
```

## SLO Targets

Rolling 30-day targets:

1. SLO-1 Availability: `>= 99.5%`.
2. SLO-2 Latency (P95): `<= 750ms`.
3. SLO-3 Async completion success: `>= 99.0%`.

## Error Budget Policy

Monthly error budget:

1. SLO-1 budget: `0.5%` failed requests.
2. SLO-3 budget: `1.0%` failed async processing attempts.

Burn-rate response policy:

1. Fast burn: `>= 10x` over 1h -> immediate incident response.
2. Slow burn: `>= 2x` over 6h -> same-day remediation plan.
3. Budget exhausted -> freeze non-reliability feature changes in affected path until recovery actions are merged.

## Alert Mapping

Mapped initial alerts (see `alerts/pulsecart-phase1-alerts.yml`):

1. `PulseCartGatewayHigh5xxRate`
2. `PulseCartGatewayTimeouts`
3. `PulseCartWorkerProcessingErrors`
4. `PulseCartWorkerDuplicateSpike` (guardrail signal, not direct SLO burn)

## Dashboards

Primary dashboard source:

1. `dashboards/pulsecart-phase1-overview.json`

## Review Cadence

1. Weekly SLO review during Phase 4.
2. Monthly threshold adjustment after baseline behavior stabilizes.
3. Re-baseline before Azure parity begins.
