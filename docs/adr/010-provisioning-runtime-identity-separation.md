# ADR 010 — Separation of Provisioning Identity from Runtime Identity

**Status:** Accepted — extends ADR 003; see ADR 011 (IaC ownership and bootstrap boundary), ADR 012 (stack layering)
**Date:** 2026-06-25

---

## Context

The serverless architecture (EMR Serverless, Athena, Glue, CloudWatch, S3) requires two
distinct categories of AWS operation: provisioning (creating and modifying infrastructure
via OpenTofu) and runtime (day-to-day data engineering work — submitting jobs, running
queries, reading results, checking logs).

ADR 003 established `cerberus-cli` as the project's sole IAM user, deliberately scoped to
EC2/S3/VPC FullAccess only, with no IAM, Glue, EMR, or Athena permissions. That scoping
was the right call for the original EC2 architecture but it means `cerberus-cli` cannot
provision the serverless stack. Two paths existed: widen `cerberus-cli` to also cover
provisioning, or create a separate dedicated provisioning identity.

Widening `cerberus-cli` would collapse the distinction between "operator of infrastructure"
and "creator of infrastructure" into a single credential — the pattern ADR 003 was designed
to avoid. Creating a second identity keeps the separation clean and matches the way
professional environments handle CI/deploy identities versus application runtime identities.

## Decision

Two distinct IAM identities with non-overlapping responsibilities:

### cerberus-admin — the provisioning identity

A new IAM user `cerberus-admin`, configured as an AWS CLI named profile `cerberus-admin`.
Used **only** for `tofu apply` and `tofu destroy` runs. Holds broad provisioning permissions
across S3, Glue, EMR Serverless, Athena, and CloudWatch, plus scoped IAM-write permissions
covering exactly two named ARNs (see Sub-decision B below).

**Bootstrap note:** `cerberus-admin` itself cannot be created by OpenTofu, because OpenTofu
needs a pre-existing credential to make its very first API call — that credential cannot
itself be OpenTofu-created. Creating `cerberus-admin` is therefore the one irreducible
manual bootstrap step in this project: a human uses pre-existing root or admin credentials
to create the minimal bootstrap identity, and OpenTofu owns everything from that point
forward. ADR 011 documents the full IaC-ownership philosophy, the bootstrap paradox
precisely, and the boundary of this one exception.

Used rarely and deliberately; never for routine data engineering work.

### cerberus-cli — the runtime identity

The existing IAM user from ADR 003, configured as AWS CLI profile `cerberus`. Used
**constantly** for day-to-day platform operation: submitting EMR Serverless job runs,
checking their status, executing Athena queries, reading Glue Catalog metadata, reading
and writing S3 data, and reading CloudWatch logs. Holds only permissions needed to
**operate** the platform, never to change its shape. Cannot create, modify, or destroy
infrastructure.

This mirrors the standard industry pattern of separating a deploy/CI identity (broad,
rarely used, offline most of the time) from an application runtime identity (narrow,
constantly used). A leaked runtime credential cannot rewrite infrastructure; a deploy
credential is not exposed by routine work.

### Sub-decision A — cerberus-admin is a plain IAM user, not an assumable role

`cerberus-admin` is currently a plain IAM user with long-lived access keys stored in the
local AWS CLI credentials file, not an IAM role gated by MFA assumption. This is the
current state, not the end state.

**The tradeoff:** An assumable-role-plus-MFA pattern is more secure — no long-lived admin
keys on disk, a fresh MFA challenge per privileged action, and no static credentials to
rotate. That pattern **is** the right choice for shared accounts, team environments, or
production systems. It was not chosen here because:

- This is a solo operator. The friction of per-deploy MFA role assumption would be absorbed
  by one person, not amortized across a team.
- The workflow is CLI-automation-driven through Claude Code. Interactive MFA prompts fight
  that automation model at every `tofu apply`.
- The blast radius is already contained: the keys live on a single personal WSL2 machine,
  are scoped to one project's resources, and can be instantly revoked from the console.

**If this were a team or production system, this decision would flip to
assumable-role-plus-MFA.**

The mitigation for carrying long-lived admin keys is an AWS Budgets alert. Long-lived
admin credentials without a spend alarm is the actual risk — the keys alone are not the
problem if any unexpected usage shows up immediately in billing. See the Budgets
configuration in `infra/` for the mandatory alert. Without the alert, this sub-decision
is not adequately mitigated.

**Planned future evolution:** The long-lived-keys-on-an-IAM-user model is the current
state, not the end state. When a GitOps or CI pipeline is introduced to run `tofu apply`,
the authentication mechanism for `cerberus-admin` will be replaced with OIDC federation:
the pipeline assumes a role via short-lived tokens, and only the OIDC provider and one
assumable role require a one-time manual setup. At that point, a new ADR supersedes the
authentication portion of this ADR. The identity separation principle established here —
provisioning identity distinct from runtime identity — carries forward unchanged; only
the credential mechanism changes. See ADR 011 for the full OIDC sequencing rationale and
the precise trigger for that transition.

### Sub-decision B — cerberus-admin holds scoped IAM-write permissions

`cerberus-admin`'s IAM-write permissions are not zero. Tofu manages two IAM policy
documents that are part of the project's reproducible infrastructure:

1. The inline permissions policy attached to `cerberus-emr-serverless-execution-role`
   (what the Spark job itself is allowed to do).
2. The runtime grant attached to `cerberus-cli` (what the daily-operator identity is
   allowed to do).

**The tradeoff:** Keeping these two policies as manual root-console steps would give
`cerberus-admin` zero IAM power, minimising its blast radius. The cost is that two
critical policy documents would live nowhere in the repo — undocumented, unauditable,
and inconsistent with the infrastructure-as-code premise that makes this project legible
to a reviewer.

The safety boundary that makes OpenTofu-managed IAM acceptable here: `cerberus-admin`'s
IAM-write permissions are scoped to **exactly two named ARNs** — `cerberus-emr-serverless-
execution-role` and the `cerberus-cli` user — via specific actions such as
`iam:PutRolePolicy` and `iam:PutUserPolicy` on those resources only. This is not
`IAMFullAccess`. It cannot touch any other identity in the account, including the
unrelated `novapay-cli` user.

The actual policy for these scoped IAM-write permissions lives in `infra/10-iam/`; it is
not enumerated here. ADR 012 documents how the two identities' policies are organized
across the project's layered stacks.

## Consequences

- **Two CLI profiles to manage.** `cerberus-admin` for provisioning runs; `cerberus` for
  all daily work. Slightly more setup than a single-profile model; materially clearer
  security posture.
- **cerberus-cli needs a broader runtime grant than ADR 008 sketched.** ADR 008 described
  the runtime requirement narrowly as `iam:PassRole` plus `emr-serverless:StartJobRun`.
  The full "operate the platform" set is larger: EMR Serverless job submission and status
  reads, Athena query execution and results retrieval, Glue Catalog reads, S3 read/write
  on Cerberus buckets, and CloudWatch log reads. The `infra/10-iam/` stack implements
  this grant. The final policy lives in code; this ADR records only the scope constraint:
  **operational access, never infrastructure-changing access.**
- **Long-lived admin keys exist on disk.** Mitigated by project-scoped permissions
  (cannot reach NovaPay or account-level resources), instant revocability from the
  console, single-machine storage on a personal WSL2 host, and a mandatory AWS Budgets
  alert.
- **This ADR extends ADR 003's boundary model.** ADR 003 established one scoped identity.
  This ADR replaces that with two identities split by purpose, while preserving the core
  principle: no single credential can both operate and reshape the infrastructure.
- The `iam-roles-for-serverless` open dependency in `infra/PLANNING_CHECKPOINT.yml`
  (tracked since ADR 008) is resolved by this decision: `cerberus-admin` is the identity
  that creates those roles via Tofu, not `cerberus-cli` and not a manual root-console step.
- **Policy organization** across the two identities follows the layered stack structure
  documented in ADR 012.

**Amends:** ADR 003
