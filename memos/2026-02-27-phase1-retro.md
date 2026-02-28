# Phase 1 Retro: Vertical Slice Completed

Date: 2026-02-27
Status: Complete

## Executive Summary

Phase 1 achieved its core goal:

- PulseCart now supports a real local end-to-end order flow.
- The system proves distributed boundaries, producer and consumer idempotency, async processing, and basic observability.
- The local stack is repeatable through a single `make e2e` command.

This is no longer a skeleton. It is a working vertical slice with reliability patterns that can be evolved into platform-grade deployment in Phase 2.

## What Was Delivered

1. API Gateway
   - Proxies `POST /v1/orders` to `orders`
   - Enforces timeout behavior
   - Propagates `X-Request-Id`

2. Orders Service
   - Validates input
   - Requires `Idempotency-Key`
   - Uses Redis to reject duplicate create requests
   - Persists `orders` and `order_items` in Postgres
   - Publishes `orders.created.v1` to NATS

3. Worker Service
   - Subscribes to `orders.created.v1`
   - Uses Redis to suppress duplicate event side effects
   - Calls notifications via HTTP

4. Notifications Service
   - Accepts `POST /v1/notify`
   - Validates payload
   - Logs accepted notifications

5. Observability
   - `/metrics` endpoints for gateway and orders
   - dedicated metrics server for worker
   - starter dashboard, alert stubs, and runbook in `triad-portfolio/reliability`
   - `request_id` now propagates through the async flow

6. Verification
   - Automated service tests across Phase 1 services
   - CI workflow for service tests
   - Manual GitHub workflow for Docker-backed E2E
   - Local `make e2e` script validating:
     - first request `201`
     - duplicate request `409`
     - DB persistence
     - single notification side effect
     - metrics endpoint readiness

## What Broke During Delivery

These failures were useful and should be remembered.

1. Reused idempotency keys caused false duplicate failures
   - Root cause: fixed test key in E2E script
   - Fix: generate unique request and idempotency IDs per run

2. Old local processes caused false-green and false-red runs
   - Root cause: stale listeners on `8080`, `8081`, `8082`
   - Fix: E2E now force-kills stale listeners before startup

3. E2E initially trusted a brittle worker log line
   - Root cause: success criteria relied on a specific timing-sensitive log message
   - Fix: notifications side effect became the primary success signal; worker log is now informational

4. Orders schema check failed because requests hit an older process
   - Root cause: port collision masked the new service version
   - Fix: fail-fast process checks and stale-process cleanup

5. `pgxpool` introduced an unnecessary dependency resolution issue
   - Root cause: transitive module checksum unavailable in restricted environment
   - Fix: switched to `pgx.Conn` for simpler offline compatibility

## What Was Learned

1. Reliability starts with deterministic local workflows
   - Before cloud infrastructure, local startup, cleanup, and repeatability matter.

2. Idempotency must exist on both sides of the queue
   - Producer-only protection is insufficient.
   - Consumer-side replay suppression is non-optional in at-least-once systems.

3. Observability should arrive early, but not at full platform weight
   - Simple counters and durations were enough to validate behavior now.
   - Full telemetry stack would have slowed the learning curve too early.

4. Testability improves design quality
   - Small refactors for injected clients, stores, and notifiers made the system easier to reason about.

5. Infrastructure modules need guardrails, not just reusable code
   - The first reusable VPC module exposed a real footgun: if public and private subnet sizing are independently configurable, Terraform can generate overlapping CIDRs.
   - The fix was not just code; the learning is that reusable infrastructure needs input validation that prevents invalid topologies before apply.
   - In early phases, narrower module contracts are better than flexible-but-unsafe interfaces.

## Remaining Gaps

Phase 1 is strong, but not finished in a production sense.

1. Orders write + event publish is not yet transactional
   - A transactional outbox is still needed.

2. Notifications is still a log sink
   - No real provider abstraction yet.

3. Metrics are intentionally simple
   - No histograms, labels strategy, or full scrape stack yet.

4. Auth and inventory are still not integrated
   - The vertical slice proves the pattern, not full product breadth.

## Phase 2 Handoff

Recommended Phase 2 focus:

1. Start in AWS only
   - Use `triad-landing-zones` for foundational networking and identity guardrails.

2. Stand up EKS and GitOps
   - Use `triad-kubernetes-platform` for cluster add-ons and ArgoCD app-of-apps.

3. Containerize and deploy current vertical slice first
   - Do not expand business scope before platform deployment is stable.

4. Preserve current local guarantees in Kubernetes
   - Health checks
   - E2E smoke
   - metrics reachability
   - duplicate suppression behavior

5. Carry infrastructure lessons forward into platform work
   - Keep subnet math explicit and validated.
   - Prefer constrained Terraform module inputs until the network model is mature.

## Bottom Line

Phase 1 proved the system is real.

You now have:
- a functioning distributed vertical slice
- repeatable local validation
- documented reliability patterns
- a clear bridge into platform deployment

That is the correct base for Phase 2.
