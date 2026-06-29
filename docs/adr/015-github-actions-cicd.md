# ADR 015 — CI/CD with GitHub Actions: Plan on PR, Apply on Merge

**Status:** Accepted — extends ADR 013 and ADR 014
**Date:** 2026-06-29

---

## Context

ADR 011 committed to all infrastructure being OpenTofu-owned and applied deliberately.
Until now, "applied" meant a human running `tofu apply` locally. That is the weakest link
in the chain: it is unreviewed, unaudited, runs from a laptop with whatever credentials are
configured, and leaves no record beyond local shell history. ADR 014 introduced OIDC roles
specifically so a pipeline could provision without static keys. This ADR defines that
pipeline.

The platform now needs a delivery mechanism that reviews changes before they reach AWS,
applies them through an auditable, least-privilege path, and enforces the stack ordering
ADR 012 established.

## Decision

Use **GitHub Actions** as the CI/CD system, with a strict plan/apply split.

### Two workflows

- **`plan.yml` — on pull request.** Assumes `cerberus-ci-plan` via OIDC, runs the policy and
  test gates (ADR 016), runs `tofu plan` for each affected stack, and posts the plan as a PR
  comment. It never applies and never holds write credentials.
- **`apply.yml` — on push to `main`.** Assumes `cerberus-ci-apply` via OIDC and runs
  `tofu apply` for the stacks in dependency order:
  `00-foundation → 10-iam → 20-compute → 30-orchestration`. The ordering from ADR 012/013
  is enforced by sequential job dependencies, not left to chance.

### The fork-credentials boundary

Plan runs use the `pull_request` event, never `pull_request_target`. This is deliberate:
`pull_request_target` runs the workflow with the base repository's secrets and token in the
context of potentially untrusted PR code, a well-known credential-exfiltration vector. With
`pull_request` and OIDC, a pull request from a fork cannot obtain the plan role's credentials.
For a solo private repository the immediate risk is low, but the pattern is the correct one
and is documented as non-negotiable should the repository ever go public or take contributors.

### Environment gating

`dev` applies automatically on merge to `main`. When `prod` is added (ADR 013), it is placed
behind a GitHub Environment with a required reviewer, so a prod apply requires an explicit
human approval after the plan is reviewed. The apply role's trust policy is correspondingly
narrowed (ADR 014).

### Drift detection

Because CI applies infrastructure but the AWS console still exists, out-of-band changes can
cause drift between code and reality. A scheduled workflow runs `tofu plan` on a cadence
(e.g. nightly) against each environment and reports a non-empty plan as drift. This is
detective, not corrective — it surfaces drift for a human to reconcile, it does not
auto-apply.

### What this does NOT decide

- The specific runner (GitHub-hosted is assumed; self-hosted is unnecessary at this scale).
- Automated `tofu apply` rollback. OpenTofu has no transactional rollback; a bad apply is
  fixed by a forward-fixing commit. This is stated so no one expects rollback that does not
  exist.

## Consequences

- **Positive:** Every infrastructure change is reviewed as a PR with a visible plan, applied
  through an auditable, least-privilege, keyless path. This is the "this is real" signal that
  distinguishes a platform from a pile of `.tf` files.
- **Positive:** Stack ordering is encoded in the pipeline, removing a class of manual-ordering
  errors ADR 012 otherwise left to the operator.
- **Negative:** CI now sits on the critical path for all infrastructure changes. A broken
  pipeline blocks applies — mitigated by the ADR 014 break-glass path.
- **Negative:** Plan output posted to PRs can reveal resource names and ARNs. Acceptable in a
  private repo; flagged as a consideration before any public exposure.
- **Dependency:** Relies entirely on the OIDC roles of ADR 014 and the gates of ADR 016.

**Extends:** ADR 013, ADR 014
