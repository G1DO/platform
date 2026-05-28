# ADR-002: GitOps controller — Argo CD

## Status

Accepted — 2026-05-28.

## Context

The platform needs a GitOps controller — a cluster-resident agent that reads desired state from Git and reconciles the cluster to match. This decision is load-bearing for Phases 3, 5, and 6: it is the surface against which the CI pipeline writes deployments, the progressive-delivery controller integrates, and the audit/forensic story in [threat-model.md](../threat-model.md) lives.

GitOps replaces the current Ahoy push-based deploy (`aws ssm send-command` from GitHub Actions, [threat-model.md §B8](../threat-model.md)) with a pull-based model: CI no longer holds cluster-write credentials. The cluster watches a config repository and applies whatever is there. Compromising CI compromises the image registry; it does not compromise the cluster directly.

Constraints driving this decision:

- **Threat-model requirements** ([threat-model.md §B8:R, B8:E](../threat-model.md)): signed-commit verification on the config repo, namespace-scoped controller permissions, full audit of who deployed what when, no cluster-wide credentials in CI.
- **Phase 5 progressive delivery.** The GitOps controller must integrate tightly with the progressive-delivery controller (Argo Rollouts or Flagger). Phase 5 is where automated rollback on SLO regression lands; the integration must be first-class, not glued.
- **One-person platform team.** Visual artifacts matter — they become the interview demo and the on-call dashboard. A UI is not optional.
- **Multi-cluster is not a requirement at Phase 1** (single MENA region, per [problem-statement non-goal #1](../problem-statement.md#non-goals)) but must not be foreclosed.
- **Local-cluster compatibility.** Per [ADR-001](001-local-cluster-kind.md), everything must run in a kind cluster for dev and CI.

## Decision

We will use **Argo CD**, deployed in-cluster, in a dedicated `argocd` namespace.

Deployment pattern: **two repositories** —

- **App repo** — Ahoy source code (backend, storefront, dashboard), Dockerfiles, CI workflows. CI builds and pushes images to the registry.
- **Config repo** — Kubernetes manifests, Helm chart values, ApplicationSet definitions, image-digest pins. CI also writes a Pull Request to this repo bumping image digests after a successful build; merge triggers Argo CD to apply.

Argo CD watches the config repo, verifies signed commits, and reconciles the cluster to match. Drift is auto-corrected (or alerted, depending on the Application's `syncPolicy`).

## Alternatives considered

- **Flux v2.** Rejected after honest comparison, not on technical grounds. Flux is equally mature (CNCF graduated), more modular (separate `GitRepository`/`Kustomization`/`HelmRelease`/`ImagePolicy` controllers), has built-in image-update automation (no `Argo CD Image Updater` needed), and has a smaller resource footprint. **Why not Flux:** (a) no first-party UI; Weave GitOps is a separate product with its own license model. The platform's interview-defensibility benefits materially from Argo CD's visual diff. (b) Flux pairs with **Flagger** for progressive delivery; Argo CD pairs with **Argo Rollouts**. Picking Flux means Phase 5 becomes a Flagger ADR, which is fine, but the integration story is more cohesive when both controllers are Argo-family — fewer abstractions to bridge. (c) Industry signal: more interviewers recognize Argo CD; for a CV-piece project this matters.
- **Argo CD hub-and-spoke (single Argo CD managing many clusters).** Deferred, not rejected. Phase 1 has one cluster; running Argo CD in-cluster minimizes blast radius from a control-plane compromise. If multi-cluster materializes, an ApplicationSet pattern on a dedicated management cluster is the upgrade path.
- **Single-repo (app + manifests together).** Rejected. Every code commit becomes a deployment commit; you cannot revert a deployment without reverting code, and break-glass deploys (hand-edit the manifest) become impossible without touching app source. The two-repo pattern is what makes "roll back deploy without rolling back code" a routine operation.
- **Manual `kubectl apply` (no GitOps).** Rejected. Defeats the purpose of the platform — no audit, no review, no drift detection, no rollback. Mentioned only to make the pull-based design explicit.
- **Push-based CI deploy (the current Ahoy pattern: `aws ssm send-command`).** Rejected. The threat model B8 walk shows the consequences: CI holds production credentials, the deploy is a side effect of CI rather than a reviewable artifact, and there is no continuous reconciliation. Replacing this pattern is one of the explicit goals of the platform.

## Consequences

**Positive.**

- CI no longer has cluster-write credentials. The Phase 3 supply-chain blast radius (per [threat-model.md §B8:E](../threat-model.md)) shrinks to "compromise the registry" rather than "compromise production."
- Every deployment is a Pull Request with diff, review, and audit trail. The repudiation gap called out in [threat-model.md §B8:R](../threat-model.md) becomes closeable: Git history + Argo CD sync history together name the responsible commit, builder, and reconciliation event.
- Argo CD's UI is a defensible interview artifact — visual diff between desired and actual state, sync history, deployment timeline.
- Tight integration with **Argo Rollouts** (ADR-006 candidate) gives Phase 5 a coherent "GitOps + progressive delivery + auto-rollback" story without bridging two ecosystems.
- Drift detection: anyone making a `kubectl edit` in production is either reverted automatically or surfaced as drift — incident-investigation evidence.

**Negative.**

- More controllers running in-cluster than Flux would require (Argo CD's own components, plus Argo Rollouts when added, plus Argo CD Image Updater if we ever want auto-image-bumping). Resource overhead is small but non-zero on a kind cluster.
- The config repo becomes a **hot dependency**. Argo CD cannot deploy if it cannot reach the config repo; this is captured in [threat-model.md §B8:D](../threat-model.md). The break-glass procedure (manual `kubectl apply` with verifiable image digest) must be documented and rehearsed.
- Argo CD's default RBAC ships with broad permissions on its target namespaces; namespace-scoped Projects must be configured deliberately (per [threat-model.md §B8:E](../threat-model.md), and likely a future ADR).
- Picking Argo CD weakly constrains downstream choices toward the Argo family (Rollouts, Workflows). This is a feature for cohesion and a constraint for flexibility.

**Follow-up work this commits us to.**

- Phase 3 ships: Argo CD Helm install pinned to a tested version; two-repo skeleton with one example Application; the GHA workflow that writes a PR to the config repo on successful image build.
- Namespace-scoped `AppProject` definitions, one per app team (per [threat-model.md §B8:E](../threat-model.md)).
- Signed-commit verification on the config repo (per [threat-model.md §B8:R](../threat-model.md)). Non-negotiable.
- Documented break-glass procedure: how to manually `kubectl apply` an image-digest-pinned manifest when Argo CD itself is unhealthy or the config repo is unreachable.
- Phase 5 picks Argo Rollouts (subject to its own ADR) as the progressive-delivery controller.
- Sync history retention configured beyond the default rotation, exported to the observability stack (per [threat-model.md §B8:R](../threat-model.md)).
