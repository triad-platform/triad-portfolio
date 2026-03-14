# AWS Rebuild Bootstrap Gaps After Cost Teardown

## Summary

The AWS EKS dev environment was intentionally torn down to reduce cost, then rebuilt through the normal Terraform plus ArgoCD path on March 14, 2026.

The rebuild succeeded and the cluster returned to a healthy known-good state, but two bootstrap gaps were exposed:

1. `external-secrets` CRDs were not fully available after the first Argo sync on a blank cluster.
2. Kyverno correctly blocked PulseCart Deployments because the app base was missing required workload labels on pod templates.

Neither issue required relaxing policy. Both were resolved by bringing the bootstrap path and app manifests into alignment with the intended controls.

## What Happened

### 1. EKS and ArgoCD bootstrap succeeded

The EKS cluster was recreated and ArgoCD root applications were applied successfully.

Most platform applications converged through normal reconciliation:

1. AWS load balancer controller prerequisites
2. `external-dns`
3. `external-secrets` prerequisites
4. storage baseline
5. NATS
6. Kyverno
7. cert-manager
8. observability baseline

### 2. `external-secrets` CRD bootstrap was incomplete

The first rebuild attempt left these CRDs missing:

1. `secretstores.external-secrets.io`
2. `clustersecretstores.external-secrets.io`

This blocked `observability-baseline` and `pulsecart-workloads` because Argo could not apply `SecretStore` and `ExternalSecret` resources reliably yet.

Recovery was cluster-side:

1. Render and apply the `external-secrets` chart with `installCRDs=true`
2. Refresh the blocked Argo applications

This confirms the current app-of-apps model is still not sufficient to guarantee CRD readiness on an empty cluster.

### 3. Kyverno blocked the app workload for the right reason

Once the CRD issue was resolved, `pulsecart-workloads` still failed.

The failure was not a bootstrap bug in Argo. It was a real policy violation:

1. the PulseCart Deployment manifests carried only `app`
2. the admission baseline required:
   - `app.kubernetes.io/name`
   - `app.kubernetes.io/part-of`

Kyverno denied all four workload Deployments until those labels were added to both the Deployment metadata and the pod template metadata.

### 4. Final cluster state

After:

1. restoring the missing `external-secrets` CRDs
2. fixing the workload labels in `triad-app`
3. refreshing the Argo workload application

the result was:

1. `pulsecart-workloads` = `Synced` / `Healthy`
2. all PulseCart pods `Running`
3. platform applications `Synced` / `Healthy`

## What This Proves

The AWS dev environment is reproducible enough to survive intentional cost teardown and come back through the normal repo-driven flow.

It still does **not** prove a zero-touch bootstrap. The remaining bootstrap boundary is now much clearer:

1. CRD-producing apps must be modeled more explicitly
2. downstream apps must not depend on chart-side CRD timing
3. admission policy must be treated as a first-class contract for app manifests

## Immediate Follow-Up

1. Move bootstrap-critical CRDs into explicit prereq resources or apps instead of relying on Helm CRD install timing.
2. Keep the Kyverno policy in `Enforce` mode and treat the label fix as the correct long-term behavior, not a one-off exception.
3. Continue AWS hardening before the next AKS/GKE workload parity pass.
