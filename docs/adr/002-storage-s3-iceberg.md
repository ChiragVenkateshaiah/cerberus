# ADR 002 — Storage: S3 + Apache Iceberg (Not MinIO on AWS)

**Status:** Accepted — unchanged through the serverless redesign in ADR 005/006
**Date:** 2026-06-20

---

## Context

The lakehouse storage layer needs a place to land raw data (bronze), transformed data
(silver/gold), and a table format that provides ACID transactions, schema evolution, and
time travel — the properties that distinguish a lakehouse from a plain data lake.

Two dimensions to decide: the object store, and the table format.

### Object store

**MinIO** is the natural "run everything locally" option. It implements the S3 API, can
run in a Docker container, and enables a fully local dev environment on WSL2. The
question is whether to use it on AWS as well.

The answer is no. MinIO on AWS would mean:
- Running an object store *inside EC2* to emulate S3, while real S3 sits in the same
  account one API call away.
- Paying for EBS storage to back MinIO on top of the S3 costs you'd pay anyway.
- Diverging the local and remote environments in a subtle way — the MinIO behavior is
  not identical to S3 in edge cases (multipart, SSE, ACLs).

**S3** is chosen for AWS deployments. MinIO is reserved for local WSL2 parity only,
where S3 would introduce latency and per-request cost into iterative dev loops.

### Table format

**Apache Iceberg** is chosen over Delta Lake and Apache Hudi.

- **Native Spark support:** Iceberg has first-class Spark integration via
  `org.apache.iceberg:iceberg-spark-runtime`.
- **Trino support:** The Trino Iceberg connector works against S3 + the chosen catalog
  without glue.
- **Glue compatibility:** AWS Glue Data Catalog natively understands Iceberg table
  metadata when the Iceberg SDK writes it — no custom serde required.
- **Ecosystem trajectory:** Iceberg is where AWS is investing (native EMR Serverless
  support, Athena Iceberg support, S3 Tables preview). Being on Iceberg keeps the door
  open to those services without a format migration.

Delta Lake was not chosen: it works well in a Databricks-centric shop, but Trino's Delta
connector lags behind Iceberg's, and this project deliberately avoids Databricks outside
of any separate Databricks notebook track.

## Decision

- **Object store on AWS:** Amazon S3, four buckets — `bronze`, `silver`, `gold`,
  `scripts` — provisioned by OpenTofu in `infra/`.
- **Object store locally (WSL2):** MinIO running in Docker, configured to match the
  same bucket names and credential patterns.
- **Table format:** Apache Iceberg throughout the bronze → silver → gold pipeline.
- **Bucket access policy:** Buckets are private. Access is through IAM roles only
  (instance role for EC2, execution role for EMR Serverless, service role for Athena).
  No public access, no bucket ACLs.
- **Lifecycle:** Noncurrent-version expiry configured to avoid unbounded storage growth
  on Silver/Gold tables where Iceberg writes many snapshot files.

## Consequences

- All pipeline code targeting S3 uses `s3a://` URIs with the Hadoop S3A connector.
- Local dev must set `MINIO_ENDPOINT_URL` and swap credentials for MinIO; production
  uses IAM roles and real S3 endpoints. A `conf/` layer in each pipeline script handles
  this with an environment flag.
- S3 storage cost is low at this scale (single learner, moderate data volume) but is
  tracked in `docs/cost-model.md`.
- Iceberg snapshot accumulation is a real operational concern at scale; the lifecycle
  rule mitigates it, and `CALL catalog.system.expire_snapshots(...)` is the manual
  cleanup path if needed.

**This decision is not revisited by the compute/query redesign in ADR 005/006.** S3 +
Iceberg was never the cost problem — it was always the right call.
