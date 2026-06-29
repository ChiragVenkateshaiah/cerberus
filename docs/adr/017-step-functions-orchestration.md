# ADR 017 — Step Functions Orchestration: Closing the ADR 009 Deferral

**Status:** Accepted — supersedes ADR 009; extends ADR 005
**Date:** 2026-06-29

---

## Context

ADR 009 deferred all orchestration. It rejected MWAA because MWAA bills for the
environment's existence regardless of use — idle cost, the exact property the serverless
pivot in ADR 005 set out to eliminate. It also declined to reintroduce self-hosted Airflow,
because that means a persistent compute instance, reversing ADR 005's central gain. ADR 009
left the door open with explicit trigger conditions and one instruction: do not add
orchestration without revisiting that ADR and the planning checkpoint.

This ADR revisits it deliberately. The project is being built out as a platform-engineering
lab, and a scheduled, observable, retrying pipeline is a core part of that. The orchestrator
ADR 009 was implicitly waiting for — one with no idle cost — exists: AWS Step Functions.

## Decision

Orchestrate the medallion pipeline with **AWS Step Functions (Standard workflow)**, triggered
by **EventBridge Scheduler**. ADR 009's deferral is closed; its status becomes "Superseded
by ADR 017".

### Shape

A Standard-type state machine runs the pipeline as a sequence of EMR Serverless job-run
tasks, with the PyDeequ data-quality gates of ADR 018 as states that fail the execution when
a check fails:

```
Bronze ingest → DQ gate → Silver build → DQ gate → Gold aggregate
```

Each task uses Step Functions' native EMR Serverless integration with `.sync` semantics, so a
state waits for its job to finish and fails if the job fails. `Retry` handles transient
errors; `Catch` routes any failure to a notify state (CloudWatch / SNS, ADR 007/019).
EventBridge Scheduler triggers the machine on a cadence (e.g. daily). Manual
`start-job-run` for ad-hoc single-step runs remains available.

### Why Step Functions, not the alternatives

- **MWAA** — rejected again for the same reason as ADR 009: it bills for an always-on
  environment. Nothing about that has changed.
- **Self-hosted Airflow** — rejected again: it requires persistent compute, reversing ADR 005.
- **A single Lambda or a cron'd script chaining jobs** — workable for a linear chain, but it
  has no built-in retry/catch, no durable execution state, and no execution history or visual
  graph. It pushes orchestration logic into imperative code.
- **Step Functions (Standard)** — chosen: pay-per-state-transition with no idle cost (the
  ADR 009 constraint satisfied), native service integrations with EMR Serverless that remove
  glue code, built-in retry/catch and durable state, and a visual execution graph that is both
  an operational aid and a portfolio artefact.

### Why Standard, not Express

Express workflows are optimised for high-volume, short-duration event processing and bill by
request and duration with at-least-once semantics. This pipeline is low-volume and
long-running (EMR jobs run minutes). Standard's per-transition pricing is cheaper here, its
runs can last well beyond Express's five-minute ceiling, and its exactly-once execution model
fits a data pipeline. Standard is the correct fit.

### What this does NOT decide

- The schedule cadence and whether ingestion is incremental or full-refresh — pipeline-design
  decisions made when the jobs are written.
- Backfill orchestration. The first machine runs forward only; historical backfill is a later
  addition if needed.
- ASL authoring style (hand-written vs generated). The state machine is defined in the
  `step-function-pipeline` module (ADR 013); how its ASL is produced is an implementation
  detail.

## Consequences

- **Positive:** The pipeline becomes scheduled, observable, and self-retrying with no idle
  cost. Event-driven serverless orchestration is a strong, current data-engineering signal,
  and the choice is defensible with explicit reasoning rather than defaulted to Airflow.
- **Negative:** Amazon States Language has a learning curve and is AWS-specific — the
  orchestration is not portable the way an Airflow DAG is. This lock-in is accepted as
  consistent with the deliberate AWS-native bet of ADR 005/006.
- **Negative:** Local testing of a state machine is harder than running a script. Mitigated by
  keeping each task a self-contained job that can also be invoked manually.
- **Cleanup:** The `airflow/dags/` directory is now vestigial and is removed or clearly
  marked superseded.
- **Tracking:** `infra/PLANNING_CHECKPOINT.yml` is updated — `airflow-orchestration` moves
  from `deferred` to `superseded`, pointing at this ADR.

**Supersedes:** ADR 009 · **Extends:** ADR 005
