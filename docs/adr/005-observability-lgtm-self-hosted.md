# ADR-005: Observability stack — self-hosted Prometheus + Loki + Tempo + Grafana (LGTM)

## Status

Accepted — 2026-05-28.

## Context

The platform's observability stack is load-bearing for the SLO commitments in [slos.md](../slos.md) — burn-rate alerts on CUJ-1 (order placement) and CUJ-2 (payment-webhook-to-ledger reconciliation) are measured by this stack, and Phase 5's automated rollback on SLO regression queries the same metrics store. Current Ahoy's CloudWatch-only setup is explicitly named as inadequate in [problem-statement.md](../problem-statement.md).

The choice here is not "which backend" in isolation; it is whether to operate the stack ourselves (self-hosted OSS) or pay a SaaS vendor (Datadog, Honeycomb, Grafana Cloud, New Relic). The cost question is real at projected Series-A scale; the lock-in question is real for a CV-piece project.

Constraints driving this decision:

- **SLO measurement and burn-rate alerting** ([slos.md](../slos.md)): metrics with reliable retention, queryable for multi-window multi-burn-rate rules. CUJs are measured at the ingress, not the application — the stack must scrape there.
- **Logs with trace IDs** for incident-correlation workflows ("from a paged alert to the offending request in two clicks").
- **Distributed tracing** across FastAPI → Celery → Postgres for CUJ-2 reconciliation debugging.
- **PII safety** ([threat-model.md §B5:I](../threat-model.md)) — telemetry collection cannot become a new PII exfiltration path.
- **kind-compatible** ([ADR-001](001-local-cluster-kind.md)) — local dev runs the same stack.
- **Cost.** This is a CV-piece project. SaaS observability at Series-A volumes is real money ($30–60k/year at this scale, depending on vendor and event volume). The platform thesis is "reliability with proof," not "reliability via vendor."

## Decision

We will use a **self-hosted LGTM stack** in a dedicated `observability` namespace:

- **Prometheus** for metrics. Single instance sufficient at projected scale (5k orders/day); Mimir deferred until horizontal scaling is actually needed.
- **Loki** for logs.
- **Tempo** for traces.
- **Grafana** as the unified UI.
- **OpenTelemetry Collector** as the single intake — applications emit OTLP, the collector routes to the three backends.
- **Alertmanager** for alert routing (paging via PagerDuty per [slos.md](../slos.md) on-call section).

All five components are CNCF projects; the four backend components are designed to interoperate within the Grafana ecosystem.

## Alternatives considered

- **Datadog.** Best-in-class commercial observability platform; mature, polished, well-staffed. Rejected on **cost + lock-in**. At 5k orders/day with full metrics + logs + APM, list pricing comes out to a material five-figure annual spend that the platform thesis cannot absorb. Datadog's advanced features (Watchdog, anomaly detection) use proprietary query semantics that don't transfer if migration is forced later. Local dev would require either a separate Datadog account or a divergent local stack — friction.
- **Honeycomb.** Best-in-class for high-cardinality observability and traces. Phenomenal product, particularly for "the question I didn't know to ask before the incident." Rejected as **primary** for the same SaaS-cost reason as Datadog. Worth revisiting as a **complement** if traces become the central debugging story and the OSS Tempo experience proves insufficient; introducing two observability backends doubles operational surface, so this is a deliberate "not yet."
- **New Relic / Dynatrace.** Commercial, lock-in, expensive. No compelling differentiator for a Kubernetes-native platform.
- **Grafana Cloud (managed LGTM).** Identical query languages to self-hosted LGTM → **zero lock-in**. Rejected as primary on cost at scale, but **available as a fallback**. The same Grafana dashboards, the same PromQL/LogQL/TraceQL — the migration cost from self-hosted LGTM to Grafana Cloud is "change the remote-write target and the data source URL." This is the platform's escape hatch if operational burden becomes the bottleneck.
- **AWS-native (CloudWatch + X-Ray).** Rejected. CloudWatch is the current Ahoy state, explicitly called out in [problem-statement.md](../problem-statement.md) as inadequate (no metrics beyond log-based, no SLO tracking, no burn-rate alerting). X-Ray is acceptable but the integration is AWS-cloud-specific in ways that contradict the Kubernetes-native platform thesis and the kind-local-dev constraint.
- **Mimir instead of Prometheus.** Rejected for Phase 1. Mimir is for horizontal scaling beyond a single Prometheus's capacity. At 5k orders/day with reasonable cardinality, a single Prometheus handles the workload with headroom. Adding Mimir adds operational complexity for a problem we don't have. Re-evaluate at 10× volume.

## Consequences

**Positive.**

- **Zero per-event SaaS cost.** The platform's operating cost is infrastructure-only — Postgres for retention, S3 (or compatible) for cold storage. Real five-figure annual savings vs SaaS at projected scale.
- **Full local-cluster reproducibility.** The same OTel + Prom + Loki + Tempo + Grafana stack runs in kind. Local debugging matches production debugging exactly — no "this works locally but the metric is named differently in Datadog" surprises.
- **Skill transfer.** PromQL, LogQL, TraceQL, OTel are the industry-standard interfaces. Operational experience running this stack is portable to almost any future employer.
- **OTel Collector as the intake** decouples app instrumentation from backend choice. If we later swap Loki for Grafana Cloud Logs, only the collector's exporter config changes — app code is untouched. This is the same decoupling pattern as GitOps and Cosign keyless, applied to telemetry.
- **Migration path stays open.** Identical query languages to Grafana Cloud mean "move to managed" is a config change, not a rewrite. The decision is reversible at low cost.

**Negative.**

- **Operational burden owned.** Backups, retention tuning, scaling, security patching, upgrades — all on us. Loki at scale has known cost pitfalls (label cardinality explodes ingest cost, undertuned chunk sizes hurt query latency). Tempo retention is expensive on object storage if not lifecycle-managed.
- **More moving parts than a SaaS install.** Six components to deploy and monitor. The observability stack's own observability is itself a problem ("who watches the watcher" — Prometheus monitors its own scrape success).
- **PII risk inherent in telemetry.** A wide-open OTel ingest accepts whatever the app sends, including PII inadvertently included in log lines or span attributes. Closing this is non-optional ([threat-model.md §B5:I](../threat-model.md)) — see the follow-up work.
- **No commercial support contract.** Bugs and outages are debugged from OSS issue trackers and community Slack. Acceptable at this scale and project shape; might not be at higher reliability tiers.

**Follow-up work this commits us to.**

- Phase 4 ships the full stack via Helm charts (or kube-prometheus-stack / loki-stack / tempo distributions), version-pinned in the config repo.
- **PII-scrubbing pipeline in OTel Collector** — `attributes` processor with explicit deny-list for known PII fields (email, phone, address, lat/lon per [threat-model.md §A1](../threat-model.md)). Closes [§B5:I](../threat-model.md).
- Retention policy per signal:
  - Metrics: 30 days hot, 1 year downsampled (Prometheus + recording rules).
  - Logs: 14 days hot, 90 days cold on S3 (Loki object-store backend).
  - Traces: 7 days hot, 30 days cold on S3.
  These are defensible defaults at our scale; document the reasoning in the runbook so a future change has context.
- Burn-rate alerts from [slos.md](../slos.md) (CUJ-1, CUJ-2) live as Prometheus recording rules + Alertmanager routes.
- Trace IDs propagated end-to-end and included in every log line (OTel SDK + structured logging). The debug-from-page workflow depends on this.
- Documented migration path to Grafana Cloud (config-only) — preserves the reversibility argument used to justify self-hosting.
- Audit-log export to S3 with Object Lock ([threat-model.md §B2:R, §B8:R](../threat-model.md)) is a *separate* concern from this observability stack — observability is for operational telemetry, not for tamper-proof audit trails. Conflating the two is a common mistake.
