# GameDay Lite: Gateway Timeout And Burn-Rate Drill

Date: 2026-03-14
Environment: AWS dev (`triad-aws-eks-dev`)
Status: Planned / Ready To Execute

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

1. scale `orders` to zero temporarily
2. block `orders` readiness by introducing a bad dependency value in dev
3. create an artificial latency spike in the `orders` path if a safe debug hook exists

The preferred drill is the simplest one that creates observable gateway timeout behavior without introducing unrelated platform drift.

## Execution Checklist

1. Confirm baseline health before injection:
   - `kubectl get app -n argocd`
   - `kubectl get pods -n pulsecart`
   - `curl -i https://pulsecart-dev.cloudevopsguru.com/healthz`
2. Start timeline notes with exact timestamps.
3. Introduce the failure.
4. Watch alerting and dashboard signals.
5. Follow the relevant runbook branch.
6. Restore service.
7. Confirm the public path and metrics recover.

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

1. tighten timeout-specific runbook guidance if triage is ambiguous
2. tune alert thresholds if noise or delay is obvious
3. update the SLO narrative if burn-rate interpretation is unclear in practice
