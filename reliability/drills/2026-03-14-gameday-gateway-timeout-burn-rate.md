# GameDay Lite: Gateway Timeout And Burn-Rate Drill

Date: 2026-03-14
Environment: AWS dev (`triad-aws-eks-dev`)
Status: Completed

## Drill Objective

Validate detection and response workflow for synchronous path degradation at the public entrypoint.

The target scenario is sustained `api-gateway -> orders` timeout behavior severe enough to threaten the order API availability and latency SLOs.

## Scenario

Inject or simulate a condition that causes the gateway to accumulate order-forwarding timeouts and verify:

1. timeout-focused alert signal appears
2. operators can distinguish synchronous-path failure from async worker failure
3. mitigation restores request success before the error budget burn becomes unacceptable
4. public health and order-create path recover

## Primary Signals

1. `PulseCartGatewayTimeouts`
2. `PulseCartGatewayHigh5xxRate`
3. `api-gateway` logs showing upstream timeout behavior
4. `orders` health, readiness, and latency metrics

## SLO / Error-Budget Mapping

This drill specifically exercises:

1. SLO-1 availability for `POST /v1/orders`
2. SLO-2 P95 latency for the same path
3. fast-burn and slow-burn reasoning from `phase4-slo-error-budget.md`

## Suggested Injection Paths

Choose one controlled failure mode:

1. switch the workload app to the Git-native drill overlay that sets `orders` replicas to `0`
2. block `orders` readiness by introducing a bad dependency value in dev
3. create an artificial latency spike in the `orders` path if a safe debug hook exists

The preferred drill is the simplest one that creates observable gateway timeout behavior without introducing unrelated platform drift.
For the current AWS dev baseline, the preferred first drill is the reversible Git-driven overlay that moves `orders` to `0` replicas.

## Execution Checklist

1. Confirm baseline health before injection:
   - `kubectl get app -n argocd`
   - `kubectl get pods -n pulsecart`
   - `curl -i https://pulsecart-dev.cloudevopsguru.com/healthz`
2. Start timeline notes with exact timestamps.
3. Introduce the failure.
4. Watch alerting and dashboard signals.
5. Follow `runbook-gateway-timeout-burn-rate.md`.
6. Restore service.
7. Confirm the public path and metrics recover.

## Actual Execution Summary

The drill was executed by switching the Argo workload app from:

1. `workloads/pulsecart/dev`

to:

1. `workloads/pulsecart/drills/gateway-timeout`

Result:

1. Argo reconciled the drill overlay successfully.
2. `orders` scaled to `0` replicas and remained unavailable.
3. `api-gateway` remained healthy at its own `/healthz` endpoint.
4. Public `POST /v1/orders` requests returned `502 upstream request failed`.
5. The workload path was restored by reverting the Argo app source path back to `workloads/pulsecart/dev`.

## What Was Proven

1. The Git-native drill path works and respects the app-of-apps model.
2. User-path failure can exist while component liveness remains green.
3. The public synchronous path is visibly impaired when `orders` is absent.

## Gaps Exposed

1. The current alert set did not clearly surface the outage the way expected during the drill.
2. The present alert rules are weaker at detecting this exact `orders` unavailability case than the public endpoint behavior is.
3. Local metric capture during the drill was awkward enough that the operator path should be tightened further.

## Current Activation Mechanism

The drill should be activated by changing:

1. `/Users/lseino/triad-platform/triad-kubernetes-platform/apps/workloads/pulsecart-workloads.yaml`

from:

1. `workloads/pulsecart/dev`

to:

1. `workloads/pulsecart/drills/gateway-timeout`

Then commit and push to `develop`, let Argo reconcile, observe the failure, and revert the same file back to the normal path.

## Evidence To Capture

1. alert firing timestamps
2. dashboard screenshots or key metric snapshots
3. exact mitigation command(s)
4. time to detect
5. time to mitigate
6. time to steady state

## Success Criteria

1. the timeout condition is visible through alerts and metrics without relying on guesswork
2. responders use the documented path rather than ad hoc debugging
3. service recovers cleanly
4. follow-up actions are concrete and limited

## Expected Follow-Up

1. Add or tighten alert coverage for `orders` unavailability and user-path failure.
2. Improve the operator metric-capture path during drills.
3. Update the SLO and alert interpretation narrative if burn-rate reasoning remains unclear in practice.
