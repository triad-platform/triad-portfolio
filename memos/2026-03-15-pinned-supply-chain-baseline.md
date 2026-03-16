# Pinned Supply-Chain Baseline

Date:

1. March 15, 2026

Purpose:

1. record the first successful AWS supply-chain evidence run after pinning `triad-app` to a reviewed `triad-ci-security` reusable-action commit
2. close the gap where cross-repo security actions were still being consumed from mutable branch refs

## Context

The `triad-app` build workflow previously depended on `triad-ci-security/reusable-actions/*` through moving refs such as `main` and `develop`.

That was workable for early iteration but weak for a defensible supply-chain story because:

1. the exact security logic executed by the workflow could drift without an intentional version bump
2. the evidence run and the reviewed security implementation were not tightly coupled

## Change

The workflow in `triad-app/.github/workflows/build-and-push-ecr.yml` was updated to pin the reusable build, SBOM, scan, and sign actions to:

1. `91b3160c081f2e0d2c2a9a903363586a3f4f9151`

That commit is now the reviewed supply-chain implementation boundary for the current AWS reference baseline.

## Validated Result

After the pin was introduced, a fresh `develop` workflow run was triggered and completed successfully.

What this run now proves:

1. the current AWS build path still builds and pushes images successfully
2. signing still succeeds on the pinned security implementation
3. the reviewed `triad-ci-security` action commit is operationally valid for the current `triad-app` workflow

## Why This Matters

This is a stronger platform/security posture than following mutable action branches.

The main improvement is not just reproducibility of the app build. It is reproducibility of the security behavior attached to the build.

That matters for:

1. evidence review
2. later incident analysis
3. intentional version advancement of CI security behavior
4. keeping the AWS reference path defensible as a principal-level platform narrative

## Next Step

Do not treat this as the end of the supply-chain story.

The next expected discipline is:

1. advance the `triad-ci-security` action pin intentionally when reviewed changes are ready
2. record that version movement as part of the evidence trail
3. return platform hardening focus to runtime secret/config cleanup and live reliability validation
