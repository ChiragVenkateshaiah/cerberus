# Cerberus — Platform Implementation Plan

**Status:** Draft for review
**Audience:** Implementer (future self) + portfolio reviewer
**Companion docs:** [`roadmap.md`](roadmap.md) (sequencing), [`docs/architecture.md`](docs/architecture.md)
(diagrams), ADRs 002/005/006/007/008/010/011/012 + new 013–019

A serverless data lakehouse delivered as a platform-engineering lab. This is the technical
specification behind the roadmap: the IaC layout, CI/CD, the data pipeline, orchestration,
data quality, observability, IAM, and cost governance.

Conventions:
- All resources tagged `Project=cerberus`, `Environment=<env>`, `ManagedBy=opentofu` (ADR 003/019).
- Bucket names shown as logical (`cerberus-bronze`); real names need a globally-unique
  suffix (e.g. account id or `-apsouth1`). Decide once, keep consistent.
- Provisioning runs through CI via an OIDC-assumed role (ADR 014); runtime is profile
  `cerberus` (ADR 010). No static admin keys.

---

## 0. Repository & environment layout (ADR 013)

```
infra/
  bootstrap/                 # one manual step: OIDC provider + CI role + state bucket
  modules/                   # reusable, tofu-tested modules
    s3-bucket/  glue-database/  iam-role/  emr-serverless-app/
    athena-workgroup/  step-function-pipeline/
  envs/
    dev/   00-foundation/ 10-iam/ 20-compute/ 30-orchestration/
    prod/  (structure mirrors dev; applied later)
.github/workflows/           # plan.yml (PR) · apply.yml (merge)
policy/                      # OPA/conftest rules
pipelines/                   # bronze/ silver/ gold/ conf/  (PySpark + PyDeequ)
sql/                         # bronze/ silver/ gold/  (Athena saved queries)
observability/               # dashboard + alarms JSON
docs/                        # adr/ runbooks/ benchmarks/ architecture.md
```

SSM naming now includes the environment: `/cerberus/<env>/<layer>/<resource>-arn`
(e.g. `/cerberus/dev/foundation/bronze-bucket-arn`). The env segment is the change that
lets the same modules serve dev and prod without duplication.

---

## 1. Infrastructure stacks (per env — ADR 012/013)

Apply order, enforced by the apply workflow:
**`00-foundation → 10-iam → 20-compute → 30-orchestration`**. Stacks are decoupled at the
state layer; they communicate only through SSM parameters.

### `00-foundation/` — changes almost never
**Resources**
- Six S3 buckets via the `s3-bucket` module (block-public-access, versioning, SSE,
  noncurrent-version expiry — ADR 002): `cerberus-bronze`, `cerberus-silver`,
  `cerberus-gold`, `cerberus-scripts`, `cerberus-athena-results`, `cerberus-emr-logs`.
- Glue Data Catalog database `cerberus_lakehouse` (ADR 005/006).
- CloudWatch log groups `/cerberus/<env>/emr-serverless`, `/cerberus/<env>/athena` (ADR 007).
- AWS Budgets monthly alert + Cost Anomaly Detection monitor (ADR 019).

**Publishes** `/cerberus/<env>/foundation/<resource>-arn`:
`bronze-bucket-arn`, `silver-bucket-arn`, `gold-bucket-arn`, `scripts-bucket-arn`,
`athena-results-bucket-arn`, `emr-logs-bucket-arn`, `glue-catalog-database-arn`.

**Consumes:** nothing (root of the dependency chain).

### `10-iam/` — changes moderately
**Resources**
- `cerberus-emr-serverless-execution-role` (trusted by `emr-serverless.amazonaws.com`):
  `s3:Get/Put/Delete/ListBucket` on the data buckets; Glue read/write on `cerberus_lakehouse`
  (ADR 008 Path B).
- `aws_iam_user_policy` on `cerberus-cli` — the "operate the platform" runtime grant (§7).
- CI role's scoped IAM-write: `iam:PutRolePolicy` on the EMR exec role + `iam:PutUserPolicy`
  on `cerberus-cli`, **exactly two named ARNs**, not `IAMFullAccess` (ADR 010 B / ADR 014).

**Publishes** `/cerberus/<env>/iam/emr-serverless-execution-role-arn`.
**Consumes:** all `/cerberus/<env>/foundation/*-bucket-arn` + `glue-catalog-database-arn`.

### `20-compute/` — changes constantly
**Resources**
- `aws_emrserverless_application` (Spark) — the named container jobs submit to (ADR 005).
- `aws_athena_workgroup` (engine v3, Iceberg) — result config → athena-results bucket;
  `enforce_workgroup_configuration = true`; per-query bytes-scanned cap as a budget guard (ADR 006/019).

**Publishes:** EMR application id (for `run-job.sh` and the state machine).
**Consumes:** `/cerberus/<env>/iam/emr-serverless-execution-role-arn`,
`/cerberus/<env>/foundation/athena-results-bucket-arn`.

### `30-orchestration/` — changes occasionally
**Resources**
- `aws_sfn_state_machine` — the medallion pipeline (one EMR job-run per step, with the
  PyDeequ gate as a failing state) (ADR 017).
- `aws_cloudwatch_event_rule` (EventBridge schedule) + target → state machine.
- Step Functions execution role; failure catch → SNS/CloudWatch notify.

**Consumes:** EMR application id + execution role ARN via SSM.

**Why this layering matters (ADR 012):** a routine compute tweak produces a ~10–15 line
plan diff touching only EMR/Athena, not a 200-line diff re-planning foundational buckets and
IAM. The SSM contract — not `terraform_remote_state` — keeps each consumer needing only
`ssm:GetParameter`, never read access to another stack's state file.

---

## 2. CI/CD & identity (ADR 014/015)

- **Bootstrap (manual, one step — the only ClickOps, ADR 011 updated by 014):** GitHub OIDC
  identity provider + `cerberus-ci` assumable role with a trust policy scoped to this repo
  and a protected branch. No static keys anywhere.
- **`plan.yml` (on PR):** OIDC-assume → `tofu fmt -check`, `tflint`, `trivy`, `conftest`,
  `tofu plan`; post the plan as a PR comment; block on policy failure.
- **`apply.yml` (on merge to `main`):** OIDC-assume → `tofu apply` per stack in dependency
  order, `dev` env. A `prod` apply is gated behind a manual-approval GitHub Environment when added.
- The CI role replaces `cerberus-admin`'s static keys as the provisioning principal. The
  *separation* of provisioning vs runtime identity (ADR 010) is unchanged — only the
  credential mechanism evolves to short-lived OIDC tokens.

---

## 3. Policy-as-code & IaC testing (ADR 016)

- **tflint** — provider lint, naming conventions, deprecations.
- **trivy** (or checkov) — security scan: public exposure, missing encryption, open SGs.
- **conftest/OPA** (`policy/`) — org rules: no public buckets; every resource carries the
  three required tags; Athena workgroup enforces config; no `IAMFullAccess`.
- **terraform-docs** — auto-generated, checked-in module docs.
- **`tofu test`** — contract tests on the `s3-bucket` and `iam-role` modules.
- **pre-commit** — runs fmt / tflint / terraform-docs locally before push.

---

## 4. Data pipeline — bronze → silver → gold (NYC Yellow Taxi)

All tables are Apache Iceberg in the `cerberus_lakehouse` Glue catalog (ADR 002/005).

### Bronze — `pipelines/bronze/ingest_yellow_trips.py`
- **Input:** TLC monthly yellow-trip Parquet (start with 1–3 months).
- **Transform:** minimal — add `ingestion_ts`, `source_file`; preserve raw schema.
- **Output:** Iceberg `bronze.yellow_trips_raw` at `s3a://cerberus-bronze/...`, partitioned
  by `year, month`.
- **Verify:** `sql/bronze/verify_yellow_trips.sql` (row count + date range).

### Silver — `pipelines/silver/build_yellow_trips.py`
- **Transform:** cast types; drop invalid rows (negative/zero fares, zero-distance,
  out-of-range pickup/dropoff timestamps); dedupe; standardise columns; derive
  `trip_duration_min`, `pickup_date`.
- **Output:** Iceberg `silver.yellow_trips`, partitioned by `pickup_date` (the partition-
  pruning lever for Athena cost, ADR 006).
- **Data-quality gate (ADR 018):** PyDeequ checks — completeness, fare/distance bounds,
  uniqueness. A failing check fails the job (and the orchestration step).

### Gold — `pipelines/gold/*.py`
- `daily_revenue_by_payment_type.py` → `gold.daily_revenue_by_payment_type`
- `trips_by_hour_of_day.py` → `gold.trips_by_hour_of_day`
- (stretch) `gold.zone_daily_rollup` with a zone lookup join.

### Athena surface
- `sql/gold/` holds committed views + the canonical analytical queries — the saved-query
  library (ADR 006).
- `docs/benchmarks/` compares bytes scanned with/without partition pruning.

### Shared job structure
- `pipelines/conf/` env-switch: local MinIO (`MINIO_ENDPOINT_URL`, `s3a://`) vs remote S3 +
  IAM execution role (ADR 002).
- Iceberg Spark runtime + Glue catalog config set per job.
- `run-job.sh`: wraps `start-job-run --execution-role-arn <SSM value>` + `get-job-run`
  polling; logs to `/cerberus/<env>/emr-serverless`.

---

## 5. Orchestration (ADR 017)

- **Step Functions** state machine: `Bronze → DQ gate → Silver → DQ gate → Gold`, each an
  EMR Serverless job-run task, with `Retry`/`Catch` and a failure-notify state.
- **EventBridge** schedule triggers the machine (e.g. daily). No idle cost — pay per state
  transition. This is the explicit reason it supersedes ADR 009's MWAA rejection (which was
  about idle cost).
- Manual `start-job-run` remains available for ad-hoc single-step runs.

---

## 6. Observability & cost governance (ADR 007/019)

**Log groups** (created in `00-foundation`): `/cerberus/<env>/emr-serverless`,
`/cerberus/<env>/athena`.

**Dashboard — `cerberus-platform`** (`observability/`, committed JSON):
- EMR job runs by state (success/failure) over time + duration trend.
- Athena `DataScannedInBytes` per query + a derived estimated-cost figure.
- Step Functions execution success/failure + duration.
- Recent failure-log excerpts (Logs Insights widget).

**Alarms / guardrails:**
- AWS Budgets monthly alert at the ceiling — mandatory mitigation (ADR 010).
- Cost Anomaly Detection monitor (ADR 019).
- Per-query Athena bytes-scanned cap on the workgroup.
- CloudWatch alarm on repeated EMR / Step Functions failures.

---

## 7. IAM checklist (ADR 008/010/014)

### `cerberus-cli` — the "operate the platform" runtime set
Implemented as `aws_iam_user_policy` in `10-iam/`:
- **EMR Serverless:** `StartJobRun`, `GetJobRun`, `ListJobRuns`, `GetApplication`,
  `StartApplication`, `StopApplication`.
- **Athena:** `StartQueryExecution`, `GetQueryExecution`, `GetQueryResults`,
  `StopQueryExecution`, `GetWorkGroup`.
- **Glue (read):** `GetDatabase(s)`, `GetTable(s)`, `GetPartition(s)`.
- **S3:** `Get/Put/DeleteObject` + `ListBucket` on the six Cerberus buckets.
- **CloudWatch Logs (read):** `GetLogEvents`, `FilterLogEvents`, `DescribeLogStreams`.
- **Step Functions:** `StartExecution`, `DescribeExecution` (trigger/inspect the pipeline).
- **`iam:PassRole`:** scoped to `cerberus-emr-serverless-execution-role` only.
- **Never:** create/modify/destroy infrastructure, create IAM, touch NovaPay (ADR 003/010).

### CI provisioning role (OIDC — ADR 014)
- Broad provisioning across S3/Glue/EMR/Athena/States/EventBridge/CloudWatch.
- Scoped IAM-write on exactly the two named ARNs (EMR exec role + `cerberus-cli`).
- Assumed via OIDC short-lived tokens; no static keys; cannot reach `novapay-cli`.

### EMR execution role
- `s3` RW on the data buckets; Glue catalog RW on `cerberus_lakehouse`.

### Bootstrap (manual, one step — ADR 011/014)
- OIDC provider + CI role, created by root/admin, recorded in `docs/runbooks/bootstrap.md`.

---

## 8. Portfolio / resume signal callouts

| Signal | Why it reads as production-grade |
|---|---|
| OIDC-federated CI/CD, zero static keys | Current AWS best practice; the single biggest "this is real" marker |
| Policy-as-code gates (OPA / trivy / tflint) in CI | Shipping guardrails, not just config — platform-team hygiene |
| Layered, multi-env Tofu with reusable tested modules | Mirrors platform-vs-product boundaries; prod-ready by structure |
| Provisioning/runtime identity separation | Exact deploy-vs-runtime pattern, now with the OIDC end-state realized |
| Serverless orchestration via Step Functions | Event-driven, no-idle-cost — the right tool, chosen with documented reasoning |
| Data quality as code (PyDeequ gates) | Trust boundaries between medallion layers, not blind transforms |
| Cost governance (Budgets + Anomaly Detection + bytes cap) | Cost-aware design, demonstrably enforced |
| Iceberg medallion on a shared Glue catalog | One catalog, two engines — current AWS-native practice |
| A 19-ADR decision trail incl. superseded designs | Genuine engineering judgment, not a tutorial clone |
