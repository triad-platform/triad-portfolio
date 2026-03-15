# Runbook: Gateway Timeouts And Burn-Rate Pressure

Scope:

1. Synchronous request-path degradation in AWS dev.
2. Primary path: `api-gateway -> orders`.
3. Signals: elevated timeouts, elevated 5xx rate, and user-visible request failure risk.

## Trigger Signals

1. `PulseCartGatewayTimeouts`
2. `PulseCartGatewayHigh5xxRate`
3. Public `POST /v1/orders` failures or very slow responses

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

For a controlled drill, prefer a reversible sync-path failure:

1. temporarily disable Argo self-heal on `pulsecart-workloads`
2. scale `orders` to zero
3. observe alerting and metrics
4. restore `orders`
5. re-enable self-heal and refresh the app

Suggested commands:

```bash
kubectl patch app pulsecart-workloads -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":false}}}}'
kubectl scale deployment/orders -n pulsecart --replicas=0
```

Restore:

```bash
kubectl scale deployment/orders -n pulsecart --replicas=2
kubectl patch app pulsecart-workloads -n argocd --type merge -p '{"spec":{"syncPolicy":{"automated":{"prune":true,"selfHeal":true}}}}'
kubectl annotate app pulsecart-workloads -n argocd argocd.argoproj.io/refresh=hard --overwrite
```

## Exit Criteria

1. `PulseCartGatewayTimeouts` returns to baseline
2. public health and order-create path recover
3. `pulsecart-workloads` returns to `Synced` / `Healthy`
4. drill notes capture detect, mitigate, and recover timestamps
