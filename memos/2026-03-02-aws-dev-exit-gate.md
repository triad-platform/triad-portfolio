# AWS Dev Exit Gate

Date: 2026-03-02
Status: Draft

## Purpose

This memo records whether the AWS dev environment is ready to be treated as the reference baseline before Azure work begins.

## What Is Proven

1. The application delivery loop works end to end.
   - `triad-app` builds and pushes images.
   - The build workflow updates the GitOps overlay in `triad-kubernetes-platform`.
   - ArgoCD reconciles the overlay.
   - Required cloud smoke validates both the synchronous request path and the asynchronous worker/notification path.

2. The platform control plane is real, not placeholder.
   - EKS is live.
   - ALB ingress, `external-dns`, and `external-secrets` are live.
   - Persistent storage is live through the EBS CSI add-on and the `gp3` storage class.

3. Managed dependencies are integrated.
   - RDS PostgreSQL is in use.
   - ElastiCache Redis is in use.
   - NATS is in-cluster and wired into the workload path.

4. Observability is operational.
   - Prometheus and Grafana are live with persistent storage.
   - Alertmanager is live.
   - Prometheus -> Alertmanager -> SNS -> email has been tested, including resolution.

5. The EKS baseline has been normalized after upgrade.
   - The cluster has been upgraded to Kubernetes `1.35`.
   - The repo now targets `1.35` as the declared AWS dev baseline.

## What Still Requires Human Intervention

This environment is reproducible, but it is not yet truly zero-touch.

1. Terraform backend bootstrap still starts locally.
   - The S3 bucket and DynamoDB lock table are intentionally bootstrapped first.
   - This is normal and acceptable.

2. ArgoCD initial installation is still a one-time bootstrap step.
   - After installation, Argo owns the in-cluster platform and workload reconciliation.
   - The first install is still manual.

3. GitHub repository secrets are still set manually.
   - Example: the cross-repo token used by the app build workflow to update the GitOps repo.
   - This is an organizational automation gap, not a cluster runtime gap.

4. SNS email subscription confirmation is inherently manual.
   - AWS requires email recipient confirmation.
   - This cannot be made fully hands-off in the same way as the rest of the stack.

5. Cloud credentials and local operator access remain prerequisites.
   - AWS credentials, `kubectl` access, and the ability to port-forward admin services are still assumed for operators.

## Readiness Judgment

The AWS dev environment now meets the practical Azure entry gate:

1. Normal application changes flow through CI, GitOps, and Argo without break-glass.
2. Required smoke validates both sync and async application behavior.
3. Observability and alerting are functional enough that troubleshooting no longer starts with ad hoc cluster forensics.
4. Remaining manual steps are bootstrap/admin concerns, not repeated recovery steps in the normal delivery path.

## Remaining Hardening Before Calling It "Zero-Touch"

1. Automate ArgoCD first-install bootstrap.
2. Replace repo-scoped cross-repo credentials with a stronger org-level automation pattern (for example, a GitHub App).
3. Move remaining bootstrap-only secrets and admin defaults fully out of repo-managed placeholders.
4. Document teardown and re-create as a single repeatable operator runbook.

## Cost Management Implication

Because the AWS dev environment is now a reproducible baseline rather than a fragile one-off build, it can be treated as a parkable environment.

That means:

1. It is reasonable to scale down or tear down dev when it is not actively needed.
2. Azure parity work does not require AWS dev to remain fully online at all times.
3. Cost control can now be part of normal platform operations instead of something avoided out of fear of a painful rebuild.

This does not mean the environment is fully zero-touch. It means the remaining manual work is now bounded enough that bringing it back is an operational task, not a rescue exercise.

## Azure Start Condition

Azure work should mirror the AWS model as it exists now, not re-invent it:

1. Landing zone first
2. AKS second
3. GitOps overlay and Argo reconciliation third
4. Secret sync, observability, and alerting parity after that

The goal is parity with the AWS operating model, not a different deployment model per cloud.
