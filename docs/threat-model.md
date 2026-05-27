# Threat Model

Threat model for the self-healing platform hosting Ahoy at projected post-Series-A scale. Anchored on the [problem statement](problem-statement.md), the [SLOs](slos.md), and the target architecture in [architecture.md](architecture.md).

## Scope and method

**What this document is.** A structured answer to: (1) what are we protecting, (2) who would attack it and how, (3) what stops them today, (4) what risk remains. Threats are tied to *specific* code paths in Ahoy where possible; abstract claims are flagged for verification.

**Method.**
1. Enumerate **assets** grounded in the real Ahoy codebase.
2. Identify **trust boundaries** from the target architecture.
3. Walk **STRIDE** (Spoofing, Tampering, Repudiation, Information disclosure, Denial of service, Elevation of privilege) at each boundary. Each STRIDE row records: *Threat*, *Vector*, *Current mitigation*, *Residual risk + platform requirement*.
4. Roll up to **top risks** and **decisions traceability** — every platform pick should trace back to a residual-risk row that justifies it.

**Phase 1 depth.** Nine boundaries are identified; two are walked end-to-end (B2 Paymob webhook, B8 supply chain) because they have the highest revenue impact and the most concrete present-day evidence. The remaining seven are stubbed with their top concern noted; deeper walks land in subsequent quarters as the platform matures.

## Assumptions

Stated explicitly because a threat model is only as defensible as its assumptions.

1. The target state described in [architecture.md](architecture.md) is what we are modeling. Current Ahoy (single EC2, no signing, no admission policy) is the baseline used to identify gaps and motivate platform decisions.
2. One trust domain inside the cluster (per [problem-statement non-goal #2](problem-statement.md#non-goals)). All app teams are equally trusted; namespace-level RBAC and NetworkPolicies are the only segmentation.
3. AWS-level controls (IAM scoping, VPC ACLs, RDS at-rest encryption via KMS, SSM Parameter Store access policies) are correctly configured by the infra team. Misconfigurations there are out of scope here but become a row in the AWS threat model.
4. Paymob's gateway integrity is out of scope. A Paymob compromise is a vendor incident, not a platform one — but its blast radius is captured in B2.
5. Browsers honor TLS and same-origin policy. No mobile app in scope (none exists today).
6. Insiders with legitimate access are modeled only via **credential compromise** and **accidental misconfiguration**, not as deliberate adversaries. Adversarial-insider modeling is a separate exercise.

## Assets

What an attacker would want from Ahoy, grounded in the real codebase at `/home/ibrahim/study/ahoy`.

### A1. Customer PII

- **B2B customers** — `backend/app/models/customer.py:48–75`: code, contact name, email, phone, address, **latitude/longitude**, balance, credit limit, assigned salesperson, price list.
- **DTC guest orders** — `backend/app/models/order.py:118–120`: `guest_customer_name`, `guest_customer_phone`, `guest_delivery_address` (no account required).
- **Employees** — `backend/app/models/user.py:64–82`: email + argon2-hashed password (`backend/app/core/security.py:14`) + role.
- **Refresh tokens** — stored hashed in the DB, not plaintext.

**Concrete attacker actions.** Exfiltrate B2B customers table → thousands of MENA business contacts + geocoords → resell to FMCG competitors or targeted SMS spam. Exfiltrate `guest_customer_*` from orders → DTC phone book usable for vishing as "Ahoy delivery support." Employee email list → targeted phishing campaigns.

### A2. Payment data and Paymob integration

- Webhook handler — `backend/app/api/webhooks/paymob.py:31–158`. HMAC-SHA512 verification with constant-time compare. Idempotency by `transaction_id`. Amount validated against `order.total` before status change.
- Secrets: `PAYMOB_API_KEY`, `PAYMOB_HMAC_SECRET`, `PAYMOB_INTEGRATION_ID`, `PAYMOB_IFRAME_ID` — loaded from AWS SSM Parameter Store at container start via `infra/scripts/fetch-ssm-env.sh`.
- No raw PANs stored — Paymob holds them; system sees only `transaction_id`, `paymob_order_id`, `amount`.

**Concrete attacker actions.** Forge webhooks to mark orders PAID without paying. Enumerate transaction history via response-distinguishability side channel (see B2:I). Replay attacks defeated by idempotency.

### A3. The ledger

- Schema — `backend/app/models/accounting.py:85–182`: JournalEntry (header) + JournalLine (debits/credits). CHECK constraints (`debit >= 0`, `credit >= 0`, `NOT (debit > 0 AND credit > 0)`).
- Entry numbers via Postgres sequence — atomic, gap-safe.
- "Immutable" by application convention only — `is_reversed`, `reversed_by_id`. **No DB-level append-only constraint, no trigger preventing UPDATE/DELETE on `journal_lines`, no hash chain.**
- Webhook-driven writes all attributed to `SYSTEM_USER_ID = 00000000-0000-0000-0000-000000000001` (`backend/app/api/webhooks/paymob.py:28`).

**Concrete attacker actions.** DBA with RDS write access (or anyone with a leaked `DATABASE_URL`) directly UPDATEs `journal_lines` to commit silent fraud — no audit trail. Forged webhook (per A2) commits a fake journal entry attributed to SYSTEM_USER_ID.

### A4. Inventory and business intelligence

- Batches with `production_date`, `expiry_date`, `unit_cost` — `backend/app/models/inventory.py:76–98`.
- StockEntry per (location, product, batch) — real-time on-hand quantities.
- **PriceList per customer** — negotiated B2B pricing.
- `Customer.balance`, `Customer.credit_limit` — financial relationship snapshot per account.

**Concrete attacker actions.** Exfiltrate price_lists + customers → know exactly what each B2B account pays per SKU → undercut top 20 accounts by 3%. Exfiltrate StockEntry → production cadence, expiry pressure, warehouse footprint → time competing flash sales. Customer lat/long → poach customers by proximity to competing distributors.

### A5. The ability to ship code to production

- CI: `.github/workflows/ci.yml` — lint, test, build on push to main.
- Deploy: `.github/workflows/deploy.yml` — OIDC trust → AWS role → ECR push → SSM RunShellScript on a single EC2.
- **Images tagged `:latest`** (`deploy.yml:44, 55, 66`); not pinned by digest at pull time.
- No image signing today. No admission-time verification.
- Pre-deploy RDS snapshot. No automated rollback (comment at `deploy.yml:194`: "Automated rollback removed — restarting :latest images is not a true rollback").

**Concrete attacker actions.** Compromise the CI's AWS role → push malicious `:latest` → next legitimate deploy pulls and runs it. Compromise a developer's GitHub account → push to main → ~10-minute pipeline to prod. EC2 metadata SSRF → SSM Parameter Store read → all production secrets.

### A6. Secrets

- `.env.example` names: `DATABASE_URL` (with password), JWT `SECRET_KEY` (placeholder `change-me-in-production` — must verify rotated in prod), Paymob keys, WhatsApp keys, Sentry tokens.
- Runtime in prod: AWS SSM Parameter Store, fetched by `infra/scripts/fetch-ssm-env.sh` at container start.
- No committed secrets found in the scan.

**Concrete attacker actions.** SSM Parameter Store read from compromised EC2 IAM role → all secrets. `JWT SECRET_KEY` leak → forge any user's access token without a DB query. `PAYMOB_HMAC_SECRET` leak → see A2.

## Trust boundaries

Lines where trust, network, authority, or code-origin changes. Each one is an attack surface.

| ID | Boundary | What changes |
|---|---|---|
| **B1** | Customer browser ↔ Ingress | Network (internet→cluster), authority (unknown user→our platform) |
| **B2** | Paymob ↔ Ingress (webhook path) | Network + origin (third-party initiates, no human session) |
| **B3** | Ingress ↔ App namespace pods | Network (edge→workload), TLS-terminated trust |
| **B4** | App namespace ↔ Platform namespace | Privilege (workload→controllers), team (app→platform) |
| **B5** | App namespace ↔ Observability namespace | Origin (write-only telemetry path) |
| **B6** | Cluster ↔ Managed services (RDS, Vault, PagerDuty) | Network (cluster→AWS-managed), authority (we don't run those) |
| **B7** | Developer ↔ GitHub | Origin (human laptop → cloud platform) |
| **B8** | GitHub Actions ↔ ECR/registry ↔ deploy | Origin (CI runner → registry), privilege (writes prod artifacts) |
| **B9** | Argo CD ↔ Cluster (apply) | Privilege (controller has wide cluster-write authority) |

Two boundaries are walked in depth below: **B2** (highest concentration of present-day evidence, ties to the trigger event in problem-statement) and **B8** (highest single-compromise blast radius). The remaining seven are stubbed in [§Remaining boundaries](#remaining-boundaries-stubbed-for-later).

## B2 — Paymob → Ingress (webhook path)

The boundary the problem-statement trigger event landed at. Walked end-to-end across all six STRIDE letters.

### B2:S — Spoofing

**Threat.** An attacker forges a Paymob webhook to mark a fake order as PAID without paying.

**Vector.** POST to `/api/v1/webhooks/paymob` with a crafted JSON and a fabricated `hmac` query parameter. Enabling conditions: (a) `PAYMOB_MODE=log` mistakenly set in prod — `verify_webhook_hmac` returns `True` unconditionally (`backend/app/services/paymob_service.py:93–94`); (b) `PAYMOB_HMAC_SECRET` leaks via SSM Parameter Store compromise, EC2 metadata SSRF, or a departing employee with stale AWS access.

**Current mitigation.** HMAC-SHA512 with `hmac.compare_digest` (constant-time). Idempotency by `transaction_id` against PaymentLog. Amount validated against `order.total` before status flip (`backend/app/api/webhooks/paymob.py:88–111`).

**Residual risk — High, config-driven.** One wrong env var or one leaked secret → silent fraud. Platform must close with: (1) Kyverno admission policy refusing pods that carry `PAYMOB_MODE=log` outside the dev namespace, (2) External Secrets + short-lived rotation backed by Vault, (3) out-of-band reconciliation comparing Paymob's daily settlement report against our ledger, (4) network-layer source filter if Paymob publishes a stable IP allowlist.

### B2:T — Tampering

**Threat.** Webhook payload modified in transit to fake a successful payment or change the amount.

**Vector.** TLS termination at ingress defeats most network MITM. The realistic vector is **in-cluster traffic** between ingress and the backend pod — currently plaintext, no mTLS, no NetworkPolicy enforcement; a malicious sidecar or compromised same-namespace pod could read or modify.

**Current mitigation.** HTTPS to ingress. The amount-validation step in the handler defends even a tampered `amount_cents` field because the order total is looked up server-side.

**Residual risk — Medium.** Defended at the app layer by amount validation, but the in-cluster "any pod talks to any pod" assumption needs Phase 6 NetworkPolicies default-deny + cert-manager mTLS to actually be true.

### B2:R — Repudiation

**Threat.** In a payment dispute we cannot reconstruct what happened — only that *some* webhook caused *some* state change.

**Vector.** (a) Customer claims "I paid but you didn't credit me" — we have PaymentLog rows but no raw Paymob payload persisted to prove what was received. The full body exists only in transient `logger.info(...)` strings, not in a durable store. (b) Someone with DB write access flips `order.payment_status` directly — the resulting row is indistinguishable from a legitimate webhook write (both attributed to `SYSTEM_USER_ID`).

**Current mitigation.** Partial. PaymentLog records `transaction_id`, amount, and a notes field. Refresh tokens stored hashed. But the raw webhook body + HMAC header are not persisted, and every webhook-driven write looks identical.

**Residual risk — Medium-high in B2B disputes** where chargebacks have teeth. We are dependent on Paymob's records to defend ourselves — third-party evidence dependency. Platform must: (1) persist raw payload + HMAC header + receipt timestamp in an append-only audit table keyed by `transaction_id`, (2) give each webhook receipt a correlation ID that PaymentLog rows trace back to, (3) export GHA + Argo CD audit logs to an external append-only store (S3 with Object Lock).

### B2:I — Information disclosure

**Threat.** Webhook responses leak operational state to an attacker who possesses (or has guessed) a valid HMAC signature.

**Vector.** With `PAYMOB_HMAC_SECRET` compromised (per B2:S), an attacker enumerates by crafting valid HMACs for arbitrary `(transaction_id, paymob_order_id, amount)` tuples and reading the distinguishable response: `{"status": "duplicate"}` reveals which transaction_ids exist; `{"status": "ignored", "reason": "order not found"}` reveals which paymob_order_ids exist; `{"status": "rejected", "reason": "amount mismatch"}` allows binary search on the order total.

**Current mitigation.** None at the handler. HMAC verification is the only gate; once past it, the response is verbose.

**Residual risk — Low standalone, but amplifies B2:S** from "forge one payment" to "map the entire payment history." Platform must: (1) collapse webhook responses to a single opaque `{"status": "received"}` regardless of internal outcome (log the real outcome internally), (2) treat the handler as a write-only sink from the caller's perspective.

### B2:D — Denial of service

**Threat.** Webhook endpoint flooded → backend pod CPU exhausted, DB connection pool saturated, Celery queue depth explodes → real Paymob webhooks miss the 60s reconciliation SLO ([slos.md](slos.md) CUJ-2).

**Vector.** (a) Anonymous flood — every request still incurs JSON parse + HMAC compute + 401 round-trip; bulk small payloads at high rate. (b) Authenticated flood with leaked HMAC — each request runs the full path: ~4 DB roundtrips, PaymentLog insert, Order update, Celery enqueue. (c) Slowloris-style — hold connections open with no body.

**Current mitigation.** TLS termination at ingress. No per-IP rate limit on the webhook path in current Ahoy. No body-size limit. Single EC2 — no horizontal scaling.

**Residual risk — High.** Directly burns CUJ-2 error budget. Platform must: (1) ingress rate limit per source IP with Paymob's published IPs allowlisted at higher quota, (2) request body size limit (≤1KB on the webhook path), (3) HPA + PDBs on backend pods, (4) bounded Celery queue with a dead-letter queue and worker autoscaling, (5) circuit-break the handler when queue depth exceeds threshold (return 503 fast → Paymob retries → buys recovery time).

### B2:E — Elevation of privilege

**Threat.** A forged webhook escalates an outside attacker to the equivalent of a privileged system actor — able to mark arbitrary orders PAID, insert PaymentLog rows, trigger WhatsApp notifications, and (downstream) trigger ledger writes, all attributed to `SYSTEM_USER_ID` with no traceability to a real principal.

**Vector.** With HMAC compromised (per B2:S), POST a crafted webhook for any `(order_id, amount)` the attacker chooses → handler runs the full privileged write path. There is **no per-request authorization** — HMAC is treated as both authentication and authorization for any state transition the webhook supports.

**Current mitigation.** HMAC is the sole gate. Amount validation prevents "mark a $1000 order paid for $0.01." Idempotency by `transaction_id` prevents trivial replay.

**Residual risk — High.** Platform must: (1) least-privilege `SYSTEM_USER_ID` — its role allows only `payment_status: UNPAID|PARTIALLY_PAID → PAID` and the specific PaymentLog write; **never** ledger reversals, amount changes, or arbitrary order edits, (2) out-of-band confirmation — on first receipt of a transaction_id, call back to Paymob's API to confirm "did you process X?" before committing the status flip; this collapses forgery from "one secret leak" to "one secret leak AND fool the outbound Paymob call."

## B8 — GitHub Actions → registry → cluster (the supply chain)

Whoever owns any link of this chain effectively owns production. No defense-in-depth past it.

### B8:S — Spoofing

**Threat.** An attacker convinces the cluster (or a step in the chain) to run code as if it were ours.

**Vector.** (a) Steal a developer's GitHub credentials → push to main → CI builds and deploys malicious code attributed to their identity. (b) Exploit a misconfigured OIDC `sub` claim (the classic `sub:*` mistake) → AWS issues a build-role token to a malicious fork → push malicious image to ECR. (c) Without image signing, any actor with ECR push permission overwrites `:latest` with a malicious image; current Ahoy state.

**Current mitigation.** GitHub auth (assume 2FA enforced). AWS OIDC trust policy with `sub` claim (verify it pins to `repo:org/repo:ref:refs/heads/main`, not a wildcard).

**Residual risk — High** until Phase 6 adds Cosign-keyless signing + admission-time verification. Until then the registry is trust-on-first-use and image tags are mutable. Platform must: (1) signed commits on main, (2) verify OIDC `sub` is pinned, (3) Cosign keyless signing in CI + Kyverno `verifyImages` policy at admission refusing unsigned or wrong-signer images, (4) image references in manifests **by digest, not by tag**.

### B8:T — Tampering

**Threat.** Source code, image bytes, or deployment config altered between developer intent and pod execution.

**Vector.** (a) Push directly to main bypassing review (branch protection off or admin override). (b) Third-party action consumed via `uses: foo/bar@v1` — `@v1` is a moving tag, not a SHA; upstream maintainer compromise propagates. (c) Mutable `:latest` tag overwritten in the registry. (d) Config repo has no commit signing; tampering at the config-repo level isn't detected.

**Current mitigation.** Assume branch protection on main and PR review. Current Ahoy `ci.yml`/`deploy.yml` do **not** pin third-party actions by SHA — a real gap.

**Residual risk — Medium-high for actions, High for unsigned images.** Platform must: (1) pin every third-party action by SHA (`uses: actions/checkout@<sha>`), (2) Cosign signature on every image + Kyverno verification at admission, (3) Argo CD configured to verify GPG-signed commits on the config repo, (4) image references in manifests by digest only.

### B8:R — Repudiation

**Threat.** After an incident, the chain "commit → image → deployment" cannot be reconstructed with high confidence.

**Vector.** (a) Git commits unsigned → attribution to a developer is just a string. (b) GHA logs are not append-only or independently signed → a GitHub org owner can rerun or delete a workflow run, removing forensic evidence. (c) No build provenance attestation → cannot prove which commit + which workflow run produced which image digest.

**Current mitigation.** Git history. GHA audit logs (default 90-day retention). ECR records `imageDigest` and `imagePushedAt`.

**Residual risk — Medium, defensible at moderate cost.** Platform must: (1) require signed commits on main, (2) **SLSA-style build provenance attestations** signed by Cosign — bind `(commit_sha, workflow_run_id, builder_identity, image_digest)` at build time, verify at admission, (3) export GHA audit logs to S3 with Object Lock, (4) preserve Argo CD sync history beyond default rotation.

### B8:I — Information disclosure

**Threat.** Secrets or proprietary metadata leak out of the CI pipeline.

**Vector.** (a) `echo $SECRET_X` in a debug step — GHA masking only catches secrets declared via `secrets:`, not arbitrary derivations. (b) Image layer accidentally bakes a `.env` file (no `--secret` mount in Dockerfile). (c) **Third-party actions** receive the workflow's secrets and OIDC token by design — a compromised action is a full secret compromise. (d) ECR repository accidentally set public.

**Current mitigation.** GHA secret masking. OIDC eliminates long-lived AWS keys. Ahoy uses `secrets.X` references in workflows.

**Residual risk — Medium.** Platform must: (1) `--secret` mount pattern in all Dockerfiles, never `COPY .env`, (2) third-party action allowlist (pinned, audited set only), (3) gitleaks in CI as the second-line backstop, (4) ECR repo policies explicitly private and reviewed in PR.

### B8:D — Denial of service

**Threat.** Supply chain unavailable when we most need to deploy: during an incident.

**Vector.** (a) GitHub outage — no commits, no CI, no merge. (b) ECR regional outage — no image pull on new pods. (c) Argo CD pod unhealthy or source repo unreachable — drift uncorrected. (d) ECR storage quota exhausted from accumulated images with no lifecycle policy.

**Current mitigation.** ECR multi-AZ. Kubelet image cache means running pods survive a registry outage. Pre-deploy RDS snapshot in current Ahoy.

**Residual risk — Medium**, mostly inherited from SaaS/AWS. Platform should: (1) image lifecycle policy (delete untagged, keep last N tagged), (2) Argo CD HA replicas, (3) documented break-glass deploy path (manual `kubectl apply` with verifiable image digest, bypassing Argo CD when Argo CD itself fails), (4) digest-pinned manifests mean restarted pods can run from kubelet cache even without registry availability.

### B8:E — Elevation of privilege

**Threat.** A compromised step in the pipeline gains privileges that allow it to take over production.

**Vector.** (a) A pinned third-party action whose upstream SHA history was rewritten (force-push by a compromised maintainer) exfiltrates the workflow's OIDC token → assumes the AWS build role → pushes a malicious image. (b) The AWS build role has more permissions than strictly "push image X to repo Y" → blast radius widens. (c) Argo CD service account has cluster-wide write permissions by design → its compromise = cluster takeover.

**Current mitigation.** Short-lived OIDC tokens (per-workflow). Argo CD ships with cluster-admin-equivalent for its target namespaces by default.

**Residual risk — High if build role is over-permissioned; structurally high for Argo CD by design.** Platform must: (1) scope the AWS build role strictly to "push to repo X" — no other repos, no other resources, (2) Argo CD with namespace-scoped Projects (not cluster-wide), per-team RBAC, (3) Kyverno policies refusing privileged containers, hostPath, hostNetwork — bounds what a compromised Argo CD can grant, (4) default-deny NetworkPolicies so a compromised pod cannot pivot.

## Remaining boundaries (stubbed for later)

Identified but not yet walked through STRIDE in full. Top concern noted; deeper walks are scheduled for after the platform's first production cut.

- **B1 — Customer browser ↔ Ingress.** Generic web threats: SQL injection (defended by ORM use), XSS (CSP headers required), session hijack (HttpOnly + Secure cookies + short-lived JWTs), DDoS on `POST /orders` directly burning CUJ-1 SLO. First letter to walk: **D** (volumetric attack on order placement).
- **B3 — Ingress ↔ App namespace pods.** In-cluster lateral movement; without NetworkPolicies + mTLS, an ingress controller compromise reaches every app pod. First letter to walk: **E** (privilege from edge into workloads).
- **B4 — App namespace ↔ Platform namespace.** Privilege escalation via misconfigured RBAC or shared service accounts. First letter to walk: **E** (app pod escaping to Argo CD's namespace).
- **B5 — App namespace ↔ Observability namespace.** Log injection (attacker-controlled log content interpreted by downstream tooling); PII inadvertently captured in telemetry then retained in Loki/Tempo. First letter to walk: **I** (PII in traces).
- **B6 — Cluster ↔ Managed services (RDS, Vault, PagerDuty).** **Highest unwalked priority.** The ledger-immutability gap (A3) lives here: anyone with `DATABASE_URL` write access bypasses every app-level invariant. Secret leak via SSM Parameter Store or Vault compromise concentrates here too. First letter to walk: **T** (tampering with ledger via direct DB access), followed by **I** (secret exfiltration from Vault/SSM).
- **B7 — Developer ↔ GitHub.** Developer laptop compromise → push to main. Partially covered by B8:S/R; standalone walk addresses developer environment hardening (SSH key handling, signed commit tooling, MFA enforcement).
- **B9 — Argo CD ↔ Cluster.** Argo CD's service account = cluster-admin-equivalent within its target namespaces. Partially covered by B8:E; standalone walk addresses Argo CD-specific controls (Project scoping, sync windows, declarative RBAC).

## Top risks (rolled up)

The five rows future-you should care about first. Each is grounded in a STRIDE row above; this list is the executive view.

1. **Config-driven HMAC bypass (B2:S).** `PAYMOB_MODE=log` returning `True` unconditionally in `verify_webhook_hmac` means one misconfigured env var collapses webhook authentication. **Mitigation: Kyverno admission refusing the env var in prod namespaces.**
2. **Ledger immutability is app-only (A3, B6).** Direct DB UPDATE/DELETE on `journal_lines` bypasses every "immutability" the application enforces. Anyone with `DATABASE_URL` write access commits silent fraud. **Mitigation: separate DB user for app vs admin; per-table DB-level append-only or trigger guards; audit-log table written by trigger.**
3. **Unsigned, mutable image tags (A5, B8:S/T).** Current Ahoy `:latest` pattern means whoever owns ECR push owns production. **Mitigation: Cosign keyless signing + Kyverno `verifyImages` + digest-pinned manifests.**
4. **Forged webhook = privileged system identity (B2:E).** SYSTEM_USER_ID inherits broad write authority; a single successful forgery (per #1) commits arbitrary order/ledger state changes. **Mitigation: scope SYSTEM_USER_ID to UNPAID→PAID only; out-of-band Paymob confirmation on first receipt.**
5. **No raw webhook audit trail (B2:R).** Disputes cannot be reconstructed because the raw Paymob payload is never persisted; we depend on Paymob's records to defend ourselves. **Mitigation: append-only audit table keyed by transaction_id, with full payload + HMAC + receipt timestamp.**

## Decisions this threat model justifies

Traceability between the residual-risk rows and the platform's specific Phase 2–6 decisions. Every item on the right has at least one row above that *requires* it.

| Platform decision | Phase | Justifying rows |
|---|---|---|
| Cosign keyless signing in CI | P3 | B8:S, B8:T, B8:R |
| Kyverno admission + `verifyImages` | P6 | B8:S, B8:T |
| Digest-pinned image references in manifests | P5/P6 | B8:S, B8:T |
| SHA-pinned third-party GHA actions | P3 | B8:T |
| SLSA-style build provenance attestations | P6 | B8:R |
| External Secrets Operator + Vault | P6 | A6, B2:S, B8:I |
| Default-deny NetworkPolicies | P6 | B2:T, B3, B8:E |
| mTLS via cert-manager | P6 | B2:T |
| Argo CD namespace-scoped Projects | P3 | B8:E, B9 |
| Append-only Paymob webhook audit table | App-level (Phase 2 backend change) | B2:R |
| Out-of-band Paymob reconciliation job | App-level (Phase 2 backend change) | B2:S, B2:E, top risk #4 |
| Least-privilege `SYSTEM_USER_ID` role | App-level | B2:E |
| Ingress rate limit + body size limit on webhook | P2/P3 | B2:D |
| HPA + PDBs + bounded Celery queue | P5 | B2:D |
| ECR image lifecycle policy | P3 | B8:D |
| Audit-log export to S3 Object Lock | P4 | B2:R, B8:R |
| Signed commits enforced on main | P3 | B8:R, B8:S |

## Review and lifecycle

- **Re-walk B6 within the first quarter of production data.** The ledger-DB boundary is the highest-priority unwalked area; deferred only because the platform's DB controls (PgBouncer, role separation, audit triggers) are not yet decided.
- **Re-walk other boundaries** after each major architecture change.
- **Post-incident review.** Every Sev-1 or Sev-2 incident produces an audit of the relevant boundary's row: does the threat row need updating? Did the residual-risk language match what actually happened?
- **Annual.** Reconcile this document against the architecture diagram. Drift between them is a defect in either or both.
