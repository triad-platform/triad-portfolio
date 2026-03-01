# Phase 2 Bootstrap Lessons

Date: 2026-03-01
Status: In Progress

## Scope

This memo captures the failures and corrections encountered while moving PulseCart from a local vertical slice into the first AWS + EKS deployment path.

## What Broke

1. Reusable GitHub Action manifest failed to load
   - Root cause: the composite action metadata was too brittle for the GitHub Actions parser.
   - Fix: normalize the action manifest shape and simplify the input naming.

2. ECR tag format was operationally unclear
   - Root cause: tags like `develop` and long raw SHAs were hard to scan across multiple service repositories.
   - Fix: include the service name in the tag (`orders-develop`, `orders-sha-<short-sha>`, etc.).

3. RDS engine pin drifted out of AWS support
   - Root cause: the module pinned PostgreSQL `16.4`, which was no longer available for new instance creation.
   - Fix: use major version `16` as the baseline and let AWS select a currently supported 16.x minor.

4. Password-based RDS setup created secret-handling risk
   - Root cause: the first Terraform shape required a plaintext database password input.
   - Fix: move to `manage_master_user_password = true` and treat the Secrets Manager ARN as the real output contract.

5. Terraform output assumed the RDS managed secret existed immediately
   - Root cause: the `master_user_secret` list can be empty during the transition from password-managed to AWS-managed credentials.
   - Fix: use `try(..., null)` in outputs during the migration path.

6. Cluster bootstrap initially violated repo ownership boundaries
   - Root cause: the EKS stack mutated subnet tags on network resources owned by the landing-zone repo.
   - Fix: move Kubernetes discovery tags into `triad-landing-zones` so `triad-kubernetes-platform` consumes cluster-ready subnets instead of modifying shared network primitives.

7. In-cluster NATS DNS was wrong
   - Root cause: app manifests pointed at `triad-nats.default.svc.cluster.local` while NATS was deployed in the `messaging` namespace.
   - Fix: change `NATS_URL` to `triad-nats.messaging.svc.cluster.local`.

8. ElastiCache introduced a transport mismatch
   - Root cause: ElastiCache was provisioned with transit encryption, but the app still used plain Redis clients.
   - Fix: add `REDIS_TLS_ENABLED=true` support in `orders` and `worker`, then wire that into Kubernetes manifests.

## What Was Learned

1. Managed services create contract changes, not just endpoint changes
   - Moving from local Docker services to AWS-managed services changes transport, auth, and secret distribution expectations.

2. Repo boundaries should be enforced by ownership, not convenience
   - If a repo owns network creation, it should also own the readiness tags and conventions required on that network.

3. "Real enough" bootstrap still needs operational readability
   - Tags, output names, and manifest defaults must be understandable during incidents, not just technically correct.

4. Secret safety must be designed before deployment speed
   - The right time to remove plaintext credentials is before the repo learns bad habits, not after.

## Remaining Gaps

1. RDS password still needs a platform-side secret sync path into Kubernetes.
2. The app still uses mutable `*-develop` tags; that is acceptable for dev but not a final rollout strategy.
3. ArgoCD is scaffolded but not yet the active control plane for the cluster.
4. External DNS / final Route 53 ALIAS automation is still pending.

## Next Phase 2 Focus

1. Stabilize the first successful in-cluster workload deployment.
2. Replace manual secret handling with a real synchronization path.
3. Move from direct `kubectl` bootstrap into GitOps-managed reconciliation.
