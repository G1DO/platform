# Problem Statement

## Context

This platform is built on reliability lessons from operating **Ahoy**, a real DTC + B2B sauce-and-condiment business in MENA. Ahoy runs as a single-host application on AWS EC2 with CloudWatch-only observability — Paymob-integrated checkout, FIFO inventory with expiry batches, multi-warehouse fulfilment, double-entry accounting, recipe-driven manufacturing. The business is operational and continues to operate; this platform is **not** being deployed onto Ahoy. It codifies the reliability practices Ahoy needed but didn't have, and demonstrates them against a purpose-built workload that exhibits the same failure-mode classes ([ADR-006](adr/006-workload-go-shim.md)).

The numbers anchoring the platform's targets are the projection Ahoy was sizing for: ~25 engineers, six cities across Egypt, Saudi Arabia, UAE, and Jordan, ~5k orders/day at weekend-grocery and flash-promo peaks, B2B onboarding with restaurant groups, hotels, and specialty retail. These are the scale-pressures that made each missing reliability practice concrete — not hypothetical.

## The problem

Reliability practice was missing where it would hurt at scale.

- **No observability beyond logs.** No metrics, no traces, no SLO tracking, no burn-rate alerts. Incidents surfaced via customer messages, not signals.
- **Manual deploys.** SSM into EC2, `docker-compose pull`, migrations run by hand. Mean deploy time ~25 minutes; no automated rollback.
- **No on-call rotation.** One engineer was the system; detection depended on whoever read WhatsApp first.
- **Single point of failure.** One EC2 instance hosted every service. Host-level failure = full outage. No autoscaling, no graceful drain, no equivalent of a PodDisruptionBudget.
- **Fragile payment-webhook handling.** A Paymob webhook backlog during a flash-promo Friday left ~200 orders stuck in `PENDING` for 90 minutes. Recovery was manual webhook replay and order-by-order ledger reconciliation. At 3× projected volume, that becomes a multi-hour revenue-and-refund incident.

Each of these reads as a class of failure that any production system at this shape encounters; the Ahoy incident is the specific instance that made the cost concrete enough to design against.

## Who felt it

On-call engineers (informal — typically the founder; on the projected trajectory, a small platform team inside a 25-person engineering org), customer-operations and finance staff handling stuck-order tickets and manually reconciling payments against the ledger, B2B accounts in early onboarding whose SLAs depend on inventory-API reliability, and end customers who saw `order failed` mid-checkout or charges that didn't resolve into a confirmed order.

## Why now

The pressure was concrete: Series A closed in Q1 2026, headcount growing toward 25 engineers, operations expanding into six cities over 12 months. The Paymob near-miss made the cost real — at 3× projected traffic the current setup would not absorb a peak event without a multi-hour revenue incident. The reliability work was scoped as one quarter of platform engineering invested before scale forced firefighting.

This repo codifies that work as a defensible, demonstrable platform — one whose decisions are written down, whose self-healing properties are testable under chaos, and whose workload ([ADR-006](adr/006-workload-go-shim.md)) is a Go shim built specifically to exhibit the failure modes the platform protects against. Shipping it as a portable artifact, decoupled from the original employer's codebase, makes the reliability story sharable, defensible in interviews, and reusable as a baseline for the next system at this shape.

## Success criteria

The platform is "done" when each of these is demonstrably true on the cluster, with evidence captured in dashboards and postmortems.

1. **MTTR under 5 minutes** for service-level failures (baseline from the Paymob incident: 90+ minutes; manual recovery for everything else).
2. **99.5% availability** on the order-placement critical path over rolling 30 days — measured as `POST /api/v1/orders` returning 2xx within 400ms at ingress against the Go workload ([slos.md CUJ-1](slos.md)).
3. **p99 latency under 400ms** on order placement under load matching the projected 5k orders/day peak.
4. **Payment-webhook-to-ledger reconciliation completed within 60s of provider callback, 99.9% of the time**, with zero manual replay required for transient gateway failures ([slos.md CUJ-2](slos.md)).
5. **Self-service deploys under 10 minutes merge-to-production**, with automated rollback on SLO regression during progressive rollout.

## Non-goals

1. **Multi-region active-active.** Single MENA region matches the original business footprint; cross-region replication is overhead without business return at projected scale. Revisit at 50k orders/day.
2. **Multi-tenant cluster isolation.** One product, one trust boundary; namespace-level RBAC and NetworkPolicies are sufficient. No per-tenant compliance to defend.
3. **Operator-managed database.** The workload uses managed Postgres (RDS in the target environment; equivalent local Postgres in dev). Migrating to CloudNativePG or a self-hosted operator would add operational risk without addressing the trigger event.
4. **Cost optimization as a primary goal.** Right-sizing, spot instances, and autoscaling are in scope; a dedicated FinOps workstream is not.
5. **Service mesh.** mTLS via cert-manager and NetworkPolicies cover the threat model at the workload's service count. Istio/Linkerd deferred unless service count crosses ~15.
