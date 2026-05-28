# ADR-004: Admission policy — Kyverno

## Status

Accepted — 2026-05-28.

## Context

Admission control is the last gate between "a manifest was applied" and "the workload runs in the cluster." It is where the threat-model's most concrete platform requirements close — image signature verification, prohibition of dangerous configurations (privileged pods, hostPath mounts), and enforcement of safe-by-default invariants (PodSecurity standards, resource limits, NetworkPolicies). Without an admission controller, [ADR-003](003-image-signing-cosign-keyless.md) is decorative — signatures are produced but nothing prevents an unsigned image from running.

This ADR picks the admission controller. It is intentionally drafted together with [ADR-003](003-image-signing-cosign-keyless.md); they must land together or the platform produces signatures with no enforcement.

Constraints driving this decision:

- **Threat-model requirements**: closes [§B8:S, §B8:T](../threat-model.md) (image signature verification), top-risk #1 (no `PAYMOB_MODE=log` in prod namespaces), [§B8:E](../threat-model.md) (no privileged containers / hostPath / hostNetwork), [§B3](../threat-model.md) (require NetworkPolicy per namespace).
- **Declarative and reviewable.** Policies must live in the config repo (per [ADR-002](002-gitops-argo-cd.md)) and be diffable in PR review like everything else.
- **Native Cosign keyless verification.** The verifier must understand OIDC issuer + subject pattern matching, not just public-key trust.
- **kind compatible** ([ADR-001](001-local-cluster-kind.md)).
- **One platform engineer maintaining this.** Policy authoring ergonomics — the cost of writing the tenth policy must not be 5× the cost of the first.

## Decision

We will use **Kyverno** as the admission controller, deployed in a dedicated `kyverno` namespace, managed by Argo CD with a sync wave that ensures Kyverno is reconciled before any policy-protected workload.

Policies are Kubernetes `ClusterPolicy` and `Policy` resources stored in the config repo under `policies/`. The Phase 1 catalog is documented in [§Initial policy catalog](#initial-policy-catalog) below.

## Alternatives considered

- **OPA Gatekeeper.** Equally powerful, equally mature (CNCF graduated, longer in market). Rejected on **ergonomics**, not capability: Gatekeeper policies are written in Rego, which is a full general-purpose policy language with a real learning curve. For a small platform team, Kyverno's "policies are Kubernetes YAML" wins because (a) PR review of policy changes uses the same diff tooling as every other manifest, (b) the on-call engineer reading a policy at 3am during an incident does not need to be a Rego practitioner, (c) Kyverno's `verifyImages` for Cosign keyless is first-class and well-documented, where Gatekeeper relies on external data providers for the same. Gatekeeper would be the right pick at a larger scale where dedicated security engineering authors policies — not our case.
- **Kubernetes-native Validating Admission Policy (VAP).** Promising long-term direction — CEL-based, no extra controller to run, no extra failure mode. Stable in Kubernetes 1.30+. Rejected for **now** because (as of 2026) there is no native equivalent of `verifyImages` for Cosign keyless verification — you would still need Kyverno or Gatekeeper for that specific capability. The plan: when CEL-based image verification lands natively and matures, this ADR is supersedable. The decision today is "use one controller for everything" rather than "use VAP for some things and Kyverno for image verification" — two controllers is two failure modes and two policy languages, worse than either alone.
- **jsPolicy, Polaris, Datree.** Rejected. Smaller ecosystems, narrower capabilities, less alignment with the CNCF stack the rest of the platform is built on. Not capable of the full Phase 1 policy catalog without supplementing with one of the above.
- **No admission policy (rely on RBAC + workload-level security contexts).** Rejected. Defeats the point of [ADR-003](003-image-signing-cosign-keyless.md) — unsigned images would run freely. The whole class of "admission-time defenses" closes nothing under this option.

## Consequences

**Positive.**

- Closes the highest-leverage threat-model rows: image signature verification ([§B8:S, §B8:T](../threat-model.md)), config-driven HMAC bypass (top-risk #1), privileged-container family ([§B8:E](../threat-model.md)), missing NetworkPolicies ([§B3](../threat-model.md)).
- Policies are Kubernetes resources — reviewed, diffed, versioned, and audited like every other manifest. The same Argo CD sync that ships the workload ships the policy.
- Kyverno's reporting CRDs (`ClusterPolicyReport`) give a structured view of policy violations — feeds into the observability stack ([ADR-005](005-observability-lgtm-self-hosted.md)) for "are we trending toward more violations or fewer."
- First-class `verifyImages` for Cosign keyless — the verifier configuration matches OIDC issuer + subject pattern, completing [ADR-003](003-image-signing-cosign-keyless.md).

**Negative.**

- Kyverno itself is a new in-cluster controller. Its compromise = bypass of every admission rule. Mitigated by: minimal RBAC (Kyverno's service account is least-privileged), namespace isolation (Kyverno's pods cannot be tampered by workload namespaces under default-deny NetworkPolicies), and Kyverno being one of the resources Argo CD verifies via signature on its own manifests.
- **Chicken-and-egg.** Argo CD must apply Kyverno *before* any policy-protected workload, or the protected workloads fail to admit. Argo CD `sync-wave` annotations handle this: Kyverno is in wave -10, policies in wave -5, workloads in wave 0.
- Kyverno's "ClusterPolicy + Policy" distinction needs care — getting cluster-scoped vs namespace-scoped wrong produces silent policy gaps. Code review of policy PRs must explicitly verify scope.
- Policy mutation (Kyverno can rewrite resources at admit time) is a powerful footgun. The Phase 1 catalog uses **validation only**; mutation is deferred to a later ADR with explicit consideration of what it changes silently.

**Follow-up work this commits us to.**

- Phase 3 ships Kyverno installed via Helm chart (pinned version), with Argo CD sync-waves configured.
- Phase 6 ships the full initial policy catalog (below).
- Every new platform decision that creates a new attack surface must include a policy in the ADR's follow-up section. The traceability table in [threat-model.md](../threat-model.md) is the master list.
- Operational runbook: how to disable a specific policy temporarily (incident response), with audit trail.

## Initial policy catalog

The Phase 1 policies, each closing a specific threat-model row.

| Policy | Type | Closes |
|---|---|---|
| `verify-images-cosign` | Validate | [§B8:S, §B8:T](../threat-model.md) — every container image must have a valid Cosign signature from `https://github.com/<org>/<repo>/.github/workflows/<workflow>.yml@refs/heads/main` |
| `disallow-paymob-log-mode-in-prod` | Validate | Top-risk #1 — reject any Pod with env var `PAYMOB_MODE=log` outside the `dev` namespace. Named after the Ahoy incident's exact vector; the workload exposes the same footgun deliberately ([ADR-006](006-workload-go-shim.md)) so this policy refuses a real misconfiguration at admission. |
| `disallow-privileged-containers` | Validate | [§B8:E](../threat-model.md) — reject Pods with `securityContext.privileged: true` |
| `disallow-host-namespaces` | Validate | [§B8:E](../threat-model.md) — reject `hostNetwork`, `hostPID`, `hostIPC` |
| `disallow-hostpath-volumes` | Validate | [§B8:E](../threat-model.md) — reject `hostPath` volume mounts |
| `require-resource-limits` | Validate | Operational — every container has CPU + memory requests and limits |
| `require-non-root` | Validate | [§B8:E](../threat-model.md) — `runAsNonRoot: true` + `readOnlyRootFilesystem: true` |
| `require-network-policy-per-namespace` | Validate | [§B3](../threat-model.md) — every workload namespace must declare at least one NetworkPolicy |
| `require-image-digest-not-tag` | Validate | [ADR-003](003-image-signing-cosign-keyless.md) — image references must use `@sha256:...`, never `:tag` |
| `require-ownership-label` | Validate | Operational — every namespace declares a `team/owner` label for on-call routing ([slos.md](../slos.md)) |
