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

9. Mutable dev tags did not reliably refresh running pods
   - Root cause: Kubernetes deployments used mutable `*-develop` tags with `imagePullPolicy: IfNotPresent`, so new builds were not guaranteed to be pulled.
   - Fix: set `imagePullPolicy: Always` for the dev manifests that track mutable branch tags.

10. Database URL construction failed on special-character passwords
   - Root cause: the RDS managed password contained shell- and URL-sensitive characters, and manual secret creation introduced malformed connection strings.
   - Fix: stop manual copy/paste, URL-encode the password, and build the final `DATABASE_URL` programmatically.

11. DNS and TLS testing can fail even when the ALB is healthy
   - Root cause: the ALB was live before Route 53 was wired, and the certificate matched the custom hostname rather than the raw ALB hostname.
   - Fix: test with the ALB hostname plus the intended `Host` header (or complete the Route 53 ALIAS step).

12. The public order payload and the service schema drifted apart
   - Root cause: the deployed API examples used `quantity` and `unit_price`, while the `orders` service still decoded only `qty` and `price_cents`.
   - Fix: make the service accept both field shapes, then add item validation so zero-value items are rejected instead of producing broken downstream events.

## What Was Learned

1. Managed services create contract changes, not just endpoint changes
   - Moving from local Docker services to AWS-managed services changes transport, auth, and secret distribution expectations.

2. Repo boundaries should be enforced by ownership, not convenience
   - If a repo owns network creation, it should also own the readiness tags and conventions required on that network.

3. "Real enough" bootstrap still needs operational readability
   - Tags, output names, and manifest defaults must be understandable during incidents, not just technically correct.

4. Secret safety must be designed before deployment speed
   - The right time to remove plaintext credentials is before the repo learns bad habits, not after.

5. Mutable infrastructure and mutable image tags need explicit rollout behavior
   - If a system depends on mutable references in dev, the operational defaults must force refreshes instead of assuming cache invalidation.

6. Managed credentials are safer, but they shift complexity into encoding and delivery
   - Moving the password out of git is correct, but application-facing formats still need careful construction.

7. Contract drift across service boundaries is easy to miss until the async path is exercised
   - A request can still return `201` while emitting an invalid downstream event if field mapping and validation are not aligned.

## Remaining Gaps

1. RDS password still needs a platform-side secret sync path into Kubernetes.
2. The app still uses mutable `*-develop` tags; that is acceptable for dev but not a final rollout strategy even with `imagePullPolicy: Always`.
3. ArgoCD is scaffolded but not yet the active control plane for the cluster.
4. External DNS / final Route 53 ALIAS automation is still pending.

## Next Phase 2 Focus

1. Stabilize the first successful in-cluster workload deployment.
2. Replace manual secret handling with a real synchronization path.
3. Move from direct `kubectl` bootstrap into GitOps-managed reconciliation.
