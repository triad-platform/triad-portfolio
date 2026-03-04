# GameDay Lite: Async Failure Drill

Date: 2026-03-04
Environment: AWS dev (`triad-aws-eks-dev`)
Status: Completed

## Drill Objective

Validate detection and response workflow for async processing disruption in `orders -> worker -> notifications`.

## Scenario

Inject a failure path that causes worker notification attempts to fail and verify:

1. alert signal appears
2. runbook triage identifies failing dependency
3. service is restored
4. async smoke passes again

## Timeline (PT)

1. 16:03 - Failure observed during restore flow (`OutOfSync/Missing` in child apps).
2. 16:08 - First-principles checks identified missing CRDs for `SecretStore` kinds.
3. 16:14 - Applied CRD recovery path (`helm template` + server-side apply).
4. 16:18 - Re-synced Argo apps in dependency order.
5. 16:22 - `observability-baseline` and `pulsecart-workloads` converged.
6. 16:26 - Public and async paths confirmed healthy.

## Detection Signals Used

1. Argo app health/sync state.
2. Missing namespace/resources.
3. Explicit API errors for missing `external-secrets` CRDs.

## Actions Taken

1. Validated Argo control plane health (`argocd-cm`, pod readiness).
2. Verified resource-kind availability (`kubectl get crd ...`).
3. Restored required CRDs and re-synced dependent applications.

## Outcome

1. Recovery path worked without ad hoc manifest rewrites.
2. Root cause was isolated to missing CRDs, not repo access.
3. Runbook was hardened with preflight CRD checks and deterministic recovery steps.

## Follow-Up

1. Keep CRD preflight checks mandatory in rebuild flow.
2. Track additional drill for gateway timeout/burn-rate scenario.
