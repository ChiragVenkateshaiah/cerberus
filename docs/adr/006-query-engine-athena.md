# ADR 006 — Query Engine: Athena Replaces Trino-on-EC2

**Status:** Accepted — supersedes ADR 001 (query engine portion)
**Date:** 2026-06-20

---

## Context

ADR 001 included Trino as the interactive SQL query engine, running as a Docker container
on the same t3.xlarge as Spark, Airflow, and the rest of the stack. Trino's JVM heap
alone requires 3–4 GB, which was a significant line item in the RAM budget that triggered
ADR 004.

With the compute approach pivoting to serverless in ADR 005, the natural question was
whether Trino-on-EC2 still made sense as the query engine, or whether a serverless SQL
option existed that addressed the same constraints.

## Decision

Replace Trino-on-EC2 with **Amazon Athena** for ad-hoc and exploratory SQL queries
against S3 + Iceberg.

### Why Athena

- **Per-query billing, zero idle cost:** Athena charges per TB of data scanned (verify
  current `ap-south-1` rate against AWS Pricing Calculator — do not treat any specific
  number as confirmed). When you're not querying, you pay nothing. This directly mirrors
  the structural fix made for compute in ADR 005.
- **Native Iceberg support:** Athena v3 (the current engine) supports Apache Iceberg
  tables. Queries land directly against the Glue Data Catalog where EMR Serverless also
  writes — one catalog, consistent table definitions.
- **Zero infrastructure:** No cluster to provision, no JVM to tune, no container to start.
  Run a query from the console or via the Athena API. Results land in an S3 output bucket.
- **SAA exam relevance:** Athena is directly tested on the AWS Solutions Architect
  Associate exam in the context of data lake query patterns. Running it for real
  reinforces the same concepts the exam covers.

### What Trino offers that Athena does not

Trino supports ANSI SQL more completely, has no per-query billing surprise, and supports
more complex session-state patterns (prepared statements, transactions). For a production
multi-tenant environment, Trino is a serious option.

For this project's usage — single user, exploratory queries, bounded data volume, strict
cost ceiling — those Trino advantages are not material. The billing model difference is.

### On Athena pricing

Per-query Athena billing rewards efficient Iceberg usage: partition pruning, column
projection via Parquet, and Iceberg's metadata-driven scan optimization all directly reduce
the bytes scanned and therefore the cost. This makes "write efficient Iceberg tables" a
cost-saving habit, not just a best practice. Verify current `ap-south-1` scan rates at the
AWS Pricing Calculator before budgeting.

### dbt

dbt was included in ADR 001's stack with the `dbt-trino` adapter. With Trino replaced by
Athena, the adapter changes to `dbt-athena`. The dbt model content (silver/gold transforms)
is a separate learning track and is out of scope for this ADR. The adapter choice is a
one-line change in `dbt/profiles.yml`.

## Consequences

- **Positive:** No always-on query coordinator. No JVM memory allocation. SQL queries
  are fire-and-forget via console or CLI.
- **Positive:** Athena + EMR Serverless + Glue catalog is a coherent, integrated AWS
  native stack that reflects current AWS data engineering practice.
- **Negative:** Each exploratory query has a cost event. Scanning large unbucketed tables
  on Athena is more expensive than on a self-hosted Trino against the same data. Good
  Iceberg table design mitigates this; bad table design punishes it.
- **Negative:** No persistent query history or saved query library on Athena beyond the
  S3 output bucket and the console's query history. Queries worth keeping go into `sql/`
  as committed `.sql` files — this is the existing repo convention.
- **Note:** Trino is not ruled out permanently. If a future learning goal specifically
  requires Trino's SQL dialect or a federated query capability Athena does not offer,
  revisiting is straightforward. That is a future decision, not a closed door.
