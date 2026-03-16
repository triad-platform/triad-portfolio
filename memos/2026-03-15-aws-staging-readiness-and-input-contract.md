# AWS Staging Readiness And Input Contract

This memo records the next Stage B maturity step after defining the AWS dev-to-staging contract and the staging GitOps promotion model.

The key change is that the project now has two operator-facing artifacts that turn the staging scaffold into a real readiness workflow:

1. a staging readiness checklist
2. a staging promotion input contract

## Why This Matters

The project now has:

1. a stage landing-zone scaffold
2. a future stage cluster placeholder
3. a non-live staging workload overlay
4. a future Argo staging app path

That structure is useful, but it is not enough by itself.

Without an explicit readiness checklist and promotion input contract, staging would still depend on:

1. operator memory
2. placeholder values surviving too long
3. accidental Argo activation before environment-specific inputs are real

## What Was Added

The platform repo now defines:

1. `docs/runbooks/aws-staging-readiness-checklist.md`
2. `docs/runbooks/aws-staging-promotion-input-contract.md`

Together they make these rules explicit:

1. stage landing-zone values must stop being examples before staging is treated as real
2. staging overlays must not contain unresolved `REPLACE_*` values
3. source revision, per-service digests, and rollback target must be known before promotion
4. Argo activation is a final intentional step, not part of placeholder scaffolding

## Principal-Level Value

This is a better platform move than adding more topology.

It shows:

1. environment maturity is being earned through explicit boundaries
2. promotion discipline is being designed before rollout
3. staging is being treated as a stronger operational contract, not just a copy of dev

## What Remains Next

The next useful step is to keep materializing Stage B without expanding topology:

1. fill the stage landing-zone placeholders with real values when stage is actually being stood up
2. fill the staging workload overlay placeholders from a reviewed digest set
3. decide when the staging Argo app is ready to be activated

The important outcome is that staging now has a clearer path from scaffold to environment, and that path is grounded in operator workflow rather than assumption.
