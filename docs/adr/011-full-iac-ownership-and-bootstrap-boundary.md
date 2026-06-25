# ADR 011 — Full IaC Ownership and Bootstrap Boundary

**Status:** Accepted
**Date:** 2026-06-25

---

## Context

A recurring question in any infrastructure-as-code project is which resources are managed
by the IaC tool and which are created manually. Without an explicit answer at the project
level, the boundary erodes: one manual step becomes two, console-created resources
accumulate, and the repo no longer represents what actually exists in the account. This
ADR answers the question as a project-wide principle and documents the one genuine
exception — not to excuse it, but to name it precisely.

## Decision

### All infrastructure is OpenTofu-owned

Every AWS resource in the Cerberus platform is defined in version-controlled OpenTofu
(Terraform-compatible HCL) under `infra/` and applied via `tofu apply`. No resource is
created by clicking through the AWS console (ClickOps).

This is not a rule for its own sake. Version-controlled, diffable, ADR-reasoned
infrastructure is the engineering source of truth for this project. A `tofu plan` diff
is reviewable; it names what will change, in what direction, and against what prior state.
A console click leaves no history, no reasoning, and no record. When a future reader — or
a future self — looks at this repo, they should be able to reconstruct the full AWS
environment from `infra/` alone, with the ADRs explaining why each piece exists and why
it is shaped the way it is. ClickOps produces a repo that describes some of the
infrastructure but cannot reproduce it.

This philosophy is the foundation every other infrastructure decision in this project is
built on.

### The bootstrap paradox — the one deliberate exception

Zero ClickOps is the goal. It is not technically achievable in the absolute sense.

OpenTofu needs a pre-existing AWS credential to make its very first API call. That
credential — the provisioning identity `cerberus-admin` — cannot itself be created by
OpenTofu, because there is no credential yet to run the creation. This is the bootstrap
paradox: the tool that creates identities cannot create the identity that runs the tool.

Therefore, the honest and precise claim is:

> **Exactly one minimal manual bootstrap step:** a human uses pre-existing root or admin
> credentials to create `cerberus-admin` and attach a minimal policy sufficient to run
> `tofu apply`. Everything beyond that single step is OpenTofu-owned.

Claiming "zero manual steps" would be false. Claiming "exactly one minimal manual
bootstrap step, everything else IaC-owned" is both true and precise. The precise claim
is the stronger engineering position — it names the boundary clearly rather than
pretending the boundary does not exist. An engineer who later audits this project can
account for every resource: either it is in `infra/`, or it is `cerberus-admin` and
is documented here.

The provisioning identity itself, and the full separation between provisioning and runtime
identities, are documented in ADR 010.

### Tool: OpenTofu, not Terraform

The infrastructure tool is OpenTofu (`tofu`), the MPL-licensed open-source fork of
Terraform. HashiCorp changed Terraform's license from MPL to BUSL (Business Source
License) in August 2023, which restricts use in competing products and commercial
derivatives. OpenTofu was created by the open-source community in response and remains
under MPL.

OpenTofu is chosen here for license freedom and version-controlled reproducibility. The
HCL syntax and AWS provider are fully compatible with Terraform; no AWS cost dimension
distinguishes the two — both are local CLIs that call AWS APIs, and cost is only the AWS
resources they create, not the tool itself.

Throughout this repo, the tool is always named "OpenTofu" or "`tofu`" in commands. When
referring to the HCL language or provider ecosystem generically, the phrase is "OpenTofu
(Terraform-compatible HCL)".

### Planned future evolution — OIDC federation

The current bootstrap mechanism — `cerberus-admin` as a long-lived IAM user with static
access keys, applied locally — is the practical starting point, not the end state.

The mature end-state replaces long-lived bootstrap keys with OIDC federation: a CI/CD
pipeline assumes a short-lived IAM role via OIDC tokens issued by the CI provider (e.g.,
GitHub Actions). In that model, only the OIDC identity provider and one assumable role
require a one-time manual setup, and no long-lived credentials exist on any machine.

OIDC federation is **not** being built now. The reason is straightforward: `tofu apply`
currently runs locally via the AWS CLI, driven by Claude Code on a personal WSL2 machine.
There is no CI pipeline running applies. Building OIDC federation now would be
authenticating a pipeline that does not exist — a premature abstraction that adds
operational complexity without solving any real problem.

The trigger for introducing OIDC is the introduction of a GitOps or CI flow that runs
`tofu apply` automatically. When that trigger fires:

1. A new ADR is written that supersedes the authentication portion of ADR 010.
2. The new ADR documents the OIDC provider, the assumable role, and any residual one-time
   manual steps (which remain the bootstrap exception documented here).
3. This ADR's bootstrap-boundary principle carries forward unchanged: there will still be
   exactly one manual step, and everything else remains IaC-owned.

Knowing the end state without prematurely building it is the point. The current model is
adequate for the current workflow; the evolution path is documented so the transition is
deliberate rather than discovered.

## Consequences

- **Every new AWS resource must be defined in `infra/` before it is created.** No
  exceptions beyond the documented bootstrap identity. A console-created resource that
  does not appear in `infra/` is a gap in the IaC contract, not an acceptable shortcut.
- **The bootstrap step must be documented.** When `cerberus-admin` is created, the exact
  minimal policy attached and the steps taken are recorded — either here or in a
  `docs/runbooks/` file — so the environment can be reproduced from scratch by someone
  with only the repo and root credentials.
- **OIDC is the documented successor.** Any future planning session that touches CI/CD
  or pipeline automation must consult ADR 010 and this ADR before designing an
  authentication model. Introducing additional long-lived keys for a new provisioning
  purpose after CI exists would contradict the documented evolution path and requires
  explicit justification in a new ADR.

**Extends:** ADR 010
