# ADR 014 — OIDC Federation for CI Provisioning: Retiring Long-Lived Admin Keys

**Status:** Accepted — supersedes the authentication portion of ADR 010; updates ADR 011
**Date:** 2026-06-29

---

## Context

ADR 010 separated the provisioning identity (`cerberus-admin`) from the runtime identity
(`cerberus-cli`) and chose, for the provisioning identity, a plain IAM user with long-lived
access keys on disk. That ADR named this as the current state, not the end state, and
recorded the trigger for change: "When a GitOps or CI pipeline is introduced to run
`tofu apply`, the authentication mechanism for `cerberus-admin` will be replaced with OIDC
federation." ADR 011 reinforced this and documented the bootstrap paradox — OpenTofu cannot
create the credential that runs OpenTofu.

ADR 015 introduces exactly that CI pipeline. The trigger has fired. This ADR replaces how
the provisioning identity authenticates. It does not touch the separation principle itself —
that remains, and is strengthened.

## Decision

Provisioning authenticates through **GitHub Actions OIDC federation**. There are no
long-lived provisioning access keys anywhere.

### The mechanism

1. An IAM OIDC identity provider for `token.actions.githubusercontent.com` is created once
   in the AWS account.
2. Two IAM roles are created, both trusting that provider:
   - `cerberus-ci-plan` — assumed by pull-request workflow runs; read-mostly permissions
     sufficient for `tofu plan` (read state, describe resources, read SSM).
   - `cerberus-ci-apply` — assumed only by workflow runs on the `main` branch; the broad
     provisioning permissions plus the scoped IAM-write from ADR 010 Sub-decision B.
3. At workflow runtime, GitHub issues a short-lived OIDC token; AWS STS exchanges it for
   temporary credentials. They expire when the job ends.

### The trust policy is the security boundary

The role trust policies condition on the GitHub OIDC `sub` claim, and that condition is
load-bearing. A `sub` scoped too broadly (e.g. `repo:ChiragVenkateshaiah/cerberus-platform:*`) would let
any branch or any pull request assume the role. The conditions are tight:

- `cerberus-ci-apply` trusts only `repo:ChiragVenkateshaiah/cerberus-platform:ref:refs/heads/main`
  (and, once `prod` exists, a GitHub Environment condition gating it further).
- `cerberus-ci-plan` trusts `repo:ChiragVenkateshaiah/cerberus-platform:pull_request`.
- The `aud` claim is pinned to `sts.amazonaws.com`.

A widened `sub` condition is treated as a security change requiring explicit review, exactly
as ADR 012 treats an SSM parameter rename.

### The bootstrap paradox shifts but does not vanish

ADR 011's "exactly one minimal manual step" still holds — its content changes. The one
manual, non-IaC step is now: a human, using root or an IAM Identity Center admin, creates
the OIDC identity provider and the two CI roles (or applies a minimal `infra/bootstrap/`
config with those credentials once). Everything else, including all subsequent changes to
these roles, is OpenTofu-owned. The improvement over ADR 010 is that the bootstrap no longer
leaves any long-lived key on disk — only a trust relationship.

### Break-glass access is retained deliberately

If CI is unavailable and infrastructure must change, there must be a path that does not
depend on the pipeline. That path is a human admin (root, used never for routine work, or an
IAM Identity Center administrative permission set) assuming sufficient privilege
interactively. This is documented in the bootstrap runbook as break-glass, used only when
the pipeline cannot run, and any such use is recorded. A provisioning model with no human
fallback is fragile, not secure.

### What this does NOT decide

- IAM Identity Center (SSO) for routine human access is not adopted now; the only human
  access is break-glass. Full SSO is a larger decision for a multi-account future.
- The `cerberus-cli` runtime identity is unchanged — it remains a profile-based identity for
  local data-plane work (ADR 010). This ADR is about provisioning authentication only.

## Consequences

- **Positive:** No long-lived provisioning credentials exist. A leaked laptop no longer
  carries the keys to rewrite infrastructure. This is the current AWS best practice and the
  single strongest security signal in the platform.
- **Positive:** Plan and apply privileges are split across two roles — least privilege at
  the pipeline-stage level, not just the identity level.
- **Negative:** OIDC setup is more moving parts than pasting an access key — an identity
  provider, two trust policies, and a correct `sub` condition. A misconfigured trust policy
  fails closed (the assume-role is denied), which is the safe failure mode.
- **Dependency:** This ADR presumes the CI pipeline of ADR 015. The two are adopted together.
- **Supersession:** ADR 010's Sub-decision A (plain IAM user with static keys) is superseded.
  ADR 010's identity-separation principle and Sub-decision B (scoped IAM-write on two named
  ARNs) are retained; the scoped IAM-write now attaches to `cerberus-ci-apply`.

**Supersedes:** ADR 010 (authentication portion) · **Updates:** ADR 011
