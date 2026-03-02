# Reliability

SLOs, error budgets, incident response, postmortems, and runbooks.

## Starter Artifacts

- `dashboards/pulsecart-phase1-overview.json`
  - Grafana starter dashboard for Phase 1 local/service metrics.
- `alerts/pulsecart-phase1-alerts.yml`
  - Prometheus alert rule stubs aligned to current metrics.
- `runbook-phase1.md`
  - Triage and mitigation playbook for the Phase 1 alert set.

## Notes

- Metric names match current `triad-app` instrumentation in:
  - `api-gateway`
  - `orders`
  - `worker`
- The dev cluster now mirrors these starter artifacts into the live observability baseline in `triad-kubernetes-platform/platform/observability`.
- Thresholds are intentionally conservative starter defaults and should be tuned from observed baseline behavior.
- Strategy rationale is documented in:
  - `../adrs/001-adr-phase1-observability-incremental.md`
  - `../adrs/002-adr-idempotency-producer-consumer.md`
