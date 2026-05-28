# Service Level Objectives

This document defines the SLIs and SLOs that the rest of the platform design protects. Targets are anchored on the [problem statement](problem-statement.md) and apply at the projected post-Series-A scale that motivated the platform: ~5k orders/day at peak, six MENA cities, ~25 engineers. The workload exhibiting these SLO surfaces is defined in [ADR-006](adr/006-workload-go-shim.md).

## Critical user journeys

Two CUJs are in scope for Phase 1. Both have direct revenue impact and span multiple services, so per-service SLOs would not capture the failure modes that matter.

- **CUJ-1: Order placement** — from `POST /api/v1/orders` at the ingress through validation, payment-intent creation against the payments-provider mock, and order persistence, ending at the confirmation response to the client. Handled by the `orderd` service.
- **CUJ-2: Payment-webhook-to-ledger reconciliation** — from receipt of a payments-provider webhook (modeled on Paymob, the provider whose backlog incident motivated this CUJ) at the backend webhook endpoint to a committed ledger record reflecting the payment. The synchronous receipt happens in `orderd`; the asynchronous commit happens in `reconcilerd` after the broker dequeue.

Other workload paths (operator fulfillment, customer order tracking) are deliberately not under SLO in Phase 1. They matter at real product scale but don't burn revenue per minute the way the two CUJs above do, and the workload doesn't model them per [ADR-006 §Out of scope](adr/006-workload-go-shim.md). They will be re-evaluated after a quarter of production data.

## SLIs and SLOs

### CUJ-1: Order placement

**SLI (availability):** the ratio of `POST /api/v1/orders` requests at the ingress that return HTTP 2xx within 400ms, excluding 4xx client errors (invalid payload, auth failures, idempotency-key conflicts), measured in 1-minute windows.

**SLI (latency):** the p99 of `POST /api/v1/orders` end-to-end latency at the ingress, measured per 1-minute window.

**SLO:**
- 99.5% availability over rolling 30 days.
- p99 latency under 400ms over rolling 30 days.

**Error budget:** 0.5% × 30 days = **3.6 hours/month** of allowed budget burn on availability.

**Rationale:**
- *Why 99.5% and not 99.9%.* At projected peak (~6 orders/min × $25 AOV ≈ $150 revenue/peak-minute), 99.5% gives 3.6h budget; if it all landed at peak the loss would be ~$32k/month, but burn is roughly proportional to traffic and most of it lands off-peak (peak is 120h of 720h/month). Realistic expected loss: ~$5–10k/month. Reaching 99.9% would shrink the budget to 43 min/month, forcing conservative deploys and contradicting [success criterion #5](problem-statement.md#success-criteria) (self-service deploys under 10 min). The marginal engineering cost of 99.9% (multi-region redundancy, conservative release process, larger on-call burden — estimated $200–500k/year-equivalent) exceeds the marginal downtime cost it would prevent.
- *Why 400ms p99.* E-commerce checkout abandonment research consistently shows perceived-slowness thresholds in the 300–500ms range; 400ms also leaves headroom for the Paymob redirect handshake within a total user-perceived budget of ~1s.
- *Why measure at ingress.* The user's experience includes TLS termination, ingress routing, and the ingress controller itself. Measuring at the application would miss these failure modes.
- *Why exclude 4xx.* These are client errors, not service failures. Including them would distort the signal and incentivize hiding real failures behind 5xx-to-4xx conversion.
- *Re-evaluation trigger.* Tighten to 99.9% at 50k orders/day or after any incident that exhausts >50% of the monthly budget.

### CUJ-2: Payment-webhook-to-ledger reconciliation

**SLI (timeliness):** the ratio of payments-provider webhook events whose corresponding ledger entry is committed within 60 seconds of webhook receipt at the backend webhook endpoint. Measured per 1-minute window across the queue-processing pipeline (`orderd` webhook handler → broker enqueue → `reconcilerd` dequeue → DB transaction commit).

**SLO:** 99.9% over rolling 30 days.

**Error budget:** 0.1% × 30 days = **~43 minutes/month** of allowed budget burn.

**Rationale:**
- *Why tighter (99.9%) than CUJ-1.* Each unreconciled webhook is a customer who has paid but sees `PENDING`. The cost asymmetry is severe: the customer can't safely retry payment (risk of double-charge), every stuck webhook generates a support ticket, and the ledger reconciliation has to be done by hand against the gateway. The Paymob near-miss documented in the problem statement was exactly this: 200 stuck webhooks × ~$25 ≈ $5k stuck revenue plus 200 support escalations in 90 minutes. The Go workload's `orderd` → broker → `reconcilerd` path is the structural analogue built so the SLO is testable.
- *Why 60 seconds.* Below 30s leaves no headroom for queue backoff under normal load spikes; above 5 minutes a customer refreshes and assumes the page is broken, then contacts support — generating the load the SLO is meant to prevent. 60s matches the typical "page must be wrong" customer threshold while leaving room for worker retry budget.
- *Why 99.9% is feasible.* Webhook ingestion is stateless and idempotent (transaction-id-keyed on the gateway side). The dominant failure mode is queue backlog, not compute — solvable with worker autoscaling and a dead-letter queue. 99.9% is achievable with attention to those mechanisms; 99.99% would require synchronous in-request ledger commits, which defeats the queue's purpose.
- *Re-evaluation trigger.* Re-evaluate after one quarter of production data; tighten only if budget consistently runs under 25%.

## Alerting policy

Burn-rate-based, following the Google SRE workbook multi-window multi-burn-rate pattern. Symptom-driven, not cause-driven: no CPU/memory threshold pages, no "queue depth > X" pages — only pages on SLO burn.

| Condition | Channel | Routing |
|---|---|---|
| 1h burn rate > 14.4× **and** 6h burn rate > 6× | **PAGE** | Primary backend on-call |
| 6h burn rate > 6× **and** 24h burn rate > 1× | Ticket | Backend team queue, business-hours review |
| Slower drifts | Dashboard only | Weekly SLO review |

Both windows must be over threshold simultaneously — this is what prevents false positives from short traffic spikes (1h alone fires too often) and what catches persistent slow degradations (24h alone fires too late).

**On-call routing:**
- CUJ-1 → backend on-call (weekly rotation, ~5 engineers).
- CUJ-2 → backend on-call, with platform on-call as escalation when the failure is in queue infrastructure (`reconcilerd`, Redis broker).
- Escalation: primary doesn't ack in 5 min → secondary; secondary doesn't ack in 10 min → engineering manager.
- Tool: PagerDuty (covered by a separate ADR).

**Every page must have a runbook.** Pages without runbooks are tickets. Example for CUJ-2: (1) check payments-provider status page, (2) check broker queue depth in Grafana, (3) check dead-letter queue, (4) `replay-webhooks.sh --since 1h` if backlog confirmed, (5) escalate to platform on-call if `reconcilerd` is unresponsive.

**Error budget policy:**
- < 50% budget remaining → no risky deploys or infra changes during peak windows.
- < 25% budget remaining → feature freeze on the affected CUJ; reliability work prioritized.
- 0% budget remaining → full feature freeze on the affected CUJ; post-budget recovery plan required before resuming.

## Review cadence

- **Quarterly:** review every SLO, baseline numbers, budget consumption, and alerting noise (pages-per-week-per-engineer; aim for < 2). Adjust targets only with data.
- **Post-incident:** any incident consuming >25% of monthly budget triggers a same-week review — does the SLO still match user pain, does the runbook need updating, does the alerting threshold need re-tuning.
- **Annual:** review CUJ selection. Has the product shape changed enough that new CUJs deserve SLOs, or existing ones should be retired?
