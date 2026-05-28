# Threat Model

Threat model for the self-healing platform demonstrated against the Go workload from [ADR-006](adr/006-workload-go-shim.md) (`orderd` + `reconcilerd` + a payments-provider mock modeling Paymob), at the projected post-Series-A scale documented in [problem-statement.md](problem-statement.md). Anchored on the [SLOs](slos.md) and the target architecture in [architecture.md](architecture.md).

## Scope and method

**What this document is.** A structured answer to: (1) what are we protecting, (2) who would attack it and how, (3) what stops them today, (4) what risk remains. Threats are tied to *specific* code paths in the workload where possible; abstract claims are flagged for verification.

**Method.**
1. Enumerate **assets** grounded in the Go workload's actual data model and integration surface ([ADR-006](adr/006-workload-go-shim.md)), with the Ahoy incident as the motivating reference for which failure-mode classes matter.
2. Identify **trust boundaries** from the target architecture.
3. Walk **STRIDE** (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege) at each boundary. Each STRIDE row records: *Threat*, *Vector*, *Current mitigation*, *Residual risk + platform requirement*.
4. Roll up to **top risks** and **decisions traceability** — every platform pick should trace back to a residual-risk row that justifies it.

**Phase 1 depth.** Nine boundaries are identified; two are walked end-to-end (B2 payments-provider webhook, B8 supply chain) because they have the highest revenue impact and the most concrete present-day evidence from the Ahoy incident. The remaining seven are stubbed with their top concern noted; deeper walks land in subsequent quarters as the platform matures.

## Assumptions

Stated explicitly because a threat model is only as defensible as its assumptions.

1. The target state described in [architecture.md](architecture.md) is what we are modeling. The pre-platform baseline (the Ahoy incident state: single EC2, no signing, no admission policy, CloudWatch-only telemetry) is used to identify gaps and motivate platform decisions.
2. One trust domain inside the cluster (per [problem-statement non-goal #2](problem-statement.md#non-goals)). All app teams are equally trusted; namespace-level RBAC and NetworkPolicies are the only segmentation.
3. AWS-level controls (IAM scoping, VPC ACLs, RDS at-rest encryption via KMS, SSM Parameter Store access policies) are correctly configured by the infra team. Misconfigurations there are out of scope here but become a row in the AWS threat model.
4. The real payments provider's gateway integrity is out of scope. A provider compromise is a vendor incident, not a platform one — but its blast radius is captured in B2. The in-cluster payments-mock has the same trust posture as a real provider for STRIDE purposes.
5. Browsers and HTTP clients honor TLS and same-origin policy.
6. Insiders with legitimate access are modeled only via **credential compromise** and **accidental misconfiguration**, not as deliberate adversaries. Adversarial-insider modeling is a separate exercise.

## Assets

What an attacker would want from the workload, grounded in the Go workload's actual model ([ADR-006](adr/006-workload-go-shim.md)). Each asset's "concrete attacker actions" includes the Ahoy incident's evidence where applicable — the workload exists specifically to make these incident classes reproducible.

### A1. Customer PII

The Go workload's `orders` table carries the minimal PII needed to model the order-placement path:

- `customer_email`, `customer_phone`, `delivery_address` — per-order fields submitted on `POST /api/v1/orders`. Used for confirmation flows and delivery routing.
- `customer_id` — optional account binding; an associated `customers` table holds `email` and `argon2-hashed password` for the small subset of authenticated callers.

**Concrete attacker actions.** Exfiltrate `orders.customer_*` → DTC contact list usable for vishing as "order support." Exfiltrate `customers.password_hash` → offline brute-force, then credential-stuff other services. At Ahoy's incident scale, the equivalent exfiltration would have included B2B customer contacts with geocoordinates and PriceList data — product depth that the workload intentionally omits ([ADR-006 §Out of scope](adr/006-workload-go-shim.md)). The platform's PII protections (default-deny NetworkPolicies, telemetry PII scrubber, mTLS) are sized for the broader Ahoy data shape, not just the workload's surface.

### A2. Payment data and payments-provider integration

- Webhook handler — `orderd`'s `POST /api/v1/webhooks/paymob` endpoint. HMAC-SHA512 verification with constant-time compare (`hmac.Equal` in Go). Idempotency by `transaction_id`. Amount validated against `order.total` before status change.
- Secrets: `PAYMOB_API_KEY`, `PAYMOB_HMAC_SECRET`, `PAYMOB_INTEGRATION_ID` — loaded from environment at container start, materialized via External Secrets Operator in Phase 6 (interim: Kubernetes Secrets in Phase 2–5).
- No raw PANs stored — the provider holds them; the workload sees only `transaction_id`, `paymob_order_id`, `amount`.

**Concrete attacker actions.** Forge webhooks to mark orders PAID without paying — the exact incident class the Paymob near-miss exposed in Ahoy when a `PAYMOB_MODE=log` env var short-circuited verification. The workload exposes the analogous `PAYMOB_MODE=log` footgun deliberately ([ADR-006 §Fidelity contract](adr/006-workload-go-shim.md)) so the Kyverno policy `disallow-paymob-log-mode-in-prod` ([ADR-004](adr/004-admission-policy-kyverno.md)) has a real footgun to refuse.

### A3. The ledger

- Schema — `ledger_entries(id, transaction_ref, debit, credit, committed_at, attributed_to)`. CHECK constraints (`debit >= 0`, `credit >= 0`, `NOT (debit > 0 AND credit > 0)`).
- Entry IDs via Postgres sequence — atomic, gap-safe.
- "Immutable" by application convention only — no DB-level append-only constraint by default, no trigger preventing UPDATE/DELETE on `ledger_entries` outside the role separation enforced in [design.md §5.8](design.md#58-data-layer), no hash chain.
- Webhook-driven writes all attributed to the worker's service-account identity (`attributed_to = 'reconcilerd'` in the workload; modeled on Ahoy's `SYSTEM_USER_ID` pattern from the incident).

This is a deliberately simplified version of the Ahoy double-entry ledger (Journal Entry header + Journal Lines), preserved only at the depth needed to reproduce the reconciliation failure classes ([ADR-006 §Out of scope](adr/006-workload-go-shim.md)).

**Concrete attacker actions.** Anyone with `DATABASE_URL` write access (DBA, leaked credential, escaped pod) directly UPDATEs `ledger_entries` to commit silent fraud — no audit trail beyond what the application records. Forged webhook (per A2) commits a fake ledger entry attributed to `reconcilerd`'s identity, indistinguishable from a legitimate webhook write.

### A4. Cache and broker state

- Redis serves dual purposes: order-lookup cache (read-aside from `orderd`) and broker for `reconcilerd`'s job queue.
- Cache contents include order summaries (per-customer minimal fields, see A1). Broker contents include the raw webhook payload + HMAC header awaiting reconciliation by `reconcilerd`.

This asset replaces Ahoy's inventory + BI + PriceList surface — the workload doesn't model that depth ([ADR-006](adr/006-workload-go-shim.md)). The remaining state in Redis is small but still load-bearing for CUJ-1 and CUJ-2.

**Concrete attacker actions.** Exfiltrate cache → recent customer + order snapshots. Tamper with broker queue → drop or replay webhooks → ledger drift. Exhaust broker memory → CUJ-2 burn (the Paymob-incident shape). Cache poisoning (write directly to Redis) → `orderd` serves attacker-controlled order data.

### A5. The ability to ship code to production

- CI: `.github/workflows/build.yml` in `ahoy-app` — lint, test, build on push to main.
- Deploy: GitOps pull-based via Argo CD ([ADR-002](adr/002-gitops-argo-cd.md)) — CI opens a PR in `ahoy-config` bumping image digests; merge triggers reconciliation.
- Images signed by Cosign keyless ([ADR-003](adr/003-image-signing-cosign-keyless.md)) and verified at admission by Kyverno ([ADR-004](adr/004-admission-policy-kyverno.md)).
- Image references in manifests are by **digest only**, never tag — admission refuses tagged references.
- Pre-deploy RDS snapshot. Automated rollback via Argo Rollouts analysis on SLO regression.

Pre-platform baseline (the Ahoy incident state): mutable `:latest` tags, no signing, push-deploy via SSM, no admission verification. The platform's supply chain closes each of those gaps.

**Concrete attacker actions.** Compromise CI's OIDC trust (misconfigured `sub` claim wildcard) → AWS issues build-role token to malicious fork → push malicious image to GHCR. Compromise a developer's GitHub account → push to main → pipeline to production. Tamper with `ahoy-config` repo (no signed commits) → Argo CD applies attacker-controlled manifests.

### A6. Secrets

- Workload runtime: `DATABASE_URL`, `REDIS_URL`, `PAYMOB_API_KEY`, `PAYMOB_HMAC_SECRET`, `PAYMOB_INTEGRATION_ID`, `JWT_SIGNING_KEY` — materialized via External Secrets Operator from Vault (Phase 6); interim Kubernetes Secrets in Phase 2–5.
- CI: no long-lived AWS keys — OIDC trust to a per-workflow role only.
- No committed secrets — gitleaks gates this in CI.

**Concrete attacker actions.** Vault read from compromised pod → all workload secrets. `JWT_SIGNING_KEY` leak → forge any user's access token without a DB query. `PAYMOB_HMAC_SECRET` leak → see A2.

## Trust boundaries

Lines where trust, network, authority, or code-origin changes. Each one is an attack surface.

| ID | Boundary | What changes |
|---|---|---|
| **B1** | Customer browser/client ↔ Ingress | Network (internet→cluster), authority (unknown user→our platform) |
| **B2** | Payments provider ↔ Ingress (webhook path) | Network + origin (third-party initiates, no human session) |
| **B3** | Ingress ↔ App namespace pods | Network (edge→workload), TLS-terminated trust |
| **B4** | App namespace ↔ Platform namespace | Privilege (workload→controllers), team (app→platform) |
| **B5** | App namespace ↔ Observability namespace | Origin (write-only telemetry path) |
| **B6** | Cluster ↔ Managed services (RDS, Vault, PagerDuty) | Network (cluster→AWS-managed), authority (we don't run those) |
| **B7** | Developer ↔ GitHub | Origin (human laptop → cloud platform) |
| **B8** | GitHub Actions ↔ GHCR ↔ deploy | Origin (CI runner → registry), privilege (writes prod artifacts) |
| **B9** | Argo CD ↔ Cluster (apply) | Privilege (controller has wide cluster-write authority) |

Two boundaries are walked in depth below: **B2** (highest concentration of present-day evidence, ties to the trigger event in problem-statement) and **B8** (highest single-compromise blast radius). The remaining seven are stubbed in [§Remaining boundaries](#remaining-boundaries-stubbed-for-later).

## B2 — Payments provider → Ingress (webhook path)

The boundary the Ahoy incident landed at. The Go workload's `orderd` implements an analogous webhook handler so the threats are testable end-to-end; the historical reference for shape and evidence is Paymob.

### B2:S — Spoofing

**Threat.** An attacker forges a payments-provider webhook to mark a fake order as PAID without paying.

**Vector.** POST to `/api/v1/webhooks/paymob` with a crafted JSON and a fabricated `hmac` query parameter. Enabling conditions: (a) `PAYMOB_MODE=log` mistakenly set in prod — the workload's HMAC verifier returns `true` unconditionally in this mode (deliberate footgun per [ADR-006 §Fidelity contract](adr/006-workload-go-shim.md), mirroring the exact Ahoy incident vector); (b) `PAYMOB_HMAC_SECRET` leaks via Vault compromise, pod escape, or a departing employee with stale access.

**Current mitigation.** HMAC-SHA512 with `hmac.Equal` (constant-time). Idempotency by `transaction_id` against the workload's `payment_log` table. Amount validated against `order.total` before status flip.

**Residual risk — High, config-driven.** One wrong env var or one leaked secret → silent fraud. Platform must close with: (1) Kyverno admission policy refusing pods that carry `PAYMOB_MODE=log` outside dev namespaces ([ADR-004](adr/004-admission-policy-kyverno.md)), (2) External Secrets + short-lived rotation backed by Vault, (3) out-of-band reconciliation comparing the provider's daily settlement report against the workload's ledger, (4) network-layer source filter if the provider publishes a stable IP allowlist.

### B2:T — Tampering

**Threat.** Webhook payload modified in transit to fake a successful payment or change the amount.

**Vector.** TLS termination at ingress defeats most network MITM. The realistic vector is **in-cluster traffic** between ingress and the `orderd` pod — plaintext until cert-manager + mTLS lands in Phase 6, with NetworkPolicy enforcement landing alongside it. Until then a malicious sidecar or compromised same-namespace pod could read or modify.

**Current mitigation.** HTTPS to ingress. The amount-validation step in the handler defends even a tampered `amount_cents` field because the order total is looked up server-side.

**Residual risk — Medium.** Defended at the app layer by amount validation, but the in-cluster "any pod talks to any pod" assumption needs Phase 6 NetworkPolicies default-deny + cert-manager mTLS to actually be true.

### B2:R — Repudiation

**Threat.** In a payment dispute we cannot reconstruct what happened — only that *some* webhook caused *some* state change.

**Vector.** (a) Customer claims "I paid but you didn't credit me" — without a raw-payload audit table, we have only the `payment_log` row and not the bytes the provider actually sent. (b) Someone with DB write access flips `order.payment_status` directly — the resulting row is indistinguishable from a legitimate webhook write (both attributed to `reconcilerd`).

**Current mitigation.** Partial. `payment_log` records `transaction_id`, amount, and outcome. The platform requires `orderd` to additionally persist raw payload + HMAC header + receipt timestamp in an append-only audit table (a Phase 2 workload deliverable, per [design.md §5.6 Audit](design.md#56-security-posture)).

**Residual risk — Medium-high in B2B disputes** where chargebacks have teeth. Without the raw audit trail, we depend on the provider's records to defend ourselves — third-party evidence dependency. Platform must: (1) persist raw payload + HMAC header + receipt timestamp in an append-only audit table keyed by `transaction_id`, (2) give each webhook receipt a correlation ID that `payment_log` rows trace back to, (3) export GHA + Argo CD audit logs to an external append-only store (S3 with Object Lock).

### B2:I — Information disclosure

**Threat.** Webhook responses leak operational state to an attacker who possesses (or has guessed) a valid HMAC signature.

**Vector.** With `PAYMOB_HMAC_SECRET` compromised (per B2:S), an attacker enumerates by crafting valid HMACs for arbitrary `(transaction_id, paymob_order_id, amount)` tuples and reading the distinguishable response: `{"status": "duplicate"}` reveals which transaction_ids exist; `{"status": "ignored", "reason": "order not found"}` reveals which paymob_order_ids exist; `{"status": "rejected", "reason": "amount mismatch"}` allows binary search on the order total.

**Current mitigation.** None at the handler by default. HMAC verification is the only gate; once past it, the response is verbose unless explicitly collapsed.

**Residual risk — Low standalone, but amplifies B2:S** from "forge one payment" to "map the entire payment history." Platform must: (1) require `orderd`'s webhook handler to return a single opaque `{"status": "received"}` regardless of internal outcome (log the real outcome internally), (2) treat the handler as a write-only sink from the caller's perspective.

### B2:D — Denial of service

**Threat.** Webhook endpoint flooded → `orderd` pod CPU exhausted, DB connection pool saturated, broker queue depth explodes → real webhooks miss the 60s reconciliation SLO ([slos.md](slos.md) CUJ-2). **This is the exact shape of the Ahoy Paymob near-miss**, and the failure mode the workload is built to reproduce ([ADR-006 §Fidelity contract](adr/006-workload-go-shim.md)).

**Vector.** (a) Anonymous flood — every request still incurs JSON parse + HMAC compute + 401 round-trip; bulk small payloads at high rate. (b) Authenticated flood with leaked HMAC — each request runs the full path: ~4 DB roundtrips, audit-table insert, order update, broker enqueue. (c) Slowloris-style — hold connections open with no body.

**Current mitigation.** TLS termination at ingress. Per-IP rate limit at ingress (Phase 3 deliverable). Body-size limit at ingress.

**Residual risk — High.** Directly burns CUJ-2 error budget. Platform must: (1) ingress rate limit per source IP with provider's published IPs allowlisted at higher quota, (2) request body size limit (≤1KB on the webhook path), (3) HPA + PDBs on `orderd` and `reconcilerd`, (4) bounded broker queue with a dead-letter queue and worker autoscaling, (5) circuit-break the handler when queue depth exceeds threshold (return 503 fast → provider retries → buys recovery time).

### B2:E — Elevation of privilege

**Threat.** A forged webhook escalates an outside attacker to the equivalent of a privileged system actor — able to mark arbitrary orders PAID, insert audit-table rows, and (downstream) trigger ledger writes, all attributed to `reconcilerd` with no traceability to a real principal.

**Vector.** With HMAC compromised (per B2:S), POST a crafted webhook for any `(order_id, amount)` the attacker chooses → handler runs the full privileged write path. There is **no per-request authorization** — HMAC is treated as both authentication and authorization for any state transition the webhook supports.

**Current mitigation.** HMAC is the sole gate. Amount validation prevents "mark a $1000 order paid for $0.01." Idempotency by `transaction_id` prevents trivial replay.

**Residual risk — High.** Platform must: (1) least-privilege identity for `reconcilerd`'s DB role — `INSERT/UPDATE` only the specific `orders` + `ledger_entries` + `payment_log` columns it owns; **never** ledger reversals, amount changes, or arbitrary order edits, (2) out-of-band confirmation — on first receipt of a transaction_id, call back to the provider's API to confirm "did you process X?" before committing the status flip; this collapses forgery from "one secret leak" to "one secret leak AND fool the outbound provider call."

## B8 — GitHub Actions → registry → cluster (the supply chain)

Whoever owns any link of this chain effectively owns production. No defense-in-depth past it.

### B8:S — Spoofing

**Threat.** An attacker convinces the cluster (or a step in the chain) to run code as if it were ours.

**Vector.** (a) Steal a developer's GitHub credentials → push to main → CI builds and deploys malicious code attributed to their identity. (b) Exploit a misconfigured OIDC `sub` claim (the classic `sub:*` mistake) → AWS or GHCR issues a build-role token to a malicious fork → push malicious image. (c) Without image signing + admission verification, any actor with registry push permission overwrites image references at the tag level; the Ahoy incident-state baseline.

**Current mitigation.** GitHub auth (assume 2FA enforced). OIDC trust policies with `sub` claim pinned to `repo:org/ahoy-app:ref:refs/heads/main`, not a wildcard. Cosign keyless signing ([ADR-003](adr/003-image-signing-cosign-keyless.md)) + Kyverno `verifyImages` ([ADR-004](adr/004-admission-policy-kyverno.md)) close the unsigned-image vector at admission time.

**Residual risk — Low at target state, High in Phase 2 before signing + admission verification land.** Platform must: (1) signed commits on main, (2) verify OIDC `sub` is pinned, (3) Cosign keyless signing in CI + Kyverno `verifyImages` policy at admission refusing unsigned or wrong-signer images, (4) image references in manifests **by digest, not by tag**.

### B8:T — Tampering

**Threat.** Source code, image bytes, or deployment config altered between developer intent and pod execution.

**Vector.** (a) Push directly to main bypassing review (branch protection off or admin override). (b) Third-party action consumed via `uses: foo/bar@v1` — `@v1` is a moving tag, not a SHA; upstream maintainer compromise propagates. (c) Mutable tag overwritten in the registry (defeats unless digest-pinned). (d) Config repo has no commit signing; tampering at the `ahoy-config` level isn't detected.

**Current mitigation.** Branch protection on main, PR review required. The workload's CI pins every third-party action to a commit SHA.

**Residual risk — Medium-high for actions if SHA-pinning lapses, Low for image bytes once signing lands.** Platform must: (1) pin every third-party action by SHA (`uses: actions/checkout@<sha>`), (2) Cosign signature on every image + Kyverno verification at admission, (3) Argo CD configured to verify GPG-signed commits on the config repo, (4) image references in manifests by digest only.

### B8:R — Repudiation

**Threat.** After an incident, the chain "commit → image → deployment" cannot be reconstructed with high confidence.

**Vector.** (a) Git commits unsigned → attribution to a developer is just a string. (b) GHA logs are not append-only or independently signed → a GitHub org owner can rerun or delete a workflow run, removing forensic evidence. (c) No build provenance attestation → cannot prove which commit + which workflow run produced which image digest.

**Current mitigation.** Git history. GHA audit logs (default 90-day retention). GHCR records `imageDigest` and `imagePushedAt`. Cosign signature carries the workflow identity in the Rekor entry.

**Residual risk — Medium, defensible at moderate cost.** Platform must: (1) require signed commits on main, (2) **SLSA-style build provenance attestations** signed by Cosign — bind `(commit_sha, workflow_run_id, builder_identity, image_digest)` at build time, verify at admission, (3) export GHA audit logs to S3 with Object Lock, (4) preserve Argo CD sync history beyond default rotation.

### B8:I — Information disclosure

**Threat.** Secrets or proprietary metadata leak out of the CI pipeline.

**Vector.** (a) `echo $SECRET_X` in a debug step — GHA masking only catches secrets declared via `secrets:`, not arbitrary derivations. (b) Image layer accidentally bakes a `.env` file (no `--secret` mount in Dockerfile). (c) **Third-party actions** receive the workflow's secrets and OIDC token by design — a compromised action is a full secret compromise. (d) GHCR repository accidentally set public.

**Current mitigation.** GHA secret masking. OIDC eliminates long-lived AWS keys. The workload uses `secrets.X` references only; gitleaks gates accidental commits.

**Residual risk — Medium.** Platform must: (1) `--secret` mount pattern in all Dockerfiles, never `COPY .env`, (2) third-party action allowlist (pinned, audited set only), (3) gitleaks in CI as the second-line backstop, (4) GHCR repo policies explicitly private and reviewed in PR.

### B8:D — Denial of service

**Threat.** Supply chain unavailable when we most need to deploy: during an incident.

**Vector.** (a) GitHub outage — no commits, no CI, no merge. (b) GHCR regional outage — no image pull on new pods. (c) Argo CD pod unhealthy or source repo unreachable — drift uncorrected. (d) GHCR storage quota exhausted from accumulated images with no lifecycle policy.

**Current mitigation.** GHCR is multi-region. Kubelet image cache means running pods survive a registry outage. Pre-deploy RDS snapshot for the workload.

**Residual risk — Medium**, mostly inherited from SaaS. Platform should: (1) image lifecycle policy (delete untagged, keep last N tagged), (2) Argo CD HA replicas, (3) documented break-glass deploy path (manual `kubectl apply` with verifiable image digest, bypassing Argo CD when Argo CD itself fails), (4) digest-pinned manifests mean restarted pods can run from kubelet cache even without registry availability.

### B8:E — Elevation of privilege

**Threat.** A compromised step in the pipeline gains privileges that allow it to take over production.

**Vector.** (a) A pinned third-party action whose upstream SHA history was rewritten (force-push by a compromised maintainer) exfiltrates the workflow's OIDC token → assumes the AWS build role or GHCR push role → pushes a malicious image. (b) The build role has more permissions than strictly "push image X to repo Y" → blast radius widens. (c) Argo CD service account has cluster-wide write permissions by design → its compromise = cluster takeover.

**Current mitigation.** Short-lived OIDC tokens (per-workflow). Argo CD ships with cluster-admin-equivalent for its target namespaces by default.

**Residual risk — High if build role is over-permissioned; structurally high for Argo CD by design.** Platform must: (1) scope the build role strictly to "push to repo X" — no other repos, no other resources, (2) Argo CD with namespace-scoped Projects (not cluster-wide), per-team RBAC, (3) Kyverno policies refusing privileged containers, hostPath, hostNetwork — bounds what a compromised Argo CD can grant, (4) default-deny NetworkPolicies so a compromised pod cannot pivot.

## Remaining boundaries (stubbed for later)

Identified but not yet walked through STRIDE in full. Top concern noted; deeper walks are scheduled for after the platform's first production cut.

- **B1 — Customer browser/client ↔ Ingress.** Generic web threats: SQL injection (defended by parameterized queries via pgx), XSS (CSP headers required when an authenticated UI exists), session hijack (HttpOnly + Secure cookies + short-lived JWTs), DDoS on `POST /api/v1/orders` directly burning CUJ-1 SLO. First letter to walk: **D** (volumetric attack on order placement).
- **B3 — Ingress ↔ App namespace pods.** In-cluster lateral movement; without NetworkPolicies + mTLS, an ingress controller compromise reaches every workload pod. First letter to walk: **E** (privilege from edge into workloads).
- **B4 — App namespace ↔ Platform namespace.** Privilege escalation via misconfigured RBAC or shared service accounts. First letter to walk: **E** (app pod escaping to Argo CD's namespace).
- **B5 — App namespace ↔ Observability namespace.** Log injection (attacker-controlled log content interpreted by downstream tooling); PII inadvertently captured in telemetry then retained in Loki/Tempo. First letter to walk: **I** (PII in traces).
- **B6 — Cluster ↔ Managed services (RDS, Vault, PagerDuty).** **Highest unwalked priority.** The ledger-immutability gap (A3) lives here: anyone with `DATABASE_URL` write access bypasses every app-level invariant. Secret leak via Vault compromise concentrates here too. First letter to walk: **T** (tampering with ledger via direct DB access), followed by **I** (secret exfiltration from Vault).
- **B7 — Developer ↔ GitHub.** Developer laptop compromise → push to main. Partially covered by B8:S/R; standalone walk addresses developer environment hardening (SSH key handling, signed commit tooling, MFA enforcement).
- **B9 — Argo CD ↔ Cluster.** Argo CD's service account = cluster-admin-equivalent within its target namespaces. Partially covered by B8:E; standalone walk addresses Argo CD-specific controls (Project scoping, sync windows, declarative RBAC).

## Top risks (rolled up)

The five rows future-you should care about first. Each is grounded in a STRIDE row above; this list is the executive view.

1. **Config-driven HMAC bypass (B2:S).** `PAYMOB_MODE=log` returning `true` unconditionally in the webhook verifier means one misconfigured env var collapses webhook authentication. The Ahoy incident's exact vector; the workload reproduces it on purpose. **Mitigation: Kyverno admission refusing the env var in prod namespaces ([ADR-004](adr/004-admission-policy-kyverno.md)).**
2. **Ledger immutability is app-only (A3, B6).** Direct DB UPDATE/DELETE on `ledger_entries` bypasses every "immutability" the application enforces. Anyone with `DATABASE_URL` write access commits silent fraud. **Mitigation: separate DB user for `reconcilerd` vs admin; per-table DB-level append-only or trigger guards; audit-log table written by trigger.**
3. **Unsigned, mutable image tags (A5, B8:S/T).** The Ahoy incident-state pattern (`:latest`) means whoever owns registry push owns production. **Mitigation: Cosign keyless signing ([ADR-003](adr/003-image-signing-cosign-keyless.md)) + Kyverno `verifyImages` ([ADR-004](adr/004-admission-policy-kyverno.md)) + digest-pinned manifests.**
4. **Forged webhook = privileged worker identity (B2:E).** `reconcilerd`'s DB role can write to `orders`, `ledger_entries`, and `payment_log`; a single successful forgery (per #1) commits arbitrary order/ledger state changes. **Mitigation: scope `reconcilerd`'s role to UNPAID→PAID transitions only and the specific audit-log inserts; out-of-band provider confirmation on first receipt of a transaction_id.**
5. **No raw webhook audit trail (B2:R).** Disputes cannot be reconstructed because the raw provider payload isn't persisted unless `orderd` writes it; we depend on the provider's records to defend ourselves. **Mitigation: append-only audit table keyed by `transaction_id`, with full payload + HMAC + receipt timestamp — a Phase 2 workload deliverable per [ADR-006 §Follow-up work](adr/006-workload-go-shim.md).**

## Decisions this threat model justifies

Traceability between the residual-risk rows and the platform's specific Phase 2–6 decisions. Every item on the right has at least one row above that *requires* it.

| Platform decision | Phase | Justifying rows |
|---|---|---|
| Workload as a Go shim modeling the incident classes | P1 (paper) | A2, A3, B2 (entire walk) — failure modes must be reproducible to be tested |
| Cosign keyless signing in CI | P3 | B8:S, B8:T, B8:R |
| Kyverno admission + `verifyImages` | P6 | B8:S, B8:T |
| Digest-pinned image references in manifests | P5/P6 | B8:S, B8:T |
| SHA-pinned third-party GHA actions | P3 | B8:T |
| SLSA-style build provenance attestations | P6 | B8:R |
| External Secrets Operator + Vault | P6 | A6, B2:S, B8:I |
| Default-deny NetworkPolicies | P6 | B2:T, B3, B8:E |
| mTLS via cert-manager | P6 | B2:T |
| Argo CD namespace-scoped Projects | P3 | B8:E, B9 |
| Append-only webhook audit table written by `orderd` | App-level (Phase 2 workload deliverable) | B2:R |
| Out-of-band provider reconciliation job | App-level (Phase 2 workload deliverable) | B2:S, B2:E, top risk #4 |
| Least-privilege `reconcilerd` DB role | App-level | B2:E |
| Ingress rate limit + body size limit on webhook | P2/P3 | B2:D |
| HPA + PDBs + bounded broker queue | P5 | B2:D |
| GHCR image lifecycle policy | P3 | B8:D |
| Audit-log export to S3 Object Lock | P4 | B2:R, B8:R |
| Signed commits enforced on main | P3 | B8:R, B8:S |

## Review and lifecycle

- **Re-walk B6 within the first quarter of production data.** The ledger-DB boundary is the highest-priority unwalked area; deferred only because the platform's DB controls (PgBouncer, role separation, audit triggers) are not yet decided.
- **Re-walk other boundaries** after each major architecture change.
- **Post-incident review.** Every Sev-1 or Sev-2 incident produces an audit of the relevant boundary's row: does the threat row need updating? Did the residual-risk language match what actually happened?
- **Annual.** Reconcile this document against the architecture diagram. Drift between them is a defect in either or both.
