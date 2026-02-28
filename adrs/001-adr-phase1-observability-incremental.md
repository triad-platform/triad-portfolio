# ADR-001: Incremental Observability Strategy for Phase 1

Status: Accepted
Date: 2026-02-27

## Context

PulseCart Phase 1 must prove core distributed-system behavior first:
- request routing through gateway
- idempotent order creation
- async event processing and side-effect handling

At this stage, the project goal is dual-purpose:
1. ship a working vertical slice
2. learn reliability and observability concepts in the right order

A full observability stack (Prometheus Operator, histogram-heavy instrumentation, OpenTelemetry collector/traces, alert routing, long-term storage) adds significant setup and operational complexity before core behavior is stable.

## Decision

Use an incremental observability baseline in Phase 1:

1. Implement lightweight, Prometheus-compatible `/metrics` endpoints in-app.
2. Track only high-signal metrics needed for current risks:
   - request volume
   - error counts
   - duplicate/idempotency behavior
   - coarse latency summaries
3. Add a starter dashboard and starter alert rules in `triad-portfolio/reliability`.
4. Add an incident runbook tied directly to these metrics and alerts.

Defer advanced telemetry stack work (full histogram strategy, distributed tracing backend, and production-grade alert routing) to later phases when platform complexity justifies it.

## Alternatives

1. Full production observability stack immediately
   - Prometheus + Grafana + Alertmanager + OTel collector + tracing backend in Phase 1.
2. No metrics until Phase 4
   - Focus only on feature delivery now and add observability later.
3. Vendor-specific managed observability now
   - Adopt cloud-native telemetry early (CloudWatch/Azure Monitor-first).

## Tradeoffs

### Why this decision is better now

- Faster delivery of core system correctness.
- Lower cognitive load while learning service boundaries and failure modes.
- Still enough telemetry to detect regressions and validate idempotency + async flow.
- Keeps interfaces Prometheus-compatible, so migration to full stack is low-friction.

### What we give up short-term

- No percentile latency precision from true histogram buckets yet.
- No end-to-end distributed traces through a trace backend yet.
- Limited dimensionality compared to mature labels/tags strategy.

## Consequences

### Immediate effects

- Observability is present now, not deferred.
- E2E checks can validate metrics endpoint availability.
- Reliability artifacts (alerts/dashboard/runbook) are versioned with the code.

### Future evolution path

Phase 2-3:
- Move metrics scraping and dashboard hosting into Kubernetes/GitOps.
- Add service-level histogram metrics and tighter alert thresholds.

Phase 4:
- Formalize SLOs/SLIs on top of stabilized telemetry.
- Add burn-rate alerts and incident response drills.

Phase 5+:
- Add distributed tracing backend and correlation across clouds.

This sequence intentionally prioritizes system correctness first, then measurement depth, then operational rigor at scale.
