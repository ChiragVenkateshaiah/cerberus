# ADR 009 — Defer Orchestration: Airflow and MWAA Both Deferred

**Status:** Superseded by ADR 017
**Date:** 2026-06-20

---

## Context

ADR 001 included Apache Airflow running on EC2 as the pipeline orchestrator. The compute
pivot in ADR 005 removed the EC2 instance, leaving Airflow without a host.

The natural successor question was: should orchestration move to **Amazon MWAA** (Managed
Workflows for Apache Airflow)?

## MWAA evaluation and rejection (for now)

MWAA was specifically evaluated. The billing model made it a poor fit:

MWAA charges an hourly rate for the **environment's existence**, regardless of whether any
DAG is running. There is no "stop the environment" operation analogous to stopping an EC2
instance. An MWAA Small environment billing continuously would be a higher fixed monthly
cost than the EC2 instance it would replace — which is precisely the opposite of the
structural improvement that made ADR 005's pivot worthwhile.

For a solo learner running a handful of pipeline jobs per week, the MWAA pricing model
is a worse fit than even a self-hosted Airflow on a stoppable EC2 box. MWAA is a good
choice when you need managed infrastructure without the operational burden; it is not a
good choice when the primary constraint is "no idle billing."

## Self-hosted Airflow evaluation

Self-hosted Airflow (whether on a new EC2 box, as a Docker container elsewhere, or in
some other form) was also evaluated. The concern:

The serverless compute redesign in ADR 005 was motivated partly by eliminating a
persistent compute instance. Immediately reintroducing a persistent compute instance to
host Airflow would partially undermine that rationale. Before adding Airflow back, the
pipeline should run manually enough times to validate that the pipeline logic is correct
and stable — and to confirm that the friction of manual submission is real, not hypothetical.

## Decision

Defer all orchestration. Pipeline steps run manually via `aws emr-serverless start-job-run`
(or the AWS console) until the trigger conditions below are met. No scheduler exists in the
current architecture.

### Trigger conditions for revisiting this ADR

The deferral ends when **any one** of the following is true:

1. **Manual run count:** The bronze → silver → gold pipeline has been run manually 10 or
   more times without issue, suggesting the pipeline logic is stable enough to automate.
2. **Second dataset added:** A second dataset is added to the project, creating a genuine
   multi-pipeline scheduling need that is cumbersome to manage manually.
3. **Session friction threshold:** Manual invocation happens more than 3 times in a single
   learning session, meaning the manual submission step has become the friction point
   rather than the pipeline logic itself.

When a trigger condition is met:
- Update `infra/PLANNING_CHECKPOINT.yml` (field `airflow-orchestration.status` from
  `deferred` to `under review`).
- Re-evaluate: a small nightly-stopped EC2 running self-hosted Airflow, or check whether
  MWAA's pricing model has changed since this ADR was written.
- Do NOT silently add orchestration without updating this file and PLANNING_CHECKPOINT.yml.

### This deferral is also machine-readable

The trigger conditions and current status are duplicated in `infra/PLANNING_CHECKPOINT.yml`.
Future planning sessions — whether conducted by a human or an AI planning assistant in
Claude Code plan mode — are required to read that file and check the status of
`airflow-orchestration` before proposing orchestration tooling.

## Consequences

- **Positive:** No persistent compute cost for orchestration. No Airflow webserver to
  secure or maintain.
- **Positive:** Manual pipeline submission forces explicit understanding of each step
  before automation. Automation before understanding is a risk for a learning project.
- **Negative:** No scheduled or triggered pipeline runs. Bronze ingestion does not happen
  automatically. This is acceptable for a project with a sole, active operator.
- **Negative:** If the project grows beyond the trigger thresholds and orchestration is
  added later, there is a rework cost: writing DAGs, testing scheduling behavior, managing
  Airflow dependencies. This is a deliberate trade-off, not an oversight.
- **MWAA is not ruled out permanently.** AWS has iterated on MWAA pricing and instance
  sizes since its launch. If the billing model changes materially, it may become viable.
  Check current pricing at the time of revisiting, not based on this ADR's evaluation date.
