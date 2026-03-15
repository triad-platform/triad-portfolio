# Memo: Prometheus Rule Reload Gap During AWS Reliability Drill

Date: 2026-03-14
Status: Closed

## Summary

During the AWS synchronous-path outage drill, the new Prometheus alert rules were present in the live cluster ConfigMap but were not visible in Prometheus rule evaluation until a manual `/-/reload` was triggered.

This was not an Alertmanager routing failure. It was a Prometheus config reload gap.

The permanent fix was to add a `configmap-reload` sidecar to the Prometheus deployment so Argo-delivered ConfigMap updates automatically trigger Prometheus reloads.

## Trigger

The drill intentionally reconciled the workload app to:

1. `workloads/pulsecart/drills/gateway-timeout`

That forced `orders` to `0` replicas and produced real public request failures:

1. `POST /v1/orders` returned `502 upstream request failed`
2. `api-gateway /healthz` still returned `200`

The alert baseline had just been strengthened with:

1. `PulseCartOrdersUnavailable`
2. `PulseCartGatewayUpstreamFailureRatio`

But no new alerts appeared in Alertmanager immediately.

## Troubleshooting Sequence

### 1. Verify Argo delivered the updated rule ConfigMap

```bash
kubectl get configmap prometheus-rules -n observability -o yaml | rg 'PulseCartOrdersUnavailable|PulseCartGatewayUpstreamFailureRatio'
```

Result:

1. The live ConfigMap contained both new alert definitions.

### 2. Check whether Prometheus had loaded the rules

```bash
kubectl port-forward -n observability svc/prometheus 9090:9090
curl -s http://localhost:9090/api/v1/rules | rg 'PulseCartOrdersUnavailable|PulseCartGatewayUpstreamFailureRatio|state'
curl -s http://localhost:9090/api/v1/alerts | rg 'PulseCartOrdersUnavailable|PulseCartGatewayUpstreamFailureRatio|state|activeAt'
```

Result:

1. Prometheus still showed only the older alert set.
2. The new rules were not being evaluated yet.

### 3. Prove the underlying outage signal was real

```bash
curl -sG http://localhost:9090/api/v1/query --data-urlencode 'query=up{job="orders"}'
curl -sG http://localhost:9090/api/v1/query --data-urlencode 'query=max_over_time(up{job="orders"}[2m])'
```

Result:

1. `up{job="orders"}` returned `0`
2. `max_over_time(up{job="orders"}[2m])` returned `0`

This proved the rule expression itself should have been true if Prometheus had reloaded the new rule file.

### 4. Verify Prometheus supported live reload

```bash
kubectl get deploy prometheus -n observability -o yaml | rg 'enable-lifecycle|/-/reload'
```

Result:

1. Prometheus was already started with `--web.enable-lifecycle`

### 5. Force a manual reload

```bash
kubectl port-forward -n observability svc/prometheus 9090:9090
curl -X POST http://localhost:9090/-/reload
```

### 6. Re-check live rule state

```bash
curl -s http://localhost:9090/api/v1/rules | rg 'PulseCartOrdersUnavailable|PulseCartGatewayUpstreamFailureRatio|state'
curl -s http://localhost:9090/api/v1/alerts | rg 'PulseCartOrdersUnavailable|PulseCartGatewayUpstreamFailureRatio|state|activeAt'
```

Result:

1. `PulseCartOrdersUnavailable` appeared immediately
2. it progressed from `pending` to `firing`
3. Alertmanager routed it
4. SNS delivered the alert to email

## Root Cause

Argo updated the `prometheus-rules` ConfigMap, but the Prometheus pod did not have an automatic reload mechanism watching the mounted ConfigMap volumes.

This left the platform in an inconsistent state:

1. cluster config was correct
2. Prometheus runtime state was stale

## Fix

The Prometheus deployment in `triad-kubernetes-platform` now includes:

1. a `configmap-reload` sidecar
2. watches on `/etc/prometheus` and `/etc/prometheus-rules`
3. automatic `POST` to `http://127.0.0.1:9090/-/reload`

This removes the manual operator reload step after rule or scrape config changes.

## What This Proved

1. The direct `orders` outage alert path is valid end to end.
2. The problem was not Alertmanager routing.
3. Argo-delivered observability changes need runtime reload support to be operationally trustworthy.

## Follow-Up

1. Keep the reload sidecar in the AWS reference baseline.
2. Keep Azure and GCP observability variants aligned with the same behavior.
3. Treat future observability rule changes as incomplete until both Git state and runtime reload behavior are validated.
