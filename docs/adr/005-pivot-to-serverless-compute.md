# ADR 005 — Pivot to Serverless Compute: EMR Serverless Replaces EC2-Hosted Spark

**Status:** Accepted — supersedes ADR 001 (compute approach)
**Date:** 2026-06-20

---

## Context

ADR 004 surfaced the concrete problem: the full OSS stack on a single t3.xlarge runs out of
RAM under a real integrated pipeline run. The RAM fix (t3.2xlarge) roughly doubles EC2 cost
and undermines the budget that made the single-box approach look viable in ADR 001.

Before accepting a higher cost, it was worth asking whether the premise of the original
decision — "run OSS services on a box" — was actually the right fit for this project's real
usage pattern and learning goals.

**The actual usage pattern:**
- A few hours per day, not all hours every day
- Single user; no concurrency requirements
- Pipeline runs are discrete events (ingest bronze, transform to silver, aggregate to gold),
  not a continuously running workload
- The learning goal is mastering PySpark, SQL, and AWS architecture — not operating a cluster

**The cost observation:** an EC2 instance that is stopped between sessions still bills for
EBS storage at rest, and must be running (billing) for the full duration of any session,
whether Spark is actively computing or you are editing a Python file. A t3.xlarge at
~$0.1856/hr running 4 hours/day over 30 days is ~$22/month — already above the $15/month
ceiling, before EBS and data transfer.

**The learning goal observation:** the AWS Solutions Architect Associate exam tests
knowledge of managed/serverless AWS services — EMR Serverless, Athena, Glue — not
Apache Spark's Docker container configuration or JVM memory tuning. Running OSS on EC2
teaches the OSS tools; running on managed AWS services teaches the managed AWS services.
The redesign can solve the cost problem and simultaneously make the learning more directly
relevant to the exam and job market.

## Decision

Replace EC2-hosted Spark (Docker container, ADR 001) with **Amazon EMR Serverless**.

### What EMR Serverless is

EMR Serverless runs Spark (and Hive) applications without provisioning or managing clusters.
You create an Application (a named container for your jobs), submit Spark jobs to it via the
API or console, and pay only for the resources used while the job actually runs.

- **Billing model:** per-second billing for vCPU and memory, with a minimum of 1 minute per
  worker. There is no charge when no job is running.
- **No always-on cost:** unlike EMR on EC2 clusters, there is no master/core node running
  at rest. Stop submitting jobs, pay nothing.
- **Native Iceberg support:** EMR Serverless supports Apache Iceberg natively; no custom
  jar management required.
- **Glue Data Catalog integration:** the same Glue catalog used for Athena (ADR 006) is
  used for EMR Serverless jobs. One catalog, two compute engines.

### What is deferred or removed

- **EC2 instance:** no longer the primary compute platform for Spark workloads. The EC2
  instance (and its associated OpenTofu module) is removed from the architecture scope.
  If a persistent bastion or development box is needed in the future, that is a separate
  decision.
- **Docker Compose stack:** removed as the runtime model for compute. Docker may still be
  used locally for `dbt` or lightweight tools, but is no longer the production-facing
  architecture.
- **Airflow, Trino, Superset:** deferred (see ADR 009 for orchestration; ADR 006 for
  query engine; Superset is deferred indefinitely pending a reassessment of the full stack).

### What is retained from ADR 001

- S3 + Iceberg as the storage and table format layer (ADR 002, unchanged).
- AWS Glue Data Catalog as the Iceberg catalog (confirmed in pre-planning).
- `Project=cerberus` tagging on all resources (ADR 003, unchanged).
- Four S3 buckets: bronze, silver, gold, scripts.
- The `cerberus-cli` IAM user and scoped permissions model (ADR 003, amended by ADR 008
  for the new service execution roles this redesign requires).

### On EMR Serverless pricing

**Do not rely on any specific dollar figure in this ADR as confirmed.** EMR Serverless
billing in `ap-south-1` (Mumbai) should be verified against the AWS Pricing Calculator
before committing to a budget estimate. What can be stated directionally:
- At the scale of a solo learner (jobs that run for minutes, not hours; a handful of
  runs per session), per-second EMR Serverless billing will be materially less per month
  than a t3.xlarge running for 4 hours/day every day.
- The no-idle-cost property is the structural change that makes the serverless model fit
  the actual usage pattern.
- Per-job costs are tracked in `docs/cost-model.md` with the Pricing Calculator caveat.

## Consequences

- **Positive:** No idle compute cost. Job cost is proportional to actual work done.
  Architecture matches what the SAA exam and the AWS job market expect. EMR Serverless
  is a résumé-relevant service, not just a convenience.
- **Positive:** The OpenTofu scope simplifies: VPC and EC2 are replaced by an EMR
  Serverless Application resource. S3 and IAM remain.
- **Negative:** Interactive PySpark exploration no longer comes "for free" from an SSH
  session. Each exploratory run submits a job and incurs a billing event. This is
  manageable with discipline (write scripts, don't iterate in the shell).
- **Dependency:** EMR Serverless requires its own IAM execution role. This role cannot be
  created by `cerberus-cli` under ADR 003's no-IAM-management scope. See ADR 008.
- **Dependency:** Orchestration (submitting jobs to EMR Serverless on a schedule) is
  deferred per ADR 009; pipeline runs are manual via `aws emr-serverless start-job-run`
  or the AWS console until trigger conditions in ADR 009 are met.
