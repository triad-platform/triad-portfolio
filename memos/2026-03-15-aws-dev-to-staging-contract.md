# AWS Dev To Staging Contract

Date:

1. March 15, 2026

Purpose:

1. define what changes between dev and staging before the project adds more topology
2. make the next AWS maturity step about environment discipline, not cluster count

## Why This Matters

Once the AWS dev baseline is mature enough, the next useful question is not immediately:

1. should we add more clusters

The next useful question is:

1. what would have to become stricter before the same platform model deserves a real staging environment

That is a better principal-level question because it tests whether the current system can survive stronger operating boundaries before multiplying topology.

## Core Position

Staging should not be a random second environment.

It should be:

1. the same proven AWS operating model
2. with stricter promotion
3. stricter rollback discipline
4. clearer secret boundaries
5. clearer alerting expectations

## What This Changes In The Project

The next maturity phase now has a cleaner shape:

1. preserve the single-cluster AWS reference model
2. strengthen environment contracts
3. only revisit multi-cluster after environment discipline is clearer

## Why This Is Better Than Jumping To Multi-Cluster

If the platform cannot clearly explain:

1. how dev promotions differ from staging promotions
2. how rollback works in staging
3. how secret and alert boundaries get tighter

then adding more clusters would mostly multiply ambiguity, not reduce it.

## Next Step

The next meaningful implementation work should define:

1. staging GitOps promotion path
2. staging rollback path
3. staging-specific secret and alert boundaries

That is the next maturity layer the roadmap calls for.
