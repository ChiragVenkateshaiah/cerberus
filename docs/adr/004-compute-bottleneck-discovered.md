# ADR 004 — Compute Bottleneck Discovered: t3.xlarge RAM Insufficient for Full Stack

**Status:** Closed (evidence record — triggered ADR 005's redesign)
**Date:** 2026-06-20

---

## Context

ADR 001 proposed running the full OSS platform on a single EC2 t3.xlarge (4 vCPU, 16 GB
RAM) via Docker Compose profiles. The profiles approach was meant to allow running subsets
of the stack within the 16 GB envelope. During detailed planning of the Docker Compose
service definitions, a per-service RAM budget was calculated to validate whether this held.

## The RAM budget

| Component | Approx RAM |
|---|---|
| OS + Docker engine | ~1.5 GB |
| Spark (driver + local executor) | 4–6 GB |
| Shared Postgres (Airflow + Superset metadata) | ~0.5 GB |
| Airflow (webserver + scheduler, LocalExecutor) | 2–3 GB |
| Trino (single-node coordinator, JVM heap) | 3–4 GB |
| Superset (web process) | ~1.5 GB |
| Prometheus + Grafana + exporters | ~1.2 GB |
| **Total** | **~14–18 GB** |

t3.xlarge provides 16 GB. The low end of this range (14 GB) leaves 2 GB headroom — tight
but conceivably workable with careful JVM tuning. The high end (18 GB) requires swap, which
degrades Spark and JVM-based services materially.

## The profile-isolation argument fails under a real pipeline run

The ADR 001 mitigation was: use Compose profiles so you're never running all services
simultaneously. In practice, an end-to-end pipeline run (Airflow triggering Spark,
results queryable via Trino, visible in Superset) requires all four profiles active at once:

- `core` (Spark) — executing the job
- `orchestration` (Airflow + Postgres) — triggering and monitoring it
- `query` (Trino + Superset) — serving the results
- `observability` (Prometheus + Grafana) — showing what happened

The profile approach works if the learning goal is operating one service at a time. It
does not work if the goal is an integrated pipeline run — which is the point of the project.

## The available fix and its cost

The direct fix for the RAM problem is to use **t3.2xlarge (8 vCPU, 32 GB RAM)**. This
comfortably fits the entire stack with headroom.

The problem: t3.2xlarge costs roughly twice the hourly rate of t3.xlarge. Even when
stopped between sessions, the EC2 cost that justified the single-box approach was predicated
on t3.xlarge rates. Doubling that cost to maintain an always-on-ish Docker stack erodes
the budget case entirely — particularly when the instance is stopped *between* sessions but
all services need to be running *during* them.

This is not a hypothetical concern or a worst-case estimate. It is a real number derived
from planning the actual service set that ADR 001 committed to.

## What this triggered

This evidence — not a vague concern about complexity — is what prompted the redesign in
ADR 005. The question shifted from "how do we fit OSS services on one box" to "do we need
the box at all if the goal is learning, not operating a cluster."

**This ADR exists as an evidence record.** It is not a decision ADR. The decision is in ADR 005.
