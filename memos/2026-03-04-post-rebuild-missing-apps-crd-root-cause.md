# Post-Rebuild Missing Apps: CRD Root Cause

Date: 2026-03-04
Status: Accepted

## Incident

After AWS EKS teardown/rebuild and root app sync:

1. `observability-baseline` was `OutOfSync` / `Missing`
2. `pulsecart-workloads` was `OutOfSync` / `Missing`
3. target namespaces (`observability`, `pulsecart`) were not created

## First-Principles Diagnosis

ArgoCD could read repository source and render resources, but failed validation on resource kinds at apply time.

Key signal:

1. `ExternalSecret` / `SecretStore` resources failed because corresponding CRDs were missing.
2. Cluster had `externalsecrets.external-secrets.io` but was missing:
   - `secretstores.external-secrets.io`
   - `clustersecretstores.external-secrets.io`

This was not a repo access issue.

## Recovery That Worked

1. Render and apply external-secrets chart with CRDs enabled.
2. Apply with Kubernetes server-side apply.
3. Re-sync Argo apps in dependency order:
   - `external-secrets-prereqs`
   - `external-secrets`
   - `observability-baseline`
   - `pulsecart-workloads`

After CRDs existed, downstream apps converged.

## Hardening Updates

Runbook now includes:

1. CRD preflight check in Step 6.
2. first-principles decision tree for `OutOfSync/Missing` child apps.
3. tested recovery command sequence using `helm template` + server-side apply.

Updated runbook:

- `/Users/lseino/triad-platform/triad-kubernetes-platform/docs/runbooks/aws-dev-teardown-rebuild.md`
