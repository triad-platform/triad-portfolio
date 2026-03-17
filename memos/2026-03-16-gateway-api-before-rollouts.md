# Gateway API Before Rollouts

This memo records the next traffic-maturity decision in the AWS Stage B path.

The project should learn and evaluate Gateway API before adopting Argo Rollouts.

## Decision

The chosen sequence is:

1. keep the proven ALB `Ingress` path as the active baseline
2. introduce a staging-only Gateway API evaluation scaffold
3. defer Argo Rollouts until staging promotion and rollback are already real and trusted

## Why

Gateway API improves the platform in ways that fit the current maturity stage:

1. clearer traffic API
2. clearer edge-to-app ownership boundary
3. cleaner route model than annotation-heavy ingress
4. stronger modern Kubernetes networking knowledge

Argo Rollouts is still early because it depends on a stronger existing discipline around:

1. promotion
2. rollback
3. post-promotion verification
4. staged exposure as a real operational need

## What Was Added

The platform repo now includes:

1. a non-live Gateway API evaluation overlay for staging
2. a non-live Argo app scaffold for that overlay
3. a runbook describing what must still be chosen before activation

This is the right kind of progress:

1. modern platform learning
2. no premature rollout controller
3. no accidental live traffic cutover

## Principal-Level Value

This keeps complexity earned.

The project is not adding Gateway API or Rollouts because they are fashionable.
It is adding the next concept that best improves the platform model for the current stage.
