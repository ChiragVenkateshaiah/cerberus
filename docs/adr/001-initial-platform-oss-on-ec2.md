# ADR 001 — Initial Platform: Full OSS Stack on EC2 via Docker Compose

**Status:** Superseded by ADR 005 (compute engine), ADR 006 (query engine), ADR 007 (observability)
**Date:** 2026-06-20

---

## Context

Cerberus is a solo, resume-grade open-source data lakehouse project. The primary goals are:
- Deep, hands-on learning of SQL, PySpark, and AWS Solutions Architecture
- Producing a public GitHub portfolio that reads as genuine engineering work, not a tutorial clone
- Staying within a self-imposed ~$15/month budget against a fixed AWS credit allotment

When laying out the initial architecture, the instinct was to run everything open-source.
Open-source gives full visibility into every layer — the scheduler, the query engine, the
metrics pipeline — which aligns directly with the learning objective. It also mirrors the
patterns in real-world data engineering job descriptions.

The infrastructure options considered were:
1. **Multiple small EC2 instances, one per service** — cleanest isolation, most operationally
   realistic, but hardest to manage solo and most expensive at rest.
2. **Single large EC2 instance running all services via Docker Compose** — one SSH target,
   one Compose file for the full topology, Docker's internal bridge network handles
   service-to-service routing, and instance can be stopped between sessions to pause billing.
3. **Managed AWS services from the start** — deferred; the learning goal was to build and
   operate the stack, not just configure managed services.

Option 2 was chosen.

## Decision

Deploy a **single EC2 t3.xlarge (4 vCPU, 16 GB RAM)** running the following services via
**Docker Compose**, organized into profiles so subsets can be brought up independently:

| Profile | Services |
|---------|----------|
| `core` | Spark (standalone, local[*]) |
| `orchestration` | Airflow (LocalExecutor) + shared Postgres |
| `query` | Trino (single-node coordinator), dbt-trino, Superset |
| `observability` | Prometheus, Grafana, node-exporter, cAdvisor, statsd-exporter |

**Storage:** S3 + Apache Iceberg (separate ADR 002).
**Catalog:** AWS Glue Data Catalog — no extra container, native Spark/Trino integration.
**IAM:** Dedicated `cerberus-cli` IAM user (separate ADR 003); EC2 instance gets an
instance role scoped to S3 bucket access + Glue.
**Networking:** Single public subnet, no NAT gateway (cost). App UIs accessed exclusively
via SSH port-forward — no public ingress ports beyond 22.

### Why this felt right at the time

- **Docker Compose's internal networking** means every service finds every other by
  hostname with zero extra config — Airflow talks to the Spark container at `spark:7077`,
  Trino at `trino:8080`, Prometheus scrapes at `node-exporter:9100`. That simplicity is real.
- **One SSH target** keeps operations simple for a solo developer: one box to start,
  stop, SSH into, and `docker compose logs` against.
- **Stop the instance between sessions** — EC2 pauses billing on a stopped instance
  (except EBS), so a t3.xlarge at ~$0.1856/hr felt manageable if kept to a few hours a day.
- **Profile isolation** was meant to address the RAM question: bring up only `core` for
  PySpark work, only `orchestration` when building DAGs. In theory, 16 GB could stretch.

## Consequences

**Positive (at the time):**
- Simple, auditable, single-box topology.
- Every service visible and tuneable.
- No per-query billing surprises.

**Negative (discovered during planning — see ADR 004):**
- A realistic RAM budget with all profiles running simultaneously reached 14–18 GB,
  outside t3.xlarge's comfortable headroom.
- The "use profiles to slice it" mitigation relies on never needing the full stack at once —
  but an orchestrated pipeline (Airflow triggering Spark, Trino querying, Superset
  rendering) needs all four profiles up simultaneously.
- The honest fix — t3.2xlarge (32 GB) — roughly doubles the EC2 hourly cost, which
  undermines the budget justification that made a single-box approach attractive.

This ADR is preserved in full as a record of what was actually decided and why it seemed
correct, before evidence from planning surfaced the cost/RAM constraint.

**Superseded by:** ADR 005 (replaces the compute/services approach), ADR 006 (replaces
the query engine), ADR 007 (replaces observability). ADR 002 and ADR 003 survive unchanged.
