# ADR 003 — IAM Boundary: Dedicated cerberus-cli User, Scoped Permissions, Hard NovaPay Separation

**Status:** Accepted — amended by ADR 008 (serverless execution roles)
**Date:** 2026-06-20

---

## Context

This AWS account hosts two unrelated projects: Cerberus (this project) and NovaPay (a
separate project under a different IAM user). Conflating their permissions or resources
would create a messy blast radius if credentials are leaked, make cost attribution
ambiguous, and produce a portfolio repo that looks unprofessional to a security-conscious
reviewer.

Even for solo learner work, IAM hygiene matters as a demonstrable practice. The goal is to
show that knowing how to scope permissions is a habit, not an afterthought.

## Decision

### IAM user

A dedicated IAM user `cerberus-cli` is created and used exclusively for all Cerberus CLI
and Terraform/OpenTofu operations. This user's access keys are configured as an AWS CLI
named profile `cerberus` (never the `default` profile, which could bleed into unrelated
commands).

The root account is never used for day-to-day operations.

### Permission scope

`cerberus-cli` is granted the following AWS managed policies:
- `AmazonEC2FullAccess`
- `AmazonS3FullAccess`
- `AmazonVPCFullAccess`

Explicitly **not granted:**
- Any IAM management permissions (`iam:CreateRole`, `iam:PutRolePolicy`, etc.)
- Any permissions for services not in the current architecture (RDS, Lambda, etc.)
- Any permissions touching NovaPay resources

### Why no IAM permissions

Granting `cerberus-cli` the ability to manage IAM roles would mean that a leaked access
key could create arbitrarily-privileged roles — a privilege escalation path. The intended
model is that IAM roles needed by AWS services (EC2 instance roles, later EMR/Athena
execution roles per ADR 008) are created manually in the console by the root account or
an admin user, and `cerberus-cli` only references them by ARN.

The one known friction point: launching an EC2 instance with an attached instance profile
requires `iam:PassRole`. This is documented as an open dependency in ADR 008 and in
`infra/PLANNING_CHECKPOINT.yml`. It will be resolved when `infra/*.tf` is written — not
here.

### Resource tagging

Every AWS resource created for Cerberus carries the tag `Project=cerberus`. This enables:
- Cost filtering in the billing console
- Tag-based IAM conditions as an optional future tightening
- The `vm-cerberus-start`, `vm-cerberus-stop`, and `vm-cerberus` shell aliases in
  `scripts/install-vm-aliases.sh` to resolve the EC2 instance by tag rather than a
  hardcoded instance ID that changes after `terraform destroy`

### NovaPay boundary

NovaPay is a separate project in the same account under a different IAM user. The
separation is enforced by:
1. `cerberus-cli` has no cross-account or assume-role permissions.
2. All Cerberus resources carry `Project=cerberus`; NovaPay resources are never tagged this way.
3. OpenTofu state for Cerberus uses its own dedicated S3 bucket (see ADR 006), separate
   from any NovaPay state.

No Cerberus code or script ever references NovaPay resource names, ARNs, or account IDs.
This is a standing convention, not a technical enforcement.

## Consequences

- Any new AWS service introduced to the Cerberus architecture needs an explicit permission
  review: does `cerberus-cli` need new managed policies, and does the service need its own
  execution role that `cerberus-cli` cannot create?
- The `Project=cerberus` tag is non-negotiable on every resource — scripts, OpenTofu, and
  manual console steps alike. Missing it breaks cost attribution and alias resolution.
- **The no-IAM rule creates an explicit dependency:** service execution roles (EC2 instance
  role, EMR Serverless execution role, Athena service role) must be created out-of-band.
  See ADR 008 for the full accounting of how the serverless redesign changed the shape of
  this dependency.

**Amended by:** ADR 008
