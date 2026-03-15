# Memos

Short, executive-friendly technical memos.

## Available Memos

- `2026-02-27-phase1-retro.md`
  - Summary of what Phase 1 delivered, what failed during implementation, what was learned, and the Phase 2 handoff.
- `2026-03-01-phase2-bootstrap-lessons.md`
  - Summary of the AWS/EKS bootstrap failures, the corrections made, and the next operational gaps in Phase 2.
- `2026-03-02-aws-dev-exit-gate.md`
  - Explicit statement of what the AWS dev environment now proves, what still requires human bootstrap, and why that is sufficient to begin Azure parity work followed by GCP parity.
- `2026-03-04-post-rebuild-missing-apps-crd-root-cause.md`
  - First-principles incident record for `OutOfSync/Missing` Argo child apps caused by missing `external-secrets` CRDs and the tested recovery sequence.
- `2026-03-14-aws-rebuild-bootstrap-gaps.md`
  - Record of the successful AWS rebuild after cost teardown, the `external-secrets` CRD bootstrap gap, the Kyverno-enforced workload label failure, and the final recovery path.
- `2026-03-14-aws-admission-validation.md`
  - Record of the live EKS admission-policy validation showing deny for unapproved registry and missing labels, and allow for the signed-image path derived from the live GitOps overlay.
- `2026-03-14-prometheus-rule-reload-gap.md`
  - Incident-style record of the Prometheus rule reload gap discovered during the AWS reliability drill, the troubleshooting commands used, the root cause, and the permanent sidecar-based fix.
- `2026-03-15-outbox-delivery-gap.md`
  - Incident-style record of the first transactional outbox validation failure, the troubleshooting commands used, the root cause in NATS publish semantics, and the flush-based fix.
- `2026-03-15-pinned-supply-chain-baseline.md`
  - Record of the first successful AWS supply-chain run after pinning `triad-app` to a reviewed `triad-ci-security` reusable-action commit.
