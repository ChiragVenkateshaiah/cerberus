# Cerberus — Cost Model

Last updated: 2026-06-20

---

## AWS credit balance

| Item | Value |
|---|---|
| Total credit | $117.98 |
| Expiry | 2026-09-02 |
| Self-imposed monthly ceiling | ~$15/month |
| Months of runway at ceiling | ~7.8 months |

---

## Historical estimate: original OSS-on-EC2 plan (ADR 001, now superseded)

This estimate is preserved for contrast against the serverless model that replaced it.

| Resource | Rate | Usage assumption | Monthly estimate |
|---|---|---|---|
| EC2 t3.xlarge | ~$0.1856/hr | 4 hrs/day × 30 days | ~$22.27 |
| EBS gp3 root (30 GB) | ~$0.096/GB-month | at rest always | ~$2.88 |
| S3 storage (bronze/silver/gold) | ~$0.025/GB-month | ~50 GB | ~$1.25 |
| S3 requests (PUT/GET) | negligible at learning scale | — | ~$0.50 |
| Data transfer out | ~$0.09/GB | minimal | ~$0.50 |
| **Total (original plan)** | | | **~$27/month** |

The original plan exceeded the $15/month ceiling at t3.xlarge rates, and fixing the RAM
problem (ADR 004) with a t3.2xlarge would have pushed it to ~$50/month. This is the
direct cost case that motivated the serverless pivot in ADR 005.

---

## Current estimate: serverless-first plan (ADR 005/006/007)

### Pricing caveat

The figures below use approximate rates and directional reasoning. **Verify all
`ap-south-1` rates against the AWS Pricing Calculator before using these numbers to
make decisions.** EMR Serverless and Athena rates in Mumbai may differ from the
US-East-1 published rates commonly cited in blog posts.

### S3 storage (unchanged)

| Resource | Rate | Usage assumption | Monthly estimate |
|---|---|---|---|
| S3 storage (4 buckets) | ~$0.025/GB-month | ~50 GB across tiers | ~$1.25 |
| S3 requests | ~$0.005/1000 PUT; ~$0.0004/1000 GET | low at learning scale | ~$0.50 |
| S3 lifecycle transitions | $0 (same storage class) | — | $0 |

### EMR Serverless compute

EMR Serverless bills per second of vCPU and memory used while a job is running,
with a minimum of 1 minute per worker.

**Directional reasoning (not a confirmed number):**
- A typical bronze ingestion job (moderate dataset, 2–4 workers, 5–15 minutes) will
  cost on the order of cents per run at current EMR Serverless rates.
- At 3–5 pipeline runs per week, monthly EMR Serverless cost will be in the low
  single-digit dollar range — well under the portion of the $15 ceiling that S3 doesn't
  consume.
- The structural property is: $0 when not running. At a few hours of actual job execution
  per month, EMR Serverless is fundamentally cheaper than a continuously-stoppable EC2
  instance, because "stopped between sessions" still means EBS billing and startup latency.

**Verify at:** AWS Pricing Calculator → EMR Serverless → ap-south-1 → enter vCPU-hours
and memory-GB-hours for a representative job.

### Amazon Athena

Athena charges per TB of data scanned (verify `ap-south-1` current rate; commonly
~$5/TB scanned in US regions, may differ in Mumbai).

**Directional reasoning:**
- With properly partitioned Iceberg tables and Parquet column projection, exploratory
  queries against a ~50 GB dataset scan far less than the full dataset.
- At learning-scale query volumes (tens of queries per session, few sessions per week),
  monthly Athena cost will be in the low single-digit dollar range or below.
- Query cost is directly controlled by table design: partition pruning and column
  projection reduce scanned bytes, which reduces cost.

### AWS Glue Data Catalog

- First 1 million objects stored: **$0**
- First 1 million requests per month: **$0**
- At this project's scale: **$0/month**

### CloudWatch

- Metrics (basic monitoring from EMR Serverless, Athena, S3): within **free tier** at
  this scale.
- Logs (EMR Serverless job logs, Athena query logs): small volume; likely within free
  tier or low single-digit cents.
- CloudWatch Alarms: first 10 alarms free.
- Estimated: **$0–$1/month**

### OpenTofu remote state (ADR 006)

| Resource | Rate | Monthly estimate |
|---|---|---|
| S3 state bucket (versioned, tiny) | ~$0.025/GB-month | < $0.10 |
| DynamoDB lock table (on-demand, near-zero traffic) | ~$1.25/million writes | < $0.01 |

### Summary: serverless-first monthly estimate

| Resource | Monthly estimate |
|---|---|
| S3 storage + requests | ~$1.75 |
| EMR Serverless (jobs) | ~$2–5 (verify) |
| Athena (queries) | ~$1–3 (verify) |
| Glue Data Catalog | $0 |
| CloudWatch | ~$0–1 |
| State backend (S3 + DynamoDB) | ~$0.10 |
| **Total** | **~$5–10/month (verify against Pricing Calculator)** |

---

## Credit runway comparison

| Model | Monthly estimate | Months of runway |
|---|---|---|
| Original OSS-on-EC2 (ADR 001, superseded) | ~$27/month | ~4.4 months |
| Serverless-first (current) | ~$5–10/month | ~12–24 months (verify) |

The serverless model significantly extends the available runway against the $117.98 credit.
The specific endpoint date is not stated here because it depends on actual usage intensity
and verified per-unit rates — use the AWS Cost Explorer and Budgets alert (configured in
CloudWatch/Budgets as part of observability setup) to track actual spend.

---

## Budget alerts

An AWS Budget alert should be configured at:
- **$10/month** — informational: approaching the ceiling
- **$15/month** — warning: at ceiling, review what's running

Alert via SNS → email. This is part of the observability setup and is not a CloudWatch
cost (Budgets alerts are free).
