# Week 1 — Day-by-Day Plan

**Companion doc:** [`overall.md`](overall.md)

Each day = one focused 1–2 hour evening block. Exit signals are hard stops — do not move
to the next day until the current day's exit signal is met.

---

## Day 1 — Repo scaffolding & local guardrails

**ADRs:** 013, 016
**Goal:** A clean, pre-commit-green skeleton that reflects the module + env layout before
any real `.tf` resource is written.

### Directory structure to create

```
infra/
  bootstrap/              # placeholder + comment: "ADR 011/014 — one manual step"
  modules/
    s3-bucket/
      main.tf             # empty resource stubs
      variables.tf
      outputs.tf
  envs/
    dev/
      00-foundation/
        backend.tf        # S3 backend config (bucket filled in on Day 2)
        versions.tf       # provider pin + default_tags
      10-iam/
        backend.tf
        versions.tf
      20-compute/
        backend.tf
        versions.tf
      30-orchestration/
        backend.tf
        versions.tf
    prod/                 # mirror structure; comment: "structure ready, applied later"
.github/
  workflows/              # empty — populated Day 4/6
policy/                   # empty — populated Day 5
docs/
  runbooks/               # empty — populated Day 2
pipelines/                # empty — Phase 2+
```

### Files to write

**`.pre-commit-config.yaml`**
```yaml
repos:
  - repo: https://github.com/antonbabenko/pre-commit-terraform
    rev: v1.92.0   # pin — check latest
    hooks:
      - id: terraform_fmt
      - id: terraform_validate
      - id: terraform_tflint
      - id: terraform_docs
        args: ["--hook-config=--path-to-file=README.md"]
```

**`.tflint.hcl`**
```hcl
plugin "aws" {
  enabled = true
  version = "0.32.0"   # pin
  source  = "github.com/terraform-linters/tflint-ruleset-aws"
}
```

**`infra/envs/dev/00-foundation/versions.tf`**
```hcl
terraform {
  required_version = "~> 1.7"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.50"
    }
  }
  backend "s3" {}   # filled in backend.tf
}

provider "aws" {
  region = "ap-south-1"
  default_tags {
    tags = {
      Project     = "cerberus"
      Environment = "dev"
      ManagedBy   = "opentofu"
    }
  }
}
```

Repeat the `versions.tf` pattern for the other three dev stacks (change `Environment` tag
per stack where applicable; `dev` is fine for all four this week).

### Exit signal

```bash
pre-commit run --all-files   # green
tofu validate                # passes in each envs/dev/* directory
```

**Time estimate:** 1–1.5 hours

---

## Day 2 — Manual bootstrap: OIDC + state backend

**ADRs:** 011, 014
**Goal:** Cross the bootstrap boundary. This is the one day you use root/admin credentials.
Everything created today enables CI to run `tofu apply` tomorrow onward without static keys.

### Sequence (order matters)

**Step 1 — S3 state bucket**
- Region: `ap-south-1`
- Name: `cerberus-tofu-state-<account-id>` (globally unique; bake in account ID)
- Versioning: enabled
- Encryption: SSE-S3 (or SSE-KMS if you want KMS practice)
- Block public access: all four settings true
- No lifecycle rule yet

**Step 2 — State locking**
- OpenTofu ≥ 1.7: native S3 locking — no DynamoDB needed. Verify: `tofu version`
- If < 1.7: create DynamoDB table `cerberus-tofu-lock` (PAY_PER_REQUEST, partition key `LockID`)

**Step 3 — OIDC identity provider**
- URL: `https://token.actions.githubusercontent.com`
- Audience: `sts.amazonaws.com`
- Thumbprint: let AWS auto-populate, or use the value from GitHub's OIDC docs

**Step 4 — `cerberus-ci-plan` role**

Trust policy:
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<account>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": {
        "token.actions.githubusercontent.com:aud": "sts.amazonaws.com",
        "token.actions.githubusercontent.com:sub": "repo:ChiragVenkateshaiah/cerberus-platform:pull_request"
      }
    }
  }]
}
```

Permissions: `ReadOnlyAccess` managed policy + inline `ssm:GetParameter*` on
`arn:aws:ssm:ap-south-1:<account>:parameter/cerberus/*`

**Step 5 — `cerberus-ci-apply` role**

Same trust policy structure, but `sub`:
```
"repo:ChiragVenkateshaiah/cerberus-platform:ref:refs/heads/main"
```

Permissions: broad provisioning (S3, Glue, EMR Serverless, Athena, States, EventBridge,
CloudWatch, SSM full, Budgets, CE) + scoped IAM-write:
```json
{
  "Effect": "Allow",
  "Action": ["iam:PutRolePolicy", "iam:GetRolePolicy", "iam:DeleteRolePolicy"],
  "Resource": "arn:aws:iam::<account>:role/cerberus-emr-serverless-execution-role"
},
{
  "Effect": "Allow",
  "Action": ["iam:PutUserPolicy", "iam:GetUserPolicy", "iam:DeleteUserPolicy"],
  "Resource": "arn:aws:iam::<account>:user/cerberus-cli"
}
```

**Step 6 — Verify OIDC trust**

Write `.github/workflows/test-oidc.yml` (temporary — delete after):
```yaml
on: push
permissions:
  id-token: write
  contents: read
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account>:role/cerberus-ci-apply
          aws-region: ap-south-1
      - run: aws sts get-caller-identity
```

Push to a branch. Confirm the output shows the `cerberus-ci-apply` role ARN. Delete the
workflow file after confirmation.

**Step 7 — Fill in `backend.tf` for each stack**

```hcl
# infra/envs/dev/00-foundation/backend.tf
terraform {
  backend "s3" {
    bucket  = "cerberus-tofu-state-<account-id>"
    key     = "dev/00-foundation/terraform.tfstate"
    region  = "ap-south-1"
    # use_lockfile = true   # OpenTofu 1.7+ native locking
  }
}
```

**Step 8 — `docs/runbooks/bootstrap.md`**

Document exact steps, ARNs, trust policy JSON, and the test-oidc verification. A cold reader
must be able to recreate the bootstrap from scratch using only this runbook.

### Exit signal

```bash
# The test-oidc workflow output shows:
# "Arn": "arn:aws:sts::<account>:assumed-role/cerberus-ci-apply/..."
```

Runbook committed. No long-lived admin keys anywhere on disk.

**Time estimate:** 1.5–2 hours (OIDC verification is where time goes)

---

## Day 3 — `s3-bucket` module + `00-foundation` stack

**ADRs:** 002, 012, 013
**Goal:** A working, plan-clean foundation stack built from a reusable, tested module.
No apply yet — that happens on Day 6 via CI.

### `infra/modules/s3-bucket/`

**`variables.tf`**
```hcl
variable "bucket_name"              { type = string }
variable "noncurrent_expiry_days"   { type = number; default = 30 }
```

**`main.tf`**
```hcl
resource "aws_s3_bucket" "this" {
  bucket = var.bucket_name
}

resource "aws_s3_bucket_versioning" "this" {
  bucket = aws_s3_bucket.this.id
  versioning_configuration { status = "Enabled" }
}

resource "aws_s3_bucket_server_side_encryption_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    apply_server_side_encryption_by_default { sse_algorithm = "AES256" }
  }
}

resource "aws_s3_bucket_public_access_block" "this" {
  bucket                  = aws_s3_bucket.this.id
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

resource "aws_s3_bucket_lifecycle_configuration" "this" {
  bucket = aws_s3_bucket.this.id
  rule {
    id     = "expire-noncurrent"
    status = "Enabled"
    noncurrent_version_expiration { noncurrent_days = var.noncurrent_expiry_days }
  }
}
```

**`outputs.tf`**
```hcl
output "bucket_id"  { value = aws_s3_bucket.this.id }
output "bucket_arn" { value = aws_s3_bucket.this.arn }
```

### `infra/envs/dev/00-foundation/main.tf`

```hcl
locals {
  env = "dev"
}

# Six S3 buckets via the s3-bucket module
module "bronze"        { source = "../../../modules/s3-bucket"; bucket_name = "cerberus-bronze-<account-id>" }
module "silver"        { source = "../../../modules/s3-bucket"; bucket_name = "cerberus-silver-<account-id>" }
module "gold"          { source = "../../../modules/s3-bucket"; bucket_name = "cerberus-gold-<account-id>" }
module "scripts"       { source = "../../../modules/s3-bucket"; bucket_name = "cerberus-scripts-<account-id>" }
module "athena_results"{ source = "../../../modules/s3-bucket"; bucket_name = "cerberus-athena-results-<account-id>" }
module "emr_logs"      { source = "../../../modules/s3-bucket"; bucket_name = "cerberus-emr-logs-<account-id>" }

# Glue Data Catalog database
resource "aws_glue_catalog_database" "lakehouse" {
  name = "cerberus_lakehouse"
}

# CloudWatch log groups
resource "aws_cloudwatch_log_group" "emr_serverless" {
  name              = "/cerberus/${local.env}/emr-serverless"
  retention_in_days = 30
}

resource "aws_cloudwatch_log_group" "athena" {
  name              = "/cerberus/${local.env}/athena"
  retention_in_days = 30
}
```

### Exit signal

```bash
cd infra/envs/dev/00-foundation
tofu init
tofu validate   # passes
tofu plan       # ~28–32 resources planned, zero errors
```

No apply. The plan output is the deliverable.

**Time estimate:** 1.5–2 hours

---

## Day 4 — SSM exports + `plan.yml` CI workflow

**ADRs:** 012, 015
**Goal:** Open a PR and see a plan comment posted automatically.

### SSM parameter exports in `00-foundation/main.tf`

```hcl
resource "aws_ssm_parameter" "bronze_bucket_arn" {
  name  = "/cerberus/dev/foundation/bronze-bucket-arn"
  type  = "String"
  value = module.bronze.bucket_arn
}
resource "aws_ssm_parameter" "silver_bucket_arn" {
  name  = "/cerberus/dev/foundation/silver-bucket-arn"
  type  = "String"
  value = module.silver.bucket_arn
}
resource "aws_ssm_parameter" "gold_bucket_arn" {
  name  = "/cerberus/dev/foundation/gold-bucket-arn"
  type  = "String"
  value = module.gold.bucket_arn
}
resource "aws_ssm_parameter" "scripts_bucket_arn" {
  name  = "/cerberus/dev/foundation/scripts-bucket-arn"
  type  = "String"
  value = module.scripts.bucket_arn
}
resource "aws_ssm_parameter" "athena_results_bucket_arn" {
  name  = "/cerberus/dev/foundation/athena-results-bucket-arn"
  type  = "String"
  value = module.athena_results.bucket_arn
}
resource "aws_ssm_parameter" "emr_logs_bucket_arn" {
  name  = "/cerberus/dev/foundation/emr-logs-bucket-arn"
  type  = "String"
  value = module.emr_logs.bucket_arn
}
resource "aws_ssm_parameter" "glue_database_arn" {
  name  = "/cerberus/dev/foundation/glue-catalog-database-arn"
  type  = "String"
  value = aws_glue_catalog_database.lakehouse.arn
}
```

### `.github/workflows/plan.yml`

```yaml
name: tofu-plan

on:
  pull_request:
    paths:
      - "infra/**"
      - "policy/**"

permissions:
  id-token: write       # required for OIDC — do not remove
  contents: read
  pull-requests: write  # to post plan comment

jobs:
  plan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account>:role/cerberus-ci-plan
          aws-region: ap-south-1

      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.7.x"

      - name: fmt check
        run: tofu fmt -check -recursive infra/

      - name: validate
        run: tofu validate
        working-directory: infra/envs/dev/00-foundation

      - name: plan
        id: plan
        run: tofu plan -no-color 2>&1 | tee plan.txt
        working-directory: infra/envs/dev/00-foundation

      - name: post plan comment
        uses: peter-evans/create-or-update-comment@v4
        with:
          issue-number: ${{ github.event.pull_request.number }}
          body: |
            ### `tofu plan` — dev/00-foundation
            ```
            ${{ steps.plan.outputs.stdout }}
            ```
```

### Exit signal

Open a PR with today's changes. The `plan.yml` workflow triggers. A comment appears on the
PR with the plan output. Gates may be absent or failing — that is Day 5's problem.

**Time estimate:** 1–1.5 hours

---

## Day 5 — Policy-as-code gates

**ADRs:** 016
**Goal:** Every gate wired into `plan.yml`; the PR is fully green.

### `policy/` OPA rules

**`policy/no_public_bucket.rego`**
```rego
package cerberus

deny[msg] {
  r := input.resource_changes[_]
  r.type == "aws_s3_bucket_public_access_block"
  val := r.change.after
  [val.block_public_acls, val.block_public_policy,
   val.ignore_public_acls, val.restrict_public_buckets][_] == false
  msg := sprintf("S3 bucket public access block not fully enabled: %s", [r.address])
}
```

**`policy/required_tags.rego`**
```rego
package cerberus

required_tags := {"Project", "Environment", "ManagedBy"}

deny[msg] {
  r := input.resource_changes[_]
  r.change.actions[_] != "delete"
  existing := {k | r.change.after.tags[k]}
  missing := required_tags - existing
  count(missing) > 0
  msg := sprintf("Resource %s missing required tags: %v", [r.address, missing])
}
```

**`policy/no_wildcard_iam.rego`**
```rego
package cerberus

deny[msg] {
  r := input.resource_changes[_]
  r.type == "aws_iam_policy_document"
  stmt := r.change.after.statement[_]
  stmt.actions[_] == "*"
  msg := sprintf("IAM wildcard action in %s", [r.address])
}
```

### Add gates to `plan.yml` (insert before the `plan` step)

```yaml
      - name: tflint
        run: |
          curl -s https://raw.githubusercontent.com/terraform-linters/tflint/master/install_linux.sh | bash
          tflint --init
          tflint --chdir=infra/envs/dev/00-foundation

      - name: trivy
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: config
          scan-ref: infra/
          exit-code: "1"

      - name: generate plan JSON for conftest
        run: |
          tofu plan -out=plan.tfplan
          tofu show -json plan.tfplan > plan.json
        working-directory: infra/envs/dev/00-foundation

      - name: conftest
        run: conftest test infra/envs/dev/00-foundation/plan.json --policy policy/

      - name: terraform-docs
        run: |
          terraform-docs markdown infra/modules/s3-bucket/ > /tmp/docs-check.md
          diff /tmp/docs-check.md infra/modules/s3-bucket/README.md || \
            (echo "Module docs are out of date — run terraform-docs locally" && exit 1)
```

Also add `infra/modules/s3-bucket/README.md` (generated by `terraform-docs`) to the
module directory and commit it.

### Exit signal

The PR is fully green — every step passes:
`fmt ✓  validate ✓  tflint ✓  trivy ✓  conftest ✓  terraform-docs ✓  plan ✓`

**Time estimate:** 1.5–2 hours (OPA Rego is the likely time sink)

---

## Day 6 — `apply.yml` + cost guardrails → foundation applied via CI

**ADRs:** 015, 019
**Goal:** Merge the PR. Foundation lands in AWS via CI. Cost controls are live.

### Cost guardrail resources in `00-foundation/main.tf`

```hcl
resource "aws_budgets_budget" "monthly" {
  name         = "cerberus-monthly"
  budget_type  = "COST"
  limit_amount = "15"
  limit_unit   = "USD"
  time_unit    = "MONTHLY"

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 80
    threshold_type             = "PERCENTAGE"
    notification_type          = "ACTUAL"
    subscriber_email_addresses = ["chiragvenkateshaiah95@gmail.com"]
  }

  notification {
    comparison_operator        = "GREATER_THAN"
    threshold                  = 100
    threshold_type             = "PERCENTAGE"
    notification_type          = "FORECASTED"
    subscriber_email_addresses = ["chiragvenkateshaiah95@gmail.com"]
  }
}

resource "aws_ce_anomaly_monitor" "cerberus" {
  name              = "cerberus-anomaly-monitor"
  monitor_type      = "DIMENSIONAL"
  monitor_dimension = "SERVICE"
}

resource "aws_ce_anomaly_subscription" "cerberus" {
  name      = "cerberus-anomaly-subscription"
  frequency = "DAILY"
  monitor_arn_list = [aws_ce_anomaly_monitor.cerberus.arn]
  subscriber {
    type    = "EMAIL"
    address = "chiragvenkateshaiah95@gmail.com"
  }
  threshold_expression {
    dimension {
      key           = "ANOMALY_TOTAL_IMPACT_ABSOLUTE"
      values        = ["5"]
      match_options = ["GREATER_THAN_OR_EQUAL"]
    }
  }
}
```

### `.github/workflows/apply.yml`

```yaml
name: tofu-apply

on:
  push:
    branches: [main]
    paths:
      - "infra/**"

permissions:
  id-token: write
  contents: read

jobs:
  apply-foundation:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<account>:role/cerberus-ci-apply
          aws-region: ap-south-1
      - uses: opentofu/setup-opentofu@v1
        with:
          tofu_version: "1.7.x"
      - run: tofu init
        working-directory: infra/envs/dev/00-foundation
      - run: tofu apply -auto-approve
        working-directory: infra/envs/dev/00-foundation

  # Stubs for Week 2 — wired up when the stacks exist
  apply-iam:
    needs: apply-foundation
    runs-on: ubuntu-latest
    if: false   # remove when 10-iam stack is ready
    steps:
      - run: echo "10-iam — Week 2"

  apply-compute:
    needs: apply-iam
    runs-on: ubuntu-latest
    if: false
    steps:
      - run: echo "20-compute — Week 2"
```

### Merge and verify

```bash
# After merge and apply run:

# 1. SSM params exist
aws ssm get-parameter \
  --name /cerberus/dev/foundation/bronze-bucket-arn \
  --profile cerberus

# 2. Buckets visible to runtime identity
aws s3 ls --profile cerberus

# 3. Runtime identity cannot create infrastructure (must get AccessDenied)
aws s3api create-bucket \
  --bucket cerberus-deny-test \
  --region ap-south-1 \
  --profile cerberus
# Expected: An error occurred (AccessDenied)
```

### Exit signal

- Foundation stack applied via CI (visible in GitHub Actions run log).
- All six S3 buckets + Glue DB + log groups exist in `ap-south-1`.
- SSM params resolve with no hardcoded ARNs.
- `cerberus-cli` `create-bucket` call returns `AccessDenied`.
- Budgets alert + Cost Anomaly Detection visible in AWS Cost Management console.

**Time estimate:** 1.5–2 hours

---

## Week 1 total time estimate

| Day | Task | Estimate |
|-----|------|----------|
| 1 | Scaffolding + pre-commit | 1–1.5 h |
| 2 | Bootstrap (OIDC + state) | 1.5–2 h |
| 3 | s3-bucket module + 00-foundation | 1.5–2 h |
| 4 | SSM exports + plan.yml | 1–1.5 h |
| 5 | Policy gates (OPA + trivy + docs) | 1.5–2 h |
| 6 | apply.yml + cost guardrails + verify | 1.5–2 h |
| **Total** | | **8–11 hours** |
