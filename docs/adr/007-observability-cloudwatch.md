# ADR 007 — Observability: CloudWatch (For Now) Replaces Prometheus + Grafana

**Status:** Accepted — supersedes ADR 001 (observability portion)
**Date:** 2026-06-20

---

## Context

ADR 001 included Prometheus + Grafana + node-exporter + cAdvisor + a statsd-exporter for
Airflow metrics, all running as Docker containers on the EC2 instance. Combined, these
services accounted for roughly 1.2 GB of RAM in the ADR 004 budget — not the biggest line
item, but non-trivial, and entirely contingent on the EC2 instance existing.

With the compute pivot in ADR 005 removing the EC2 instance and replacing it with EMR
Serverless, the observability stack lost its natural home. There is no always-on VM to
run Prometheus on and no Docker network to scrape from.

## Decision

Use **Amazon CloudWatch** for observability, rather than a self-hosted Prometheus + Grafana
stack.

This is a softer decision than ADR 005 or ADR 006 and is held with less permanence. The
reasoning:

### Why CloudWatch is sufficient for now

- **EMR Serverless emits metrics natively to CloudWatch.** Application-level and job-level
  metrics (vCPU used, memory used, job duration, job state transitions) appear without any
  agent or configuration. This is the only compute engine in the current architecture.
- **Athena emits query metrics to CloudWatch.** Data scanned, query duration, and success/
  failure states are available out of the box.
- **Glue Data Catalog has its own metrics surface.** Crawl durations and table counts flow
  into CloudWatch without agent setup.
- **Free tier:** At this project's scale (low query volume, one learner), CloudWatch costs
  will likely remain within the free tier for metrics and logs. No additional budget impact.
- **No infrastructure to run.** With no persistent EC2 instance, there is no obvious place
  to host a Prometheus scraper that would scrape... nothing running persistently.

### Why this is stated softly

Prometheus + Grafana provides a genuinely better observability experience when you want
custom dashboards, cross-service correlation, or fine-grained SLO tracking. CloudWatch's
query language (CloudWatch Metrics Insights) is useful but less ergonomic than PromQL for
complex queries. Grafana's visualization capabilities are richer than CloudWatch dashboards.

If a future learning goal specifically targets observability tooling — building dashboards,
writing alert rules in PromQL, understanding Prometheus' data model — that goal is better
served by a self-hosted Prometheus stack than CloudWatch. That goal does not exist today.

The bar for revisiting this decision is: "I want to learn Prometheus/Grafana specifically."
It is not: "CloudWatch dashboards are less pretty." Aesthetics are not a reason to add
infrastructure and cost.

### What CloudWatch is asked to do here

- Monitor EMR Serverless job outcomes (success/failure, duration, resource use)
- Monitor Athena query costs (data scanned per query, as a budget signal)
- Alert on anomalous spend or repeated job failures (CloudWatch Alarms with a monthly
  budget notification via AWS Budgets)
- Store EMR Serverless and Athena logs for debugging

No application-level custom metrics are planned for now. If pipeline code needs
structured logging, it goes to CloudWatch Logs via the EMR Serverless log configuration.

## Consequences

- **Positive:** Zero infrastructure overhead. Metrics exist from day one without any agent
  setup.
- **Positive:** No additional cost at this scale.
- **Positive:** One fewer argument for needing a persistent compute instance.
- **Negative:** CloudWatch Alarms + Metrics Insights is a less learnable, less
  portfolio-visible observability stack than Prometheus + Grafana. If an interviewer asks
  "how did you instrument your pipeline?", CloudWatch is an honest answer but a less
  interesting one.
- **Soft commitment:** This decision is revisited if (a) a specific learning goal requires
  Prometheus/Grafana, or (b) a new compute surface (e.g., a lightweight EC2 box) is added
  that gives the self-hosted stack a place to run. Until then, CloudWatch is good enough.
