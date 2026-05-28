# ADR-006: Workload — Go shim modeling Ahoy failure modes

## Status

Accepted — 2026-05-28.

## Context

[problem-statement.md](../problem-statement.md) frames Ahoy as the real workload whose operational pain motivates this platform. Ahoy is closed-source production code; the platform repo cannot contain it. The architecture, SLOs, and threat-model are nonetheless grounded in Ahoy's actual failure-mode classes — the Paymob webhook backlog ([threat-model.md §B2:D](../threat-model.md)), the connection-pool exhaustion shape, the cache-vs-DB consistency story under Redis outage, the async webhook→queue→ledger reconciliation pattern ([slos.md CUJ-2](../slos.md)). These are not abstract concerns; they are the incidents the platform is built to absorb.

What the platform needs in-repo is a workload that **exhibits the same failure-mode classes** under load and during chaos experiments. It does not need to be Ahoy, and it does not need to model Ahoy's product depth (FIFO inventory with expiry batches, recipe-driven manufacturing, multi-warehouse fulfilment, double-entry accounting). Reproducing those would dilute the platform's deliverable into "Ahoy rebuild" — which is neither what the user can ship nor what a hiring manager wants to read.

This ADR picks the workload's language, topology, and fidelity contract. It is a Phase 1 decision because every artifact downstream — Helm charts ([ADR-002](002-gitops-argo-cd.md)), Cosign signatures ([ADR-003](003-image-signing-cosign-keyless.md)), Kyverno policies ([ADR-004](004-admission-policy-kyverno.md)), OTel instrumentation ([ADR-005](005-observability-lgtm-self-hosted.md)), Argo Rollouts canary analysis — depends on knowing what shape of workload is being deployed, signed, admitted, observed, and rolled out.

Constraints driving this decision:

- **Failure-mode fidelity.** The workload must exhibit the incidents from [threat-model.md](../threat-model.md) and [slos.md](../slos.md) — most importantly CUJ-2's payment-webhook reconciliation path and the Paymob-incident-class queue-depth-explosion shape ([threat-model.md §B2:D](../threat-model.md)). If the chaos experiments in Phase 6 can't reproduce the incident, the platform's claim of "self-healing for this class of failure" is untested.
- **Shareable.** The repo is the CV deliverable. The workload code must be publishable without IP friction.
- **One platform engineer maintaining this.** Workload complexity that requires more attention than the platform itself defeats the project's purpose. Two services is the floor (an async path needs a worker); three is the ceiling at Phase 1.
- **Aligned with the Phase 2 walking-skeleton constraints** ([ADR-001](001-local-cluster-kind.md)) — must run in kind, must build under 5 minutes, must instrument cleanly via the OTel SDK chosen in [ADR-005](005-observability-lgtm-self-hosted.md).
- **Hiring-signal-positive language choice.** SRE/platform job descriptions over the last 24 months show Go appearing more often than Python in cluster-resident workloads; the Go ecosystem around platform tooling (pgx, distroless static, otel-go, the kubernetes/client-go family) is materially better-fit for what the platform exercises.

## Decision

We will build the workload as **two Go services** in a separate `ahoy-app` repository ([ADR-002](002-gitops-argo-cd.md) two-repo pattern), modeling Ahoy's failure-mode classes without cloning its product surface:

- **`orderd`** — HTTP API service. Endpoints: `POST /api/v1/orders` (CUJ-1 entry), `POST /api/v1/webhooks/paymob` (CUJ-2 entry), `GET /healthz` (liveness), `GET /readyz` (readiness, checks DB + broker), `GET /metrics` (Prometheus scrape).
- **`reconcilerd`** — async worker. Reads from a Redis-backed queue, validates the webhook payload's HMAC, looks up the order, commits a row to a minimal `ledger_entries` table in Postgres. Closes CUJ-2.

Persistence: managed Postgres (RDS in prod, local container in dev) and in-cluster Redis (queue + cache-aside). External: a **payments-provider mock** HTTP service in the same repo, modeling Paymob's retry/timing/HMAC behavior — gives reproducible incident simulations without depending on a real third party.

### Fidelity contract — what the workload must exhibit

The platform's claims of self-healing rest on this contract. If a property below isn't present, the platform's coverage of that failure class isn't demonstrable.

- **Connection-pool exhaustion under bursty load.** pgx pool with `MaxConns`, `MinConns`, `MaxConnIdleTime` exposed via env; tunable to reproduce the Ahoy-class pool starvation under flash-promo load.
- **Async webhook reconciliation with retry/backlog dynamics.** A `reconcilerd` worker that consumes from a bounded Redis queue with per-message attempt counts. Under enqueue pressure, queue depth grows visibly — the exact shape of the Paymob incident.
- **Cache-aside Redis pattern with graceful degradation.** `orderd` reads through Redis on order lookup; on Redis unavailability, falls back to DB-only reads with a tracked counter. CUJ-1 keeps serving; latency rises. This is what makes Redis-kill chaos experiments meaningful.
- **Distinct liveness/readiness probe semantics** (per the discipline in [design.md §5.5](../design.md#55-reliability-mechanisms)). `/healthz` returns 200 if the HTTP server is up — **never** checks dependencies. `/readyz` checks the DB pool, the Redis broker, and the payments-mock reachability — fails the pod out of rotation without restarting it.
- **Graceful SIGTERM with preStop endpoint-propagation buffer.** `orderd` and `reconcilerd` trap SIGTERM, stop accepting new work, drain in-flight, close pools, exit cleanly within `terminationGracePeriodSeconds`. Helm template adds a `preStop` sleep so endpoint-removal propagation precedes drain.
- **Prometheus middleware with route-template labels.** `http_requests_total{route,method,status}` and `http_request_duration_seconds_bucket{route,method}` — labels use the route pattern (`/api/v1/orders`, `/api/v1/webhooks/paymob`), never the raw request path. CUJ-1 SLO recording rule in [design.md §5.4](../design.md#54-observability-stack) reads these metrics directly.
- **OTel-Go trace context propagated into structured JSON logs.** Every log line carries `trace_id` and `span_id`; Grafana's logs-to-traces correlation in [ADR-005](005-observability-lgtm-self-hosted.md) depends on this.
- **HMAC-SHA512 webhook verification with constant-time compare, idempotency by transaction ID, amount validation against stored order.** Mirrors the Ahoy handler defended in [threat-model.md §A2](../threat-model.md); makes the §B2 STRIDE walk land on a real implementation.
- **A `PAYMOB_MODE` env knob** with a `log` value — exactly the Ahoy incident vector ([threat-model.md §B2:S](../threat-model.md)) — that short-circuits HMAC verification. **Deliberately present** so the Kyverno policy `disallow-paymob-log-mode-in-prod` ([ADR-004](004-admission-policy-kyverno.md)) has a real footgun to refuse at admission. The workload uses `PAYMOB_*` env var names throughout because it explicitly models Paymob; this keeps the policy name, the threat-model references, and the workload's env consistent.

### Out of scope — what the workload deliberately does NOT clone from Ahoy

Naming these makes the boundary defensible.

- FIFO inventory with `production_date`/`expiry_date`/`unit_cost` batches.
- Multi-warehouse fulfilment, StockEntry per (location, product, batch).
- Recipe-driven manufacturing, BOMs, production cadence.
- Double-entry accounting beyond a single `ledger_entries(id, transaction_ref, debit, credit, committed_at)` table.
- B2B PriceList per customer, Customer.balance, Customer.credit_limit.
- Real Paymob integration (the mock is sufficient and reproducible).
- Real auth (a stub JWT verifier on `POST /orders` is enough).
- Next.js storefront, operations dashboard. The platform isn't demonstrated against a browser frontend in this workload — CUJ-1 entry is HTTP into `orderd` directly.

These are Ahoy product depth, not platform substrate. Adding any of them to the workload trades platform-deliverable hours for fidelity to a private codebase no one will read.

## Alternatives considered

- **Anonymized Ahoy fork.** Take Ahoy, strip business names, sanitize PII, ship it. Rejected on three grounds: (a) IP risk — even structural patterns (table schemas, internal API conventions, queue topology choices) can leak proprietary design; the user's lawyers, not the user, would make this call. (b) Drift cost — every Ahoy upstream change either has to be reapplied or the fork rots. (c) Language mismatch with this project's planning notes and with the SRE/platform job-market signal — Ahoy is Python; staying in Python means accepting a weaker hiring signal for no compensating gain.
- **Sample e-commerce service in Go (cart, checkout, payment).** Rejected. Loses the bite. A generic order service has generic failure modes; the platform's claim shifts from "absorbs the class of failures I lived through at Ahoy" to "absorbs a textbook order service's failures." The whole point of grounding on a real incident is that real incidents have specific shapes (queue depth explodes during flash-promos, connection pools starve at the same wall-clock window when promos hit, the webhook handler's config-driven HMAC bypass is the actual incident vector the threat-model defends). A generic sample doesn't deliver any of that.
- **Stay in Python with a FastAPI stand-in.** A faithful FastAPI + Celery shim, sized to the fidelity contract above. Considered seriously, rejected on hiring-signal grounds (Go appears more in SRE/platform JDs over the project's audience), ecosystem fit (pgx + distroless static + otel-go is a more cohesive platform-engineering story than asyncpg + slim Python image + opentelemetry-instrumentation-fastapi), and project-internal alignment (the planning notes the user works from were Go-shaped). Would be the right pick if the user were a Python expert preserving a learning curve — they are explicitly using this project to learn Go.
- **Three Go services (split out a separate frontend-for-backend or a dedicated webhook ingestor).** Deferred, not rejected. Starting with two keeps Phase 2 manageable; a third service only earns its place if a CUJ demands it (e.g., if the webhook ingestor needs different scaling characteristics from the order API). The two-service split (API + worker) is the minimum that gives canary-rollout independence between the synchronous and asynchronous paths.
- **Single Go service with an internal goroutine worker (no broker).** Rejected. Loses the queue-shape failure mode that CUJ-2 measures and the Paymob incident embodies. An internal goroutine is a single process; a separate worker over a Redis broker reproduces the actual operational topology — queue depth metric, worker autoscaling on broker depth, dead-letter queue, independent restart blast radius.

## Consequences

**Positive.**

- **Repo is fully publishable.** The workload code is the user's, owned outright; no IP review, no fork-maintenance overhead. The CV claim — "I built this platform end-to-end" — covers the workload as well as the platform.
- **Fidelity contract is testable.** Each item in the contract maps to a Phase 5/6 chaos experiment or canary-rollout test ([design.md §5.7](../design.md#57-failure-handling-and-self-healing)). "Self-healing for this class of failure" becomes a measurable property, not a claim.
- **Language choice compounds with platform-tool choices.** Go's static binaries fit distroless ([threat-model.md §B8:E](../threat-model.md)), pgx is the gold-standard Postgres pool for SRE-grade tuning ([design.md §9 open question 4](../design.md#9-risks-and-open-questions)), OTel-Go is mature for the trace-id-in-log-line pattern ([ADR-005](005-observability-lgtm-self-hosted.md)). Each of these is a follow-up consequence with zero added cost because the language fits.
- **The `disallow-paymob-log-mode-in-prod` Kyverno policy ([ADR-004](004-admission-policy-kyverno.md)) has a real target.** Without a workload that actually exposes `PAYMENTS_PROVIDER_MODE=log` as a footgun, the policy is decorative. With it, the Phase 6 admission demo refuses a real misconfiguration.
- **Two-service split preserves canary independence.** `orderd` and `reconcilerd` roll out independently under Argo Rollouts ([design.md §5.5](../design.md#55-reliability-mechanisms)); a regression in the webhook path doesn't block synchronous order traffic and vice versa. Mirrors how Ahoy's FastAPI/Celery split would behave at production scale.

**Negative.**

- **Learning-Go cost.** The user is explicitly learning Go as part of this project. Workload progress will be slower than for a Python expert. Mitigation: the workload is intentionally minimal — the fidelity contract is the spec, not the surface area. Two services × ~500–1000 LOC each is achievable; the platform itself is the larger lift.
- **Loss of Ahoy-specific product richness in the threat model.** [threat-model.md §A4](../threat-model.md) (inventory + BI + PriceList) doesn't have a workload-side equivalent post-pivot. Mitigation: A4 is rewritten to cache-state and broker-state concerns, which are still real assets for the new workload. The lost product detail wasn't earning its place in a platform threat model anyway.
- **Mock payments provider has fidelity ceiling.** A mock can simulate Paymob's HMAC, retry cadence, and timing, but not its real-world quirks (rate-limit response codes, undocumented latency tails, occasional duplicate webhook deliveries). Phase 6 chaos experiments compensate via injected fault patterns; the gap is named, not closed.
- **Workload becomes a separate maintenance surface.** `ahoy-app` is a real repo with its own CI, dependency updates, security patches. At one-engineer scale this is real friction. Mitigation: keep the workload locked to a fidelity-contract scope; any feature that doesn't trace to a row in the contract is rejected.

**Follow-up work this commits us to.**

- Phase 2 ships: `ahoy-app` repo with `cmd/orderd/`, `cmd/reconcilerd/`, shared `internal/` packages (DB, broker, observability, http middleware), Dockerfiles per binary (multi-stage, distroless static, non-root), a `payments-mock` service for local + chaos use, and migrations for `orders` + `ledger_entries` + a webhook audit table closing [threat-model.md §B2:R](../threat-model.md).
- Phase 2 also ships: Helm chart in `ahoy-config` with the resource requests/limits from [design.md §5.2](../design.md#52-application-layer), ServiceAccounts with `automountServiceAccountToken: false`, NetworkPolicy stubs that Phase 6 fills in.
- Phase 3 ships: GHA workflow that builds + signs both binaries via Cosign keyless ([ADR-003](003-image-signing-cosign-keyless.md)), opens the config-repo PR bumping both image digests in one commit.
- Phase 4 ships: route-template Prometheus middleware in `internal/http/`, OTel-Go instrumentation in `cmd/*` main funcs, structured zerolog with `trace_id` enrichment, recording rules in `ahoy-config/` for the CUJ-1 and CUJ-2 SLIs.
- Phase 5 ships: Argo Rollouts manifests for both binaries with Prometheus-backed analysis; the analysis query for CUJ-2 reads the `webhook_to_ledger_seconds` histogram emitted from `reconcilerd`.
- **Library pinning is deliberately deferred to Phase 2 kickoff** — `chi` vs the stdlib mux, `pgx` v5 minor version, `go-redis` major, `zerolog` vs `slog`, OTel-Go SDK version. These decisions belong to the build phase, not Phase 1; pinning prematurely commits to choices the user hasn't earned through use.
- **Ahoy stays an off-repo private reference.** Failure modes the user encounters in Ahoy that aren't yet in the fidelity contract above become candidate additions — each new item is an amendment to this ADR with a one-line justification linking to the chaos experiment or rollout test that exercises it.
