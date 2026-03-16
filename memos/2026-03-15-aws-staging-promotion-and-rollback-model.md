# AWS Staging Promotion And Rollback Model

Date:

1. March 15, 2026

Purpose:

1. define the next concrete AWS maturity step after the dev-to-staging contract
2. make staging a stricter delivery model, not just another environment name

## Core Position

If staging is going to mean anything, it must introduce stronger promotion and rollback discipline.

The key improvement is not:

1. another cluster

The key improvement is:

1. explicit movement of reviewed digests
2. explicit rollback to a known-good GitOps revision
3. explicit post-promotion verification

## Why This Is The Right Next Step

The current dev model is intentionally direct:

1. build on `develop`
2. promote into the dev GitOps overlay
3. let Argo reconcile

That is good for dev, but it is not yet a staging discipline.

The next maturity layer is to preserve the same GitOps model while tightening:

1. promotion intent
2. rollback clarity
3. environment boundaries

## What Staging Must Add

Staging should require:

1. explicit digest selection
2. a promotion record
3. a known rollback target
4. verification before acceptance

That is a much better principal-level progression than inventing new topology first.

## Why This Fits The Roadmap

The roadmap now says:

1. mature single-cluster AWS
2. strengthen the environment model
3. only then justify multi-cluster

This promotion-and-rollback model is the practical center of Stage B.

## Next Step

The next implementation work should define:

1. the staging overlay path
2. the staging Argo app path
3. the promotion record format
4. the rollback checklist

Only after those are explicit should the project decide whether it is time to actually stand up a staging environment.
