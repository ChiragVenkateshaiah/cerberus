# ADR 016 — Policy-as-Code and IaC Testing: Guardrails in the Pipeline

**Status:** Accepted — extends ADR 015
**Date:** 2026-06-29

---

## Context

ADR 015 gave the platform a pipeline that reviews and applies changes. A pipeline that only
runs `tofu plan` checks that the code is syntactically valid and shows what will change — it
does not check that the change is *allowed*. Nothing stops a PR that makes a bucket public,
omits the mandatory tags, or grants a wildcard IAM action, as long as the HCL parses.

For a platform whose entire premise is disciplined, reviewable infrastructure, the rules that
encode that discipline should themselves be code — enforced automatically, not remembered by
a reviewer. The question is how much gating is proportionate for a solo project before it
becomes ceremony that slows work without adding signal.

## Decision

Add a layered set of **policy-as-code and testing gates** to the `plan.yml` workflow, with a
fast subset mirrored locally via pre-commit.

### The gates, fastest and cheapest first

1. **`tofu fmt -check` and `tofu validate`** — formatting and internal consistency. Free,
   instant.
2. **tflint** — provider-aware linting: deprecated arguments, invalid instance/resource
   attributes, naming conventions.
3. **trivy** (config scan) — security misconfiguration: public exposure, missing encryption,
   over-broad security groups.
4. **conftest / OPA** — the project's own policy, written as Rego in `policy/`:
   - no S3 bucket may be public;
   - every resource must carry `Project`, `Environment`, and `ManagedBy` tags;
   - no IAM policy may use `"Action": "*"` or `"Resource": "*"` outside an explicit allowlist;
   - the Athena workgroup must enforce its configuration.
5. **terraform-docs** — module documentation is generated and checked in; a docs-drift check
   fails the PR if a module's inputs/outputs changed without its docs.
6. **`tofu test`** — native contract tests on the `s3-bucket` and `iam-role` modules (e.g.
   asserting public-access-block is always set, asserting an IAM policy has no wildcard
   resource).

### Why OPA/conftest, not Sentinel

Sentinel is HashiCorp's policy framework and is well integrated with Terraform Cloud, but it
is a paid, closed feature tied to HashiCorp's commercial platform. ADR 011 chose OpenTofu
specifically for license freedom. OPA/conftest is open-source, runs anywhere, and is the
broader industry standard for policy-as-code beyond just Terraform. It is the consistent
choice.

### Why `tofu test`, not Terratest

Terratest is powerful and provisions real resources to assert on them, but it requires a Go
toolchain and real AWS round-trips, which is heavy for module contract tests. Native
`tofu test` runs in HCL with plan-level assertions, no extra language, and is fast enough to
gate every PR. Terratest is reserved for if true end-to-end integration testing against live
AWS is later warranted.

### Proportionality

The OPA policy set is deliberately small and high-signal — a handful of rules that encode the
ADRs' actual invariants, not a generic catalogue. Policy is code: it is reviewed, versioned,
and itself tested for false positives. Tool versions are pinned so a gate does not change
behaviour under the project's feet.

## Consequences

- **Positive:** The platform's security and tagging invariants are enforced mechanically.
  "No public buckets" and "everything tagged" stop being review etiquette and become a gate.
- **Positive:** Policy-as-code with OPA and module testing with `tofu test` are strong,
  current platform-engineering signals.
- **Negative:** Each gate adds CI time and a maintenance surface; policies and pinned tool
  versions need occasional upkeep, and false positives must be tuned out. Kept manageable by
  a small rule set.
- **Dependency:** The gates run inside the ADR 015 pipeline and block merge on failure.

**Extends:** ADR 015
