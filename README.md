# Cosign-Gatekeeper-SupplyChain-Engine

Ensure only policy-compliant, cryptographically verified container images run on AKS.

## Problem statement

A release request binds an image digest to keyless signing evidence and an admission policy plan that rejects unsigned or unapproved artifacts before AKS execution.

A production implementation can still fail even when every resource deploys successfully. The material risk is apparently successful automation that lacks bounded scope, a reproducible denial path, or evidence operators can use during review and recovery. The design therefore treats AKS, ACR, Gatekeeper, and the surrounding identity and evidence controls as one reviewable system rather than unrelated configuration tasks.

## Example case study

### Situation

A compromised registry credential could let an attacker replace a trusted image tag. Digest signatures and admission enforcement make provenance verifiable at deployment time rather than assuming the registry alone is sufficient.

### Response

A pipeline submits one trusted signed image and one image rebuilt outside the approved workflow. Gatekeeper admits the matching signature and provenance, denies the second before scheduling, and returns an actionable reason to developers.

The team first exercises the repository's synthetic approved and denied fixtures. An approved request must produce the same idempotent plan on replay; a stale, unscoped, public, or unapproved request must fail before an Azure adapter is allowed to run.

### Expected outcome

Stakeholders receive a decision package they can attach to a change record: requested scope, controls evaluated, the reason for approval or denial, and the explicit handoff to live integration. The example supports design review and incident rehearsal without pretending that a local test changed Azure.

## Architecture

GitHub Actions builds and scans images, generates an SBOM and provenance, signs with Cosign using ephemeral identity, and pushes to ACR. Gatekeeper admission policies require trusted issuer, identity, digest, and approved exceptions.

Primary services: `AKS`, `ACR`, `Gatekeeper`, `Cosign`, `GitHub Actions`.

This repository implements the first production-oriented vertical slice: a
fail-closed, adapter-neutral control plane that validates tenant scope,
freshness, approvals, secretless identity, private access, and the exact
project action before producing a deterministic execution plan. Azure adapters
consume that plan; they are deliberately outside the local simulator so local
tests cannot claim a live cloud change occurred.

![Icon-based architecture for Cosign-Gatekeeper-SupplyChain-Engine](docs/architecture.svg)

The upper boundary names the principal services and technologies used by this repository. The lower boundary shows the implemented control flow: desired state is validated, provider action remains an explicit integration gate, and sanitized evidence is retained for review and deterministic replay.

## Best complementary diagram

**Recommended view: Container supply-chain verification pipeline.** A delivery-pipeline view is the strongest complement because it makes artifact progression, security gates, promotion authority, and evidence outputs visible.

![Icon-based container supply-chain verification pipeline for Cosign-Gatekeeper-SupplyChain-Engine](docs/operational-view.svg)

The view follows **Build immutable image → Sign and publish artifact → Verify admission policy → Run approved workload**. Use it during design reviews, operational walkthroughs, and failure-mode discussions; use the logical architecture above when the question is which technologies integrate.

## Quickstart

Requirements: Python 3.11+ and Git. No Azure credentials are required.

```bash
./scripts/validate.sh
python3 src/control_plane.py --request examples/approved-request.json
```

The command emits canonical JSON with a stable idempotency key. The denied
fixture exits with status 2 and explains the failed invariants.

## Security boundaries

- Managed identity or workload identity only; embedded credentials are denied.
- Public network access and stale evidence are denied.
- Production and break-glass targets require explicit approval.
- The IaC entry point is opt-in and defaults to deploying nothing.
- Evidence output contains identifiers and decisions, never credential values.

## Verification and limitations

Local validation covers 13 tests, deterministic replay, JSON parsing, Python
compilation, ignore hygiene, and Bicep compilation when a compiler is present.
It does **not** prove Azure deployment, service licensing, quota, data-plane
permissions, provider/API availability, cloud failover, load, cost, or teardown.
See [`docs/test-matrix.md`](docs/test-matrix.md) and [`docs/runbook.md`](docs/runbook.md) before any integration trial.

## Community

See [`CONTRIBUTING.md`](CONTRIBUTING.md), [`SECURITY.md`](SECURITY.md), [`SUPPORT.md`](SUPPORT.md), and [`LICENSE`](LICENSE). The reference
is intentionally conservative and uses synthetic identifiers only.

## Repository guide

- [Architecture](docs/architecture.md)
- [Threat model](docs/threat-model.md)
- [Operations runbook](docs/runbook.md)
- [Test matrix](docs/test-matrix.md)
- [Cost model](docs/cost-model.md)
- [Security policy](SECURITY.md)
- [Contributing guide](CONTRIBUTING.md)
- [Support policy](SUPPORT.md)
- [Changelog](CHANGELOG.md)
- [License](LICENSE)

## Infrastructure inputs

Resource behavior and deploy-time values are intentionally separated:

- [Bicep template](infra/main.bicep) — Azure resources, modules, and security controls.
- [Bicep parameters](infra/main.bicepparam) — environment-specific names, regions, identities, and feature inputs.

Start with the parameter file's safe values, replace synthetic identifiers, and run an Azure what-if before deployment.

## Attribution

Azure product icons come from [Microsoft's official Azure Architecture Icons](https://learn.microsoft.com/azure/architecture/icons/). Open-source marks are sourced from [Simple Icons](https://simpleicons.org/) when shown; each mark identifies its respective technology.
