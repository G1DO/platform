# Architecture

Target architecture for the self-healing platform that hosts Ahoy at projected post-Series-A scale (see [problem-statement.md](problem-statement.md) and [slos.md](slos.md)). This is the **Phase 6 end state**; each piece is labelled with the phase that introduces it so the same diagram doubles as a roadmap.

## How to read this diagram

- **Subgraphs are boundaries**, not just groupings. Each subgraph border represents a change in trust, network, or authority.
- **Solid arrows** = synchronous request/response on the data path.
- **Dashed arrows** = asynchronous, control-plane, or telemetry flow.
- **`[Pn]` tags** = the phase that introduces this component.

## Diagram

```mermaid
flowchart LR
    subgraph external["External (untrusted)"]
        customer["Customer Browser"]
        paymob["Paymob Gateway"]
        dev["Developer"]
    end

    subgraph supply["Supply Chain (P3)"]
        github["GitHub<br/>(app + config repos)"]
        ghactions["GitHub Actions"]
        ghcr["GHCR Image Registry"]
        cosign_sign["Cosign Keyless Sign"]
    end

    subgraph cluster["Kubernetes Cluster — kind (dev) / EKS (prod) [P2]"]

        subgraph edge["Edge"]
            ingress["Ingress Controller [P2]"]
        end

        subgraph platform_ns["Platform Namespace"]
            argocd["Argo CD [P3]"]
            kyverno["Kyverno Admission [P6]"]
            cosign_verify["Cosign Verify [P6]"]
            extsecrets["External Secrets Operator [P6]"]
            rollouts["Argo Rollouts [P5]"]
        end

        subgraph app_ns["App Namespace"]
            storefront["Next.js Storefront [P2]"]
            dashboard["Next.js Dashboard [P2]"]
            backend["FastAPI Backend [P2]"]
            celery["Celery Workers [P2]"]
            redis[("Redis [P2]")]
        end

        subgraph obs_ns["Observability Namespace [P4]"]
            otel["OTel Collector"]
            prom["Prometheus"]
            loki["Loki"]
            tempo["Tempo"]
            grafana["Grafana"]
            alertmanager["Alertmanager"]
        end

        subgraph chaos_ns["Chaos Namespace [P6]"]
            chaosmesh["Chaos Mesh"]
        end
    end

    subgraph managed["Managed Services"]
        rds[("RDS Postgres [P2]")]
        vault["Vault / AWS Secrets Manager [P6]"]
        pagerduty["PagerDuty [P4]"]
    end

    %% CUJ-1: Order placement
    customer -->|HTTPS| ingress
    ingress --> storefront
    storefront --> backend
    backend --> redis
    backend --> rds
    backend -->|payment intent| paymob

    %% CUJ-2: Payment webhook to ledger
    paymob -->|webhook POST| ingress
    ingress --> backend
    backend -.->|enqueue| redis
    redis -.->|dequeue| celery
    celery --> rds

    %% Operator path (not under SLO in P1)
    dashboard --> backend

    %% Delivery pipeline
    dev -->|git push| github
    github --> ghactions
    ghactions -->|build push| ghcr
    ghactions -->|sign| cosign_sign
    cosign_sign -.-> ghcr
    github -.->|config repo poll| argocd
    argocd -.->|apply| app_ns
    argocd -.->|apply| obs_ns
    argocd -.->|apply| platform_ns
    ghcr -.->|pull on deploy| app_ns
    cosign_verify -.->|verify at admission| app_ns
    kyverno -.->|policy enforce| app_ns
    rollouts -.->|progressive rollout| app_ns

    %% Secrets
    extsecrets -.-> vault
    extsecrets -.->|inject| app_ns

    %% Observability
    backend -.->|metrics logs traces| otel
    celery -.-> otel
    storefront -.-> otel
    dashboard -.-> otel
    ingress -.-> otel
    otel --> prom
    otel --> loki
    otel --> tempo
    prom --> alertmanager
    alertmanager -.->|page| pagerduty
    prom --> grafana
    loki --> grafana
    tempo --> grafana

    %% Chaos
    chaosmesh -.->|inject faults| app_ns
```

## What each zone is for

- **External (untrusted).** Anything we don't run. All input crossing this boundary is hostile until validated.
- **Supply Chain.** Source code, build, image registry, and the signing step that makes the chain verifiable. Compromising any of these compromises everything downstream — admission-time verification ([P6]) closes that loop.
- **Edge.** TLS termination, ingress routing, and request normalization. Single entry point for both customer traffic and Paymob webhooks.
- **Platform Namespace.** Controllers that manage the cluster but don't serve customer traffic. Kept separate from app namespace so RBAC and NetworkPolicies can be tighter here.
- **App Namespace.** The Ahoy product. The only namespace where pulling code into production has direct revenue impact.
- **Observability Namespace.** Telemetry sink for everything. Isolated so a chaos experiment on app namespace can't take down the dashboards used to observe the experiment.
- **Chaos Namespace.** Separate so chaos resources are explicitly scoped and removable.
- **Managed Services.** RDS, Vault, PagerDuty — explicit "we don't run this" boundary. Failure modes and SLOs of these services bound everything inside the cluster.

## Tracing the SLO'd CUJs through the diagram

These traces are the consistency check between [slos.md](slos.md) and this diagram. If they ever stop tracing cleanly, one of the docs is wrong.

**CUJ-1: Order placement.** Customer Browser → Ingress → Next.js Storefront → FastAPI Backend → (Redis for hold, RDS for persist, Paymob for payment intent) → response back through the same path. p99 < 400ms, 99.5%, measured at Ingress.

**CUJ-2: Payment-webhook-to-ledger.** Paymob → Ingress → FastAPI Backend (validates + enqueues) → Redis → Celery Worker → RDS (ledger commit). 99.9% within 60s, measured from Ingress receipt to RDS commit.

## Phase-by-phase build order

The same boxes, sorted by when they appear. This is the diagram an interviewer would ask you to walk if they wanted to test sequencing.

- **P2 (Walking Skeleton):** kind cluster, ingress, App Namespace (one service to start, then the rest), Redis in-cluster, RDS external, one basic dashboard. *Goal: a request can reach a pod and a pod can reach RDS.*
- **P3 (Delivery Pipeline):** GitHub Actions, GHCR, Cosign signing, Argo CD with two-repo pattern, Trivy + gitleaks in CI. *Goal: merge-to-prod is automated and the artifact is signed.*
- **P4 (Observability):** OTel Collector, Prometheus, Loki, Tempo, Grafana, Alertmanager, PagerDuty wired up. Burn-rate alerts from `slos.md` go live here. *Goal: every SLO from slos.md has live measurement and live alerting.*
- **P5 (Reliability):** Argo Rollouts with automated analysis from Prometheus, PDBs, HPA, probes tuned, tested backup/restore with measured MTTR. *Goal: a bad deploy auto-rolls-back without human intervention.*
- **P6 (Security + Chaos):** Kyverno, Cosign verification at admission, External Secrets + Vault, NetworkPolicies default-deny, Chaos Mesh experiments tied to per-failure-mode postmortems. *Goal: a chaos experiment that breaks the system produces a paged alert and a known runbook.*

## Decisions deferred to ADRs

This diagram shows *what* is in the platform; it doesn't justify *why each pick over alternatives*. Those defenses live in `docs/adr/`:

- kind vs minikube vs k3d (local dev)
- Argo CD vs Flux (GitOps)
- Kyverno vs OPA Gatekeeper (admission policy)
- Cosign keyless vs key-based (signing)
- Argo Rollouts vs Flagger (progressive delivery)
- Prometheus + Loki + Tempo vs a hosted alternative (observability stack)

## Non-goals for this diagram

- **No service mesh.** Per problem-statement non-goal #5, mTLS via cert-manager + NetworkPolicies covers the threat model at current service count.
- **No multi-region.** Per problem-statement non-goal #1, single MENA region is sufficient.
- **No multi-tenant isolation.** Per problem-statement non-goal #2, one trust boundary, namespace-level RBAC.
