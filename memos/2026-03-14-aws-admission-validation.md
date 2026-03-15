# AWS Admission Validation After Rebuild

## Summary

After proving the AWS rebuild contract on March 14, 2026, the admission enforcement baseline was validated against the live EKS cluster.

The validation result was the expected one:

1. unapproved registry workload was denied
2. missing-label workload was denied
3. signed-image workload rendered from the live GitOps overlay was allowed

This closes a practical Phase 3 loop on the AWS reference environment: rebuild, reconcile, and policy enforcement now line up with each other.

## What Was Tested

Three cases were exercised against the live cluster:

1. `pod-deny-unapproved-registry.yaml`
   - expected denial by approved-registry policy
2. `pod-deny-missing-labels.yaml`
   - expected denial by required-label policy
3. `render-pod-allowed.sh`
   - expected allow path using the same digest-pinned ECR image shape that Argo deploys from the live GitOps overlay

## Result

The cluster behavior matched the intended guardrails:

1. Kyverno denied workloads from unapproved registries.
2. Kyverno denied workloads missing the required `app.kubernetes.io/*` labels.
3. A properly labeled workload using a signed digest from the live overlay was accepted.

## Why This Matters

This is more meaningful than a static policy file review.

It demonstrates that:

1. the enforcement policy is active in the cluster
2. the app manifests now satisfy the enforced contract
3. the GitOps image promotion path produces runtime artifacts that admission policy accepts

## Next Move

With rebuild and admission validation both proven, the next AWS hardening focus should move toward Phase 4:

1. alert fidelity
2. dashboard and runbook accuracy
3. at least one fresh failure drill against the current rebuilt environment
