# ADR-003: Image signing — Cosign keyless

## Status

Accepted — 2026-05-28.

## Context

The supply chain is the highest-blast-radius attack surface in this platform ([threat-model.md §B8](../threat-model.md)). The Ahoy-incident-state baseline used mutable `:latest` tags and no signing — anyone with registry push permission could replace a production image, and the cluster had no way to detect the substitution.

The platform needs a cryptographic binding between the GitHub Actions workflow run that built an image and the image bytes themselves, verifiable at admission time without trusting the registry as authority. This ADR picks the signing mechanism; [ADR-004](004-admission-policy-kyverno.md) picks the verification mechanism. They must be designed together.

Constraints driving this decision:

- **Threat-model requirements** ([threat-model.md §B8:S, §B8:T, §B8:R](../threat-model.md)): signed images, identity bound to the CI workflow run (not a developer or a long-lived service account), audit-grade record of every signing event.
- **No long-lived signing keys if avoidable.** Long-lived keys become high-value secrets ([threat-model.md §A6](../threat-model.md)); compromise produces undetected forgeries.
- **Verifiable from a kind cluster** ([ADR-001](001-local-cluster-kind.md)) — local dev must be able to sign and verify with the same tooling that prod uses.
- **OCI-registry-native storage.** Signatures should live alongside images, not in a separate signing infrastructure that becomes another availability dependency.

## Decision

We will use **Cosign keyless signing** with the GitHub Actions OIDC token as the signer identity.

In CI, every successful image build runs:

```
cosign sign --yes <registry>/<image>@<digest>
```

The OIDC token from GHA is exchanged with **Fulcio** (sigstore's CA) for a short-lived (~10 minute) signing certificate that encodes the OIDC subject (e.g., `https://github.com/<org>/<repo>/.github/workflows/build.yml@refs/heads/main`). Cosign signs the image digest with this certificate and writes the signature, certificate, and a **Rekor** transparency-log entry to the OCI registry alongside the image.

Verification (in [ADR-004](004-admission-policy-kyverno.md)) matches the expected OIDC issuer + subject pattern, not a public key.

## Alternatives considered

- **Cosign with long-lived keys managed in AWS KMS.** Rejected. Adds a long-lived signing key as a new high-value asset; key rotation has operational friction (every rotation requires re-signing or layered trust); a leaked key produces forged signatures with no in-band detection. Keyless eliminates the asset entirely — there is no key for an attacker to steal. The argument "but Fulcio is a third-party CA" cuts both ways: Fulcio certs are ~10 minutes valid, every issuance is in Rekor, and revocation is implicit at expiry.
- **Notary v2.** Rejected. Technically capable and OCI-native, but as of 2026 the ecosystem (Kyverno integration, Argo CD docs, community examples) is materially smaller than sigstore's. Notary v2's "bring your own PKI" model adds infrastructure we explicitly do not want to run. If the broader industry shifts and Notary v2 ecosystem catches up, this ADR is supersedable.
- **Docker Content Trust (Notary v1).** Rejected — deprecated and not maintained.
- **No image signing (status quo).** Rejected. The whole reason this ADR exists. Without it, [threat-model.md §B8:S, §B8:T](../threat-model.md) reduce to "we hope no one with registry push goes rogue or gets compromised."
- **In-house signing using GPG or sigspec.** Rejected. Reinvents key management; offers no transparency log; no admission-time verification tooling.

## Consequences

**Positive.**

- **Zero long-lived signing keys.** There is no signing secret for an attacker to steal — closes the "compromised key" branch of [threat-model.md §B8:S](../threat-model.md) by removing the asset.
- **Signer identity bound to the workflow run.** A signature proves not just "someone signed this" but "GHA workflow run X on commit Y signed this." This is the cryptographic version of the build-provenance binding required by [threat-model.md §B8:R](../threat-model.md).
- **Independent audit via Rekor.** Every signing event is in a public, append-only transparency log. Forensic reconstruction of "what was signed and when" no longer depends on GitHub's audit log alone.
- **OCI-registry-native.** No separate signing infrastructure to operate or include in disaster-recovery planning.
- **First-class Kyverno integration.** `Kyverno verifyImages` natively understands Cosign keyless verification with OIDC issuer + subject patterns — see [ADR-004](004-admission-policy-kyverno.md).

**Negative.**

- **Dependency on sigstore SaaS** (Fulcio + Rekor) at sign time. If sigstore is down, builds cannot sign. Mitigation: documented break-glass procedure (manual approval of an unsigned image deployment, with explicit non-repudiation by a named human). Self-hosted sigstore is an option later if scale or sovereignty justifies it.
- **Public Rekor log.** Every signing event — including the OIDC subject (which encodes repo path and branch) — is publicly visible. For this platform's `ahoy-app` repo this is acceptable; for a project with proprietary repo paths or strict commercial confidentiality, self-hosted Rekor would be required.
- **Verifier policies are richer than "trust this public key."** Matching the OIDC issuer + subject pattern is a small upfront cost in policy authoring — but it's also what makes the binding meaningful.
- **Cosign verification requires connectivity** (to Rekor for inclusion proofs) by default. Offline verification is possible with cached proofs; ADR-004's verifier configuration must address this.

**Follow-up work this commits us to.**

- Phase 3 ships: GHA workflow step that runs `cosign sign` with `id-token: write` permission for OIDC, after image push, before any deployment manifest update.
- Phase 6 ships [ADR-004 Kyverno verifyImages](004-admission-policy-kyverno.md) — the verification half. These two must land together or the platform has a signing step with no enforcement.
- Image references in manifests are by **digest** (`@sha256:...`), never by tag. Without digest pinning, signature verification is meaningless because the tag can move to point at an unsigned image.
- Break-glass procedure: documented runbook for "sigstore is down, we need to deploy a fix" — sign-with-ephemeral-key fallback or manual unsigned-deploy approval, with the latter producing a named human accepting non-repudiation.
- A scheduled job verifies that every running pod's image has a valid Cosign signature and is recorded in Rekor — catches images that bypassed admission via some edge path.
