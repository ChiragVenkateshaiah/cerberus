# ADR 008 — IAM for Serverless: Execution Roles Required, cerberus-cli Cannot Create Them

**Status:** Accepted — amends ADR 003
**Date:** 2026-06-20

---

## Context

ADR 003 established that `cerberus-cli` has no IAM management permissions. The original
compute design in ADR 001 needed one IAM role: an EC2 instance role granting the instance
access to S3 and Glue. ADR 003 documented a known tension — `cerberus-cli` can't create
that role — but deferred resolution to when `infra/*.tf` was actually written.

The pivot in ADR 005 changed the shape of the IAM problem. EMR Serverless and Athena each
require their own service-level execution roles. These are different in nature from an EC2
instance role:

- An **EC2 instance role** is attached to a running instance and grants it AWS access.
  It can be created once and referenced by Tofu's `data` source.
- An **EMR Serverless execution role** is passed to each individual job run at submission
  time (`--execution-role-arn`). It defines what the Spark job itself is allowed to do:
  read from S3, write to S3, interact with Glue. EMR Serverless's service principal
  must be able to assume it.
- An **Athena service role / workgroup configuration** controls where Athena writes query
  results and whether it can access encrypted buckets.

None of these can be created by `cerberus-cli`, because `cerberus-cli` has no `iam:*`
permissions.

## Decision

### What this ADR decides

This ADR names the dependency precisely and records that it is **unresolved** — not that
it is resolved. Resolution happens in `infra/` when OpenTofu files are written, at which
point the implementer must choose one of the paths below.

### Known resolution paths (choose when writing infra/)

**Path A — Manual console creation, Tofu references via data source (recommended)**

Create the following roles manually in the IAM console using the root account or an
admin user:
1. `cerberus-emr-execution-role` — trusted by `emr-serverless.amazonaws.com`; inline
   policy granting `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject` on the four Cerberus
   S3 buckets; `glue:*` on Cerberus tables and databases.
2. `cerberus-athena-role` (if needed) — trusted by the Athena service principal or used
   as the Athena workgroup output bucket identity; grants S3 read on bronze/silver/gold and
   S3 write on the results bucket; Glue read.

Tofu uses `data "aws_iam_role"` to look these up by name and passes their ARNs to the
EMR Serverless Application and Athena Workgroup resources. Tofu never creates or modifies
the roles.

Add a scoped `iam:PassRole` statement to `cerberus-cli` covering only these two role ARNs.
This is the minimal crack in the no-IAM-management wall required to let `cerberus-cli`
submit jobs that reference the roles.

**Path B — Bootstrap via separate admin Tofu run**

A separate `infra/bootstrap/` Tofu config (run once with root/admin credentials, not
`cerberus-cli`) creates the IAM roles, then the main Tofu config references them via
data source. Keeps all role definitions in code without granting `cerberus-cli` IAM
management permissions.

**Path C — Expand cerberus-cli scope (not recommended)**

Grant `cerberus-cli` scoped IAM permissions (`iam:CreateRole`, `iam:PutRolePolicy`,
`iam:AttachRolePolicy`, `iam:PassRole`) restricted by a `ConditionStringEquals` on
resource names prefixed with `cerberus-`. This is auditable but widens the blast radius
of a key leak. Only consider if the manual/bootstrap paths are operationally painful.

### What this ADR does NOT decide

- Which path is taken — that is decided when `infra/` is written.
- Whether additional Athena or Glue permissions are needed — that depends on the actual
  pipeline and query patterns, which are the separate learning track out of scope here.
- Whether the EMR Serverless execution role needs VPC configuration — deferred until
  the first job submission test.

## Consequences

- **The open dependency is tracked** in `infra/PLANNING_CHECKPOINT.yml` as
  `iam-roles-for-serverless` with status `unresolved`. Any future planning session must
  check that file before writing `infra/*.tf` for EMR Serverless or Athena.
- **ADR 003 is amended** in that the "no IAM permissions" model holds for role creation
  but `iam:PassRole` (scoped to named Cerberus roles) is a necessary addition at job
  submission time. ADR 003 is not rewritten — this ADR is its amendment record.
- **`cerberus-cli`'s managed policy list** will need at minimum a new inline statement
  for `iam:PassRole` on specific role ARNs once Path A or B is implemented.
