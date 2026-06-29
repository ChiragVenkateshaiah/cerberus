# ADR 018 — Data Quality as Code: PyDeequ Gates Between Medallion Layers

**Status:** Accepted — extends ADR 002 and ADR 017
**Date:** 2026-06-29

---

## Context

The medallion architecture (ADR 002) promotes data from bronze to silver to gold, each layer
asserting more trust than the last. Nothing in the pipeline yet enforces that trust. A silver
table built from bronze data with null keys, negative fares, or impossible timestamps will be
promoted to gold and queried in Athena as if it were correct. ADR 017 made the pipeline an
orchestrated state machine — which gives a natural place to put a gate: a state that must
pass before the next layer is built.

"Trust me, the transform is correct" is not data quality. Data quality is a checked, failing
assertion at the boundary between layers. The question is which tool, and what happens when a
check fails.

## Decision

Enforce data quality with **PyDeequ**, as gate states between medallion layers in the ADR 017
state machine.

### Where the gates sit

- **Bronze → Silver gate:** is the raw data complete and plausible enough to clean and promote?
- **Silver → Gold gate:** is the conformed data correct enough to aggregate and expose?

Each gate is a PyDeequ verification run. Representative checks for the NYC taxi dataset:
completeness on key columns; value bounds (`fare_amount >= 0`, `trip_distance >= 0`, pickup
before dropoff, timestamps within the dataset's period); membership (`payment_type` in the
known code set); uniqueness on the trip identifier where available; and row-count plausibility
against the prior run.

### Why PyDeequ, not the alternatives

- **PyDeequ** — chosen: it is a Spark-native library, so checks run inside the EMR Serverless
  job that already exists, with no second runtime to provision. It is an AWS-aligned library
  (built on Amazon's Deequ), its constraint/verification model maps directly to a pass/fail
  gate, and it can persist computed metrics to a repository for trend tracking.
- **Great Expectations** — richer suites and documentation, but a separate runtime and heavier
  setup. Rejected to avoid standing up a second validation system alongside Spark. Noted as
  the choice if a non-Spark validation surface (e.g. validating Athena query results directly)
  is later needed.
- **AWS Glue Data Quality** — managed and uses DQDL, but it is oriented around Glue jobs and
  crawlers, which are not this project's compute path (EMR Serverless is). Rejected to avoid
  introducing Glue ETL purely for its DQ feature.

### Enforcement semantics: halt-and-alert

A failed check fails the PyDeequ verification, which fails the Spark job, which fails the
Step Functions state — halting the pipeline before bad data is promoted, and routing to the
ADR 017 notify state. The pipeline stops loudly rather than promoting silently.

The alternative — quarantine the bad rows, continue with the good ones — is a more advanced
pattern that keeps the pipeline flowing while isolating bad data. It is deliberately deferred:
halt-and-alert is simpler, is the correct default for a pipeline still establishing its
baseline, and forces the data problem to be understood before it is automated around.
Quarantine-and-continue is the documented future evolution.

### Where results go

PyDeequ's computed metrics are persisted (a metrics repository in S3) so quality can be
tracked over time, and key outcomes surface to CloudWatch (ADR 007), so a failing gate is
visible in the same observability surface as job and query metrics.

## Consequences

- **Positive:** Trust between medallion layers becomes an enforced, checked boundary, not an
  assumption. Data quality as code is a strong data-engineering signal.
- **Positive:** Checks run inside the existing Spark job — no new runtime, no new always-on
  cost.
- **Negative:** PyDeequ bridges Python to a JVM library (py4j) and must be version-matched to
  the EMR Serverless Spark/Scala version, which can be finicky to pin. Mitigated by fixing the
  Deequ version against the chosen EMR release.
- **Negative:** Checks add runtime, and therefore a little cost, to each job. Kept
  proportionate by targeting key columns and invariants, not exhaustive profiling on every run.
- **Dependency:** The gates are states in the ADR 017 state machine; this ADR presumes that
  orchestration.

**Extends:** ADR 002, ADR 017
