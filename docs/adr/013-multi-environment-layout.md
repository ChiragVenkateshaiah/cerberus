# ADR 013 — Multi-Environment Layout: Directory-per-Environment Over Reusable Modules

**Status:** Accepted — extends ADR 012
**Date:** 2026-06-29

---

## Context

ADR 012 split the infrastructure into three stacks (`00-foundation`, `10-iam`, `20-compute`)
along change-frequency boundaries, with SSM Parameter Store as the cross-stack contract. It
described a single deployment. It did not answer how more than one environment — a `dev` to
iterate in and a `prod` to keep stable — is represented.

A portfolio platform that only ever has one environment is a weaker demonstration than one
that shows a clean dev/prod separation, because environment promotion is where real IaC
discipline is tested: shared logic must not be copy-pasted, and a change to dev must not be
able to silently alter prod. The original repo skeleton also carried `infra/modules/ec2` and
`infra/modules/vpc` stubs from the ADR 001 era; EC2 and the VPC were removed by ADR 005, so
the module layout needs to be restated, not just extended.

Three ways to represent environments in OpenTofu were considered.

## Decision

Use **directory-per-environment with shared modules**: all reusable logic lives in
`infra/modules/`, and each environment is a directory tree of thin stack configurations that
instantiate those modules.

```
infra/
  bootstrap/                 # the one manual step (ADR 011/014)
  modules/                   # all real resource logic, reusable + tofu-tested
    s3-bucket/  glue-database/  iam-role/  emr-serverless-app/
    athena-workgroup/  step-function-pipeline/
  envs/
    dev/   00-foundation/ 10-iam/ 20-compute/ 30-orchestration/
    prod/  (same structure; applied later)
```

The stacks under `envs/<env>/` hold backend configuration, provider config, module calls,
and SSM wiring — not resource bodies. The SSM contract from ADR 012 gains an environment
segment: `/cerberus/<env>/<layer>/<resource>-arn`.

### Why directory-per-environment, not workspaces

OpenTofu workspaces keep one configuration and switch a state namespace per workspace. They
are convenient for ephemeral, identical copies, but they are the wrong tool for durable
dev/prod separation: a single set of provider credentials and one configuration serve every
workspace, so a misapplied `tofu apply` in the wrong workspace can hit the wrong environment,
and the environments cannot diverge in structure without conditionals polluting the
configuration. The separation is a runtime flag, not a directory boundary — exactly the kind
of invisible coupling ADR 012 set out to avoid. Directory-per-environment makes the boundary
visible in the filesystem and gives each environment its own state file and backend.

### Why not Terragrunt

Terragrunt removes the boilerplate that directory-per-environment introduces (repeated backend
and provider blocks) and is a legitimate choice at scale. It was not chosen here because it
adds a third-party wrapper and its own DSL on top of OpenTofu for a project with two
environments and four stacks. ADR 011 committed to OpenTofu for license freedom and
reproducibility; adding a wrapper dependency to save a few dozen lines of boilerplate is not
a trade worth making at this size. If the environment or stack count grows enough that the
boilerplate becomes a real maintenance burden, Terragrunt is the documented next step.

### Modules retired and added

`infra/modules/ec2` and `infra/modules/vpc` are removed — EC2 and the VPC left the
architecture with ADR 005. `infra/modules/s3` is replaced by a fuller `s3-bucket` module
(block-public-access, versioning, SSE, lifecycle — ADR 002). New modules: `glue-database`,
`iam-role`, `emr-serverless-app`, `athena-workgroup`, and `step-function-pipeline` (the
last introduced by ADR 017).

### Stack ordering extended

ADR 012 defined three stacks. This ADR adds a fourth: `30-orchestration/` holding the Step
Functions state machine and EventBridge schedule (ADR 017). The full apply order is:

```
00-foundation → 10-iam → 20-compute → 30-orchestration
```

### What this does NOT decide

- The `prod` environment is structured now but not applied now. When it is, prod-specific
  guardrails (manual-approval gating in CI, tighter cost caps) are decided then, not here.
- Whether dev and prod share an AWS account or use separate accounts. They share one account
  for now (consistent with ADR 003's single-account, tag-separated model); a multi-account
  split is a larger decision left open.

## Consequences

- **Positive:** Environment boundaries are visible in the directory tree; each env has
  isolated state. All real logic lives once, in tested modules.
- **Positive:** Adding `prod` is creating a directory of thin module calls, not rewriting
  resources — the structure proves it scales.
- **Negative:** Backend and provider configuration is repeated per stack per environment.
  This is the accepted cost of explicitness; the modules hold everything that matters.
- **Cleanup:** The `ec2` and `vpc` module stubs are deleted; `observability/grafana/`
  remains vestigial under ADR 007 and is addressed separately.
- **Contract change:** Every SSM parameter name now carries an `<env>` segment. This is a
  breaking change to the ADR 012 naming convention and is adopted before any stack is written.

**Extends:** ADR 012
