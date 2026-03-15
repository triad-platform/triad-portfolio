# Runbook: Gateway Timeouts And Burn-Rate Pressure

Scope:

1. Synchronous request-path degradation in AWS dev.
2. Primary path: `api-gateway -> orders`.
3. Signals: elevated timeouts, elevated 5xx rate, and user-visible request failure risk.

## Trigger Signals

1. `PulseCartGatewayTimeouts`
2. `PulseCartGatewayHigh5xxRate`
3. Public `POST /v1/orders` failures or very slow responses
4. `orders` Deployment unexpectedly at `0` available replicas during a drill or outage

## Triage Sequence

1. Confirm whether the issue is on the synchronous path only or broader platform instability.

```bash
kubectl get app -n argocd
kubectl get pods -n pulsecart
curl -i https://pulsecart-dev.cloudevopsguru.com/healthz
```

2. Check `api-gateway` and `orders` logs.

```bash
kubectl logs -n pulsecart deployment/api-gateway --tail=200
kubectl logs -n pulsecart deployment/orders --tail=200
```

3. Check timeout and request metrics direction.

```bash
kubectl port-forward -n pulsecart svc/api-gateway 8080:8080
curl -sf http://localhost:8080/metrics | rg 'http_response_status_500_total|orders_forward_timeouts_total|http_request_duration'
```

4. Confirm whether `orders` is healthy or absent.

```bash
kubectl get deploy,po,svc -n pulsecart | rg 'api-gateway|orders'
```

Operational note:

1. In the March 14 drill, public `POST /v1/orders` returned `502 upstream request failed` while `api-gateway /healthz` still returned `200`.
2. Treat public path failure as higher-signal than component liveness alone when judging user impact.

## Immediate Mitigation

1. Restore `orders` capacity if pods are missing or unavailable.
2. Reconcile the `pulsecart-workloads` Argo app if workload drift is involved.
3. Re-test the public path.

```bash
kubectl annotate app pulsecart-workloads -n argocd argocd.argoproj.io/refresh=hard --overwrite
kubectl get app pulsecart-workloads -n argocd
curl -i https://pulsecart-dev.cloudevopsguru.com/healthz
```

## Burn-Rate Reasoning

Use the Phase 4 policy in `phase4-slo-error-budget.md`:

1. fast burn: immediate incident response
2. slow burn: same-day remediation plan

If the public order path stays impaired long enough to threaten the availability or latency SLO, bias toward restoring service first and root-causing after recovery.

## Drill Injection Path (GitOps-Aware)

For a controlled drill, use a reversible Git-driven workload overlay rather than fighting Argo with cluster-side patches.

The current drill overlay lives at:

1. `/Users/lseino/triad-platform/triad-kubernetes-platform/workloads/pulsecart/drills/gateway-timeout`

That overlay reuses the normal dev workload base and forces `orders` replicas to `0`.

Activation path:

1. update `apps/workloads/pulsecart-workloads.yaml`
2. change `spec.source.path` from `workloads/pulsecart/dev` to `workloads/pulsecart/drills/gateway-timeout`
3. commit and push to `develop`
4. wait for Argo to reconcile

Restore path:

1. revert `apps/workloads/pulsecart-workloads.yaml` back to `workloads/pulsecart/dev`
2. commit and push to `develop`
3. confirm `pulsecart-workloads` returns to `Synced` / `Healthy`

Suggested verification:

```bash
kubectl get app pulsecart-workloads -n argocd
kubectl get deploy orders -n pulsecart
kubectl get pods -n pulsecart
```

## Exit Criteria

1. `PulseCartGatewayTimeouts` returns to baseline
2. public health and order-create path recover
3. `pulsecart-workloads` returns to `Synced` / `Healthy`
4. drill notes capture detect, mitigate, and recover timestamps

## Follow-Up Gap

The March 14 execution showed that current alert coverage should be revisited for this scenario:

1. an `orders` outage produced clear public `502` responses
2. the expected alert path was not strong enough to count as complete detection evidence
3. alert rules and/or Prometheus expressions should be tightened before calling this scenario fully covered
