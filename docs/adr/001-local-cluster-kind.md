# ADR-001: Local cluster — kind

## Status

Accepted — 2026-05-28.

## Context

The platform project needs a local Kubernetes distribution that runs on a developer machine and in GitHub Actions for integration tests. Every Phase 2–6 artifact (Argo CD, Kyverno admission policies, NetworkPolicies, Argo Rollouts analysis, Chaos Mesh experiments) must be exercised in this local cluster before reaching the managed prod environment (EKS, per [architecture.md](../architecture.md)).

Constraints driving this decision:

- **Production parity.** The cluster is the test bed for admission control, signature verification, and NetworkPolicy enforcement. API drift from upstream Kubernetes (custom defaults, missing extension points, opinionated ingress) creates false confidence — local tests pass, prod fails.
- **Multi-node.** PDBs, HPA scheduling, node-failure chaos experiments, and pod-to-pod NetworkPolicies all require ≥2 worker nodes to exercise meaningfully.
- **Fast iteration.** Phase 6 chaos work assumes cheap teardown/rebuild. A 5-minute cluster boot kills iteration discipline.
- **CI compatibility.** Integration tests run in GitHub Actions; the local distribution must spin up there with the same config used on a laptop.
- **One developer at this stage.** Workstation is Linux with sufficient RAM; no GPU need; no Windows constraints.

## Decision

We will use **kind** (Kubernetes IN Docker) for the local cluster, configured with 1 control-plane node and 2 worker nodes.

A single `kind-cluster.yaml` config in the repository is the source of truth, used identically by `make cluster-up` on a laptop and by the GitHub Actions integration-test workflow.

## Alternatives considered

- **minikube.** Rejected. Heavier (defaults to VM driver, slower boot). The add-on ecosystem (`minikube addons enable ingress`) introduces project-specific defaults that drift from upstream behavior; tests passing locally can fail in production for reasons unrelated to the code under test.
- **k3d (k3s in Docker).** Rejected despite being faster and lighter than kind. k3s ships with opinionated defaults (Traefik as ingress, sqlite as the default datastore in minimal mode, klipper-lb as service load balancer) that diverge from upstream Kubernetes. The whole platform would need to paper over those differences to remain prod-realistic, and the gain in startup time isn't worth that maintenance cost.
- **k3s alone.** Rejected for the same reasons as k3d, plus more setup overhead. k3s is optimized for edge/IoT, not "production-similar dev cluster."
- **Docker Desktop's bundled Kubernetes.** Disqualified — single-node only, fails the multi-node constraint.
- **microk8s.** Rejected. Snap-packaged; heavier; stronger fit for Ubuntu production hosts than for dev iteration.

## Consequences

**Positive.**

- API parity with managed Kubernetes — Kyverno, Argo CD, NetworkPolicies, Gateway API behave the same in dev and prod.
- Cluster spins up in ~60 seconds; teardown and recreate is cheap enough that chaos experiments can run on ephemeral clusters.
- First-class GitHub Actions support via `helm/kind-action`; the CI integration-test path uses the same config as local development.
- Multi-node native — `nodes:` block in the kind config gives PDB, HPA, and NetworkPolicy behavior that actually mirrors production.

**Negative.**

- Hard dependency on Docker as the container runtime. Contributors using Podman-only environments need a shim or to switch.
- Kind node images are larger than k3s alternatives (~500MB vs ~100MB) — minor disk cost, but a real download on first use.
- No first-class GPU support. Not relevant to this project but worth noting if scope expands.

**Follow-up work this commits us to.**

- Ship `kind-cluster.yaml` and `make cluster-up`/`make cluster-down` targets as part of the Phase 2 walking skeleton.
- The GHA integration-test workflow consumes the same config — divergence between dev and CI is a defect.
- Future ADRs that pick admission controllers, ingress, GitOps controllers, and observability must verify their component runs cleanly in this kind config; if it doesn't, either the component pick or the kind config changes — but the parity constraint stands.
