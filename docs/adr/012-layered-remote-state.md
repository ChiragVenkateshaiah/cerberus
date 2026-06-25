# ADR 012 — Layered Remote State: Stacks Split by Change Frequency

**Status:** Accepted
**Date:** 2026-06-25

---

## Context

A single OpenTofu configuration spanning all Cerberus infrastructure — S3 buckets, Glue
catalog, IAM roles, EMR Serverless application, Athena workgroup — is the simplest
starting layout. It is also the riskiest for routine work. Every `tofu plan` re-evaluates
every resource, so a change to the EMR Serverless application version produces a plan diff
that includes foundational S3 buckets, Glue databases, and IAM roles that did not change.
A reviewer must mentally filter the noise. More critically, a `tofu apply` error in the
compute layer can interrupt a plan that is touching foundational resources unnecessarily.

ADR 008 documented the intra-graph dependency problem: IAM role ARNs do not exist until
after the roles are created, creating ordering constraints that a single-stack graph must
resolve internally via `depends_on` chains. That problem is more naturally solved by
separating stacks along lifecycle boundaries.

## Decision

### Three stacks, split by change frequency and blast radius

```
infra/00-foundation/   S3 buckets, Glue data catalog, CloudWatch log groups
infra/10-iam/          IAM roles and policies (reads foundation outputs)
infra/20-compute/      EMR Serverless application, Athena workgroup (reads above)
```

The organizing principle is **change frequency and blast radius**. Foundation resources
change almost never — S3 buckets and the Glue catalog are created once and persist for
the lifetime of the project. IAM resources change moderately, when platform permissions
evolve. Compute resources change constantly: the EMR Serverless application version,
Athena workgroup configuration, and query result settings are tuned frequently during
active development.

Bundling all three means a routine compute tweak re-plans foundational resources on every
iteration — unnecessary risk with no benefit. Separating them means a compute plan diff
is 10–15 lines touching EMR and Athena, not 200 lines touching everything. A reviewer
can read it confidently in seconds and approve it. This advantage compounds once a GitOps
review flow exists: smaller, scoped diffs are faster to review and carry less risk of an
accidental approval of an unintended foundational change.

### This mirrors real AWS organizational structure

The three-stack split is not arbitrary HCL organization. It mirrors the boundary between
teams in a mature AWS environment: a platform team owns foundation and IAM (shared, stable,
security-sensitive); product teams own compute workloads (frequently changing, owned by
feature work). Building that same structure here — even for a solo project — makes the
architecture legible to any engineer who has worked in a multi-team AWS organization. The
directory layout communicates intent before a single resource block is read.

### Cross-stack data passing via SSM Parameter Store

Stacks are decoupled at the state level: `infra/10-iam/` needs the S3 bucket ARNs that
`infra/00-foundation/` creates; `infra/20-compute/` needs the IAM role ARNs that
`infra/10-iam/` creates. These values must be passed across stack boundaries without
coupling the stacks at the state-file layer.

The chosen mechanism is **AWS SSM Parameter Store**. The producing stack publishes ARNs
as SSM parameters at apply time; the consuming stack reads them via
`data.aws_ssm_parameter` at plan time.

This is chosen over `terraform_remote_state` (reading another stack's `.tfstate` directly)
for a deliberate reason: `terraform_remote_state` requires the consumer to have read access
to the producer's state bucket and state file path, coupling the stacks at the storage
layer and requiring coordinated IAM grants across stacks. SSM parameters decouple them:
the consumer needs only `ssm:GetParameter` on named paths, not access to another stack's
state file. The contract between stacks is the parameter name, not the state layout.

**The tradeoff:** SSM introduces a dependency on parameters existing at known names. A
typo in a parameter name, or a removed export that a consumer still reads, fails at plan
time with a parameter-not-found error. This is the preferred failure mode — failing at
plan time is safer than failing mid-apply — but it means the naming convention is
load-bearing. A parameter name is not just a label; it is the inter-stack API. It must
be consistent, documented, and treated as a breaking change when modified.

**SSM parameter naming convention — the cross-stack contract:**

Foundation stack publishes under:
```
/cerberus/foundation/<resource>-arn
```

Examples:
- `/cerberus/foundation/bronze-bucket-arn`
- `/cerberus/foundation/glue-catalog-database-arn`
- `/cerberus/foundation/emr-logs-bucket-arn`

IAM stack publishes under:
```
/cerberus/iam/<role-name>-arn
```

Example: `/cerberus/iam/emr-serverless-execution-role-arn`

Any change to a parameter name is a breaking change that must update both the producing
`aws_ssm_parameter` resource and every consuming `data.aws_ssm_parameter` block in the
same commit. Standard-tier SSM parameters are free at this scale.

### Stack apply order enforces dependency ordering

With layers, the dependency ordering problem that ADR 008 documented — IAM role ARNs
cannot be referenced before the roles exist — is solved by explicit stack apply order
rather than intra-graph `depends_on` chains:

```
foundation → iam → compute
```

Each stack is applied after its producer stack completes. This sequencing is visible in
the `infra/` directory structure itself; a reader can see the dependency chain from the
directory names without tracing resource references through HCL. A `Makefile` or runbook
sequences the three applies to make this explicit for operators.

## Consequences

- **Apply order is the operator's responsibility.** Running `tofu apply` in
  `infra/20-compute/` before `infra/00-foundation/` has been applied fails at plan time
  when SSM parameters do not exist. This is correct and informative behavior.
- **IAM policy definitions for `cerberus-admin` and `cerberus-cli`** live in
  `infra/10-iam/`, consistent with ADR 010. Reviewers auditing identity permissions look
  in one place, not scattered across multiple configurations.
- **The SSM parameter naming convention is the cross-stack contract.** It is enforced by
  convention and code review, not by any tool. Renaming a parameter without updating all
  consumers is a breaking change that fails at plan time; treat it accordingly.
- **Foundation and IAM changes remain rare.** The day-to-day development loop is almost
  entirely in `infra/20-compute/`. Accidental modifications to foundation or IAM during
  a compute change are caught immediately by the scoped plan diff, before apply.
- **This layering resolves the ADR 008 ordering problem more explicitly** than the
  single-stack approach ADR 008 anticipated. The intra-graph dependency chains are
  replaced by inter-stack apply ordering, which is visible at the directory level and
  requires no hidden `depends_on` attributes inside resource blocks.

**Extends:** ADR 008, ADR 010
