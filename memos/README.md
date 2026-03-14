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
