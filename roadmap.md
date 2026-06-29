# Cerberus — Project Roadmap

**Status:** Draft for review
**Scope:** A serverless data lakehouse built as a full platform-engineering lab —
IaC, CI/CD, policy-as-code, orchestration, data quality, and cost governance.
**Cadence:** Solo, evening/weekend work, ~8 calendar weeks
**Region:** ap-south-1 (Mumbai) · **Budget ceiling:** ~$15–25/month
**Dataset:** NYC TLC Yellow Taxi (public Parquet, partitioned by year/month)

Phases carry the exit criteria. Weeks are loose containers — if a week slips, it slips.
Days are enumerated for Week 1 only, as a concrete example of the task granularity to
aim for. Everything maps back to a finalised ADR.

This is deliberately built as a Cloud Computing Lab: the data pipeline is the workload,
but the point is the platform around it — provisioned, deployed, governed, and observed
the way a real AWS environment is. Companion docs: [`platform-plan.md`](platform-plan.md)
(technical spec) and [`docs/architecture.md`](docs/architecture.md) (diagrams).

---

## Dataset choice — NYC TLC Yellow Taxi

Picked over OpenWeather and GitHub Archive as the best fit for an Iceberg + Athena
medallion demo:

- **Already Parquet, already partitioned** by year/month — a realistic ingest, not a
  CSV toy. The bronze job focuses on Iceberg registration, not parsing.
- **Rich silver cleaning:** negative fares, zero-distance trips, out-of-range timestamps,
  and vendor/payment-type code columns give genuine conform/validate work.
- **Natural gold aggregations** that read well on a resume: daily revenue, trips per
  hour-of-day, average fare by payment type, zone-level rollups.
- **Partition-pruning story for Athena:** partitioning silver by pickup date directly
  demonstrates the "good Iceberg design lowers bytes scanned, lowers cost" point that
  ADR 006 makes load-bearing.
- **Bounded volume:** pull 1–3 months first; the cost model stays inside the ceiling.

OpenWeather needs an ingestion API loop (orchestration pressure). GitHub Archive is large
and JSON-heavy — more parsing pain, weaker partition-pruning narrative. Taxi data wins.

---

## Phase 1 — Foundation, Identity & CI/CD  (Weeks 1–2)

**Goal:** Stand up the layered, multi-environment OpenTofu structure (ADR 012/013), the
two-identity model (ADR 010), OIDC-federated CI/CD (ADR 014/015), and policy-as-code
guardrails (ADR 016). Everything from here is deployed through the pipeline, not a laptop.

**Exit criterion:**
- The one manual bootstrap (ADR 011, updated by ADR 014) — GitHub OIDC provider +
  assumable CI role + state bucket — is created and recorded in a runbook. No static
  admin keys exist.
- A merge to `main` triggers GitHub Actions, which assumes the OIDC role and applies
  `00-foundation → 10-iam → 20-compute` to the `dev` environment in dependency order.
- Every PR runs `tofu plan` + tflint + trivy + conftest(OPA) + terraform-docs and blocks
  on policy failure (e.g. a public bucket or an untagged resource).
- `cerberus-cli` (profile `cerberus`) can operate the platform but cannot apply Tofu or
  create IAM (verified by a denied call).
- Cross-stack SSM lookups resolve with no hardcoded ARNs. Budgets + Cost Anomaly
  Detection are live (ADR 019).

### Week 1 — Bootstrap, OIDC, CI/CD skeleton + foundation stack

**Week goal:** Cross the bootstrap boundary (ADR 011/014), and get a policy-gated CI/CD
pipeline applying the foundation stack to `dev` via OIDC.

Deliverables:
1. Repo scaffolding: `infra/modules/` + `infra/envs/dev/`, pre-commit, `.tflint.hcl`,
   pinned providers, `Project`/`Environment`/`ManagedBy` default tags (ADR 013/016).
2. Manual bootstrap: GitHub OIDC provider + minimal assumable CI role; OpenTofu state
   bucket; recorded in `docs/runbooks/bootstrap.md` (ADR 011/014).
3. `infra/envs/dev/00-foundation/` built from a reusable `s3-bucket` module: six buckets,
   Glue database, CloudWatch log groups, SSM exports under `/cerberus/dev/foundation/*`.
4. GitHub Actions: `plan` on PR, `apply` on merge to `main`, OIDC-assumed role (ADR 015).
5. Policy-as-code in CI: trivy + conftest/OPA ("no public bucket", "must be tagged") +
   terraform-docs; first PR goes green (ADR 016).

*(Day-by-day breakdown for Week 1 below.)*

### Week 2 — IAM + compute stacks + IaC testing

**Week goal:** Complete the layered IaC — execution roles, runtime grant, EMR Serverless
application, Athena workgroup — all delivered through CI, with module tests.

Deliverables:
1. `10-iam/`: `cerberus-emr-serverless-execution-role`, `cerberus-cli` runtime grant,
   and the CI role's scoped IAM-write on exactly the named ARNs (ADR 008 Path B, ADR 010 B).
2. `20-compute/`: EMR Serverless Spark application + Athena workgroup (engine v3), wired
   via SSM (ADR 005/006).
3. `tofu test` unit/contract tests for the `s3-bucket` and `iam-role` modules (ADR 016).
4. Negative test: `cerberus-cli` denied apply/role-creation. Tag + policy audit green.
5. `prod/` environment directory stubbed (structure only) to prove the layout scales (ADR 013).

---

## Phase 2 — Data Plane & Bronze  (Week 3)

**Goal:** Submit the first EMR Serverless job as `cerberus-cli` and land the first Iceberg
table in bronze, queryable in Athena through the shared Glue catalog.

**Exit criterion:**
- `aws emr-serverless start-job-run` (profile `cerberus`, execution role from `10-iam`)
  succeeds end to end.
- `bronze.yellow_trips_raw` Iceberg table exists and returns rows from Athena.
- Driver/executor logs land in `/cerberus/dev/emr-serverless` (ADR 007).

### Week 3 — Bronze ingestion

**Week goal:** Stand up the PySpark job pattern, the `conf/` env-switching layer
(MinIO local vs S3 remote, ADR 002), and write the bronze Iceberg table.

Deliverables:
1. `pipelines/bronze/ingest_yellow_trips.py`: TLC Parquet → Iceberg `bronze.yellow_trips_raw`,
   partitioned by year/month; adds `ingestion_ts`, `source_file`.
2. `pipelines/conf/` env-switch: MinIO locally vs IAM execution role + real S3 remotely (ADR 002).
3. Artifacts staged to `cerberus-scripts`; first `start-job-run` via a `run-job.sh` helper.
4. `sql/bronze/verify_yellow_trips.sql` Athena sanity query (row count, date range).

---

## Phase 3 — Medallion Pipeline + Data Quality  (Weeks 4–5)

**Goal:** Build bronze→silver→gold in Iceberg with PyDeequ data-quality gates between
layers, and an Athena query surface over gold. This is the analytical core of the portfolio.

**Exit criterion:**
- `silver.yellow_trips` + at least two `gold.*` tables exist; PyDeequ checks pass as a
  gate (ADR 018).
- The full chain runs manually end to end at least once.
- Athena gold queries visibly prune partitions (lower bytes scanned vs an unpartitioned scan).

### Week 4 — Silver (clean & conform) + data quality

Deliverables:
1. `pipelines/silver/build_yellow_trips.py`: cast types, filter invalid rows (negative/zero
   fares, zero-distance, out-of-range timestamps), dedupe, derive `trip_duration_min` /
   `pickup_date`; Iceberg `silver.yellow_trips` partitioned by pickup date.
2. PyDeequ checks (completeness, fare/distance bounds, uniqueness) as a build gate (ADR 018).
3. `sql/silver/` DQ queries; Iceberg snapshot expiration documented (ADR 002).

### Week 5 — Gold (aggregate) + Athena surface

Deliverables:
1. `pipelines/gold/`: `daily_revenue_by_payment_type.py`, `trips_by_hour_of_day.py` → `gold.*`.
2. `sql/gold/` committed views + canonical analytical queries (ADR 006).
3. `docs/benchmarks/`: partition-pruning effect on Athena bytes scanned.

---

## Phase 4 — Orchestration  (Week 6)

**Goal:** Replace manual submission with a Step Functions state machine on an EventBridge
schedule — the serverless orchestrator ADR 009 was waiting for (ADR 017).

**Exit criterion:**
- `infra/envs/dev/30-orchestration/` deploys a Step Functions state machine that runs
  bronze → silver → gold, each step gated on the prior succeeding.
- An EventBridge schedule triggers the chain; a failed step stops the machine and alarms.
- ADR 009 is updated to "superseded by ADR 017".

### Week 6 — Step Functions + EventBridge

Deliverables:
1. `30-orchestration/` stack: state machine (one EMR job per medallion step), EventBridge
   schedule rule, execution role, failure catch/notify (ADR 017).
2. State machine reads job config from SSM; integrates the PyDeequ gate as a failing state.
3. End-to-end scheduled run; capture an execution-graph screenshot for the README.

---

## Phase 5 — Observability, Cost Governance & Portfolio Polish  (Weeks 7–8)

**Goal:** Make the platform legible and interview-ready: dashboards, alarms, cost
governance, README with architecture diagrams, and honest runbooks.

**Exit criterion:**
- `cerberus-platform` CloudWatch dashboard shows EMR outcomes/duration, Athena bytes
  scanned, and Step Functions execution health (ADR 007).
- Budgets + Cost Anomaly Detection + a job-failure alarm are live (ADR 019).
- README carries the architecture diagrams and links the full ADR trail.
- A cold reader reconstructs the whole environment from `infra/` + ADRs (ADR 011).

### Week 7 — Dashboards, alarms, cost governance

Deliverables:
1. CloudWatch dashboard JSON in `observability/` (EMR, Athena bytes, Step Functions).
2. Cost Anomaly Detection monitor + per-query Athena bytes cap + cost-allocation tags (ADR 019).
3. Alarm on repeated EMR / Step Functions failures.

### Week 8 — README, diagrams, prod-readiness, teardown rehearsal

Deliverables:
1. README: overview, architecture diagrams, medallion story, ADR index, honest bootstrap note.
2. `docs/runbooks/operate.md`: submit a job, run a query, read logs, tear down.
3. `tofu destroy` rehearsal on `dev` to prove reproducibility; final tag + cost audit.

---

## Week 1 — day-by-day

Each day = one focused 1–2 hour evening block.

**Day 1 — Repo scaffolding & local guardrails (ADR 013/016)**
- Create `infra/modules/` + `infra/envs/dev/` layout; `.pre-commit-config.yaml`,
  `.tflint.hcl`, pinned providers, default tags (`Project`/`Environment`/`ManagedBy`).
- `pre-commit run --all-files` green locally.

**Day 2 — Manual bootstrap: OIDC + state backend (ADR 011/014)**
- With root/admin: create the GitHub OIDC identity provider + a minimal assumable
  `cerberus-ci-bootstrap` role; create the versioned, private OpenTofu state bucket.
- Record the exact steps + trust policy in `docs/runbooks/bootstrap.md`.

**Day 3 — Foundation stack from a reusable module (ADR 012/013)**
- Write `infra/modules/s3-bucket/` (block-public-access, versioning, SSE, lifecycle).
- `infra/envs/dev/00-foundation/`: six buckets via the module, Glue db, log groups.
  `tofu plan` clean locally.

**Day 4 — SSM exports + CI plan workflow (ADR 012/015)**
- Publish `/cerberus/dev/foundation/<resource>-arn` SSM params.
- `.github/workflows/plan.yml`: on PR, OIDC-assume the role, `tofu plan`, post the plan.

**Day 5 — Policy-as-code gates (ADR 016)**
- Add trivy + conftest/OPA policies ("no public bucket", "all resources tagged") +
  terraform-docs to CI; open a PR and make the gates pass.

**Day 6 — Merge → CI applies + cost guardrails (ADR 015/019)**
- `.github/workflows/apply.yml`: on merge to `main`, OIDC-assume + `tofu apply` dev.
- Verify foundation applied + SSM params exist; create Budgets alert + Cost Anomaly
  Detection monitor. Confirm `cerberus-cli` can list but not modify. Open the first real PR.

---

## ADR trail referenced by this roadmap

Existing: 001–012 (see `docs/adr/`). New decisions introduced by this Cloud-Lab scope,
to be written as a separate reviewed batch:

| ADR | Decision |
|---|---|
| 013 | Multi-environment layout: `modules/` + `envs/{dev,prod}/<stack>` (extends 012) |
| 014 | OIDC federation for CI provisioning; retires static admin keys (supersedes auth part of 010) |
| 015 | GitHub Actions CI/CD: plan-on-PR, apply-on-merge, per-env gating |
| 016 | Policy-as-code: tflint, trivy, conftest/OPA, terraform-docs, pre-commit, `tofu test` |
| 017 | Step Functions + EventBridge orchestration (supersedes 009) |
| 018 | Data-quality gates with PyDeequ between medallion layers |
| 019 | Cost governance: Budgets + Cost Anomaly Detection + tagging policy + Athena bytes cap (extends 010) |
