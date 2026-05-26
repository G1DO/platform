# Problem Statement

## Context

Ahoy is a DTC + B2B sauce and condiment company in MENA, running a FastAPI backend, two Next.js frontends (a public storefront and an operations dashboard), Postgres, Redis, and Celery workers on a single AWS EC2 instance with CloudWatch-only observability. The business is operational — Paymob-integrated checkout, FIFO inventory with expiry batches, multi-warehouse fulfilment, double-entry accounting, and recipe-driven manufacturing. This document treats Ahoy as the use-case for an 18-month forward-looking platform design, anchored on post-Series-A scale: ~25 engineers, six cities across Egypt, Saudi Arabia, UAE, and Jordan, ~5k orders/day at weekend-grocery and flash-promo peaks, and B2B onboarding underway with restaurant groups, hotels, and specialty retail.

## The problem

Reliability practice is missing where it will hurt at scale.

- **No observability beyond logs.** No metrics, no traces, no SLO tracking, no burn-rate alerts. Incidents surface via customer messages, not signals.
- **Manual deploys.** SSM into EC2, `docker-compose pull`, run migrations by hand. Mean deploy time ~25 minutes; no automated rollback.
- **No on-call rotation.** One engineer is the system; detection depends on whoever reads WhatsApp first.
- **Single point of failure.** One EC2 instance hosts every service. Host-level failure = full outage. No autoscaling, no graceful drain, no equivalent of a PodDisruptionBudget.
- **Fragile payment-webhook handling.** Six weeks ago, a Paymob webhook backlog during a flash-promo Friday left ~200 orders stuck in `PENDING` for 90 minutes. Recovery was manual webhook replay and order-by-order ledger reconciliation. At 3× projected volume, that becomes a multi-hour revenue and refund incident.

## Who feels it

On-call engineers (informal — currently the founder; soon a small platform team inside the 25-person engineering org), customer operations and finance staff who handle stuck-order tickets and manually reconcile payments against the ledger, B2B accounts in early onboarding whose SLAs depend on inventory API reliability, and end customers who see `order failed` mid-checkout or charges that don't resolve into a confirmed order.

## Why now

Series A closed in Q1 2026; headcount grows toward 25 engineers and operations expand into six cities over the next 12 months. The Paymob near-miss made the cost concrete: the current setup will not absorb a 3× traffic event. Investing one quarter of platform engineering work *before* scale forces firefighting is materially cheaper than rebuilding under incident load. Leadership has agreed to ring-fence the quarter.

## Success criteria

1. **MTTR under 5 minutes** for service-level failures (baseline: 90+ minutes for the Paymob incident; manual recovery for everything else).
2. **99.5% availability** on the order-placement critical path over rolling 30 days — measured as `POST /orders` returning 2xx within 400ms at ingress.
3. **p99 latency under 400ms** on order placement at projected 5k orders/day peak.
4. **Payment-webhook-to-ledger reconciliation completed within 60s of provider callback, 99.9% of the time**, with zero manual replay required for transient gateway failures.
5. **Self-service deploys under 10 minutes merge-to-production**, with automated rollback on SLO regression during progressive rollout.

## Non-goals

1. **Multi-region active-active.** Single MENA region matches the business footprint; cross-region replication is overhead without business return at projected scale. Revisit at 50k orders/day.
2. **Multi-tenant cluster isolation.** One product, one trust boundary; namespace-level RBAC and NetworkPolicies are sufficient. No per-tenant compliance to defend.
3. **Replacing managed Postgres.** RDS Postgres stays. Migrating to CloudNativePG or a self-hosted operator would add operational risk without addressing the trigger event.
4. **Cost optimization as a primary goal.** Right-sizing, spot instances, and autoscaling are in scope; a dedicated FinOps workstream is not.
5. **Service mesh.** mTLS via cert-manager and NetworkPolicies cover the threat model at current service count. Istio/Linkerd deferred unless the service count crosses ~15.
