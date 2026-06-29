# ADR 019 — Cost Governance: Layered Guardrails Around a Finite Credit Budget

**Status:** Accepted — extends ADR 010 and ADR 007
**Date:** 2026-06-29

---

## Context

ADR 010 mandated a single AWS Budgets alert as the mitigation for long-lived admin keys, and
`docs/cost-model.md` records the hard constraint behind it: a finite credit balance
(~$117.98, expiring 2026-09-02) and a self-imposed ~$15/month ceiling. A single
after-the-fact budget alert is a start, not a governance posture. The Cloud-Lab scope adds
services that can each run away — EMR Serverless jobs, Athena scans, Step Functions
executions — and the cost case is sharpened by the credit expiry: spend that is wasted before
the credit lapses is spend that simply disappears.

This ADR makes cost governance a coherent, layered control set rather than one lonely alarm.

## Decision

Adopt a layered cost-governance posture combining **detective** controls (which observe spend)
and **preventive** controls (which cap it before it happens).

### Detective controls

- **AWS Budgets** — a monthly cost budget at the ceiling, with alert thresholds at 50%, 80%,
  and 100% of budget and a forecasted-overspend alert. This subsumes and formalises the
  single alert ADR 010 mandated.
- **AWS Cost Anomaly Detection** — a service-level monitor that flags unusual spend patterns
  (e.g. an EMR or Athena spike) using AWS's own modelling. Free, and catches what a fixed
  threshold misses.
- **Cost-allocation tags** — `Project` and `Environment` activated as cost-allocation tags so
  spend is attributable per project and per environment. This also sharpens the ADR 003
  boundary: Cerberus spend is cleanly separable from NovaPay's in the same account.

### Preventive controls

- **Athena per-query data-scanned cap** — a control limit on the workgroup (ADR 006) that
  aborts any single query scanning beyond a set volume. This is the real-time guard against
  a fat-fingered full-table scan, which is the most likely way to spend a lot of money fast
  on Athena.
- **EMR Serverless maximum capacity + job timeout** — a maximum aggregate vCPU/memory limit
  on the application and a per-job timeout, so a runaway or hung Spark job cannot consume
  unbounded compute.

### Why both, and the honest limit of each

Detective and preventive controls cover different failure modes, and neither alone is enough.
Budgets and Cost Anomaly Detection are inherently lagging — they read billing data that
updates only every several hours, so by the time a Budgets alert fires, the spend has already
happened. The preventive caps (Athena bytes, EMR max capacity) act in real time but are blunt:
they cannot tell a legitimate large query from a mistake. The layered set is the answer —
real-time caps stop the catastrophic case immediately, while Budgets and anomaly detection
catch the slow drift the caps are set too high to notice.

### The tuning tension

A cap set too low blocks legitimate work — a query that genuinely needs to scan a lot of data,
or a job that genuinely needs more workers. A cap set too high protects nothing. The caps are
set generously relative to the expected solo-learner workload but far below "disaster", and
they are per-environment (dev tighter than a future prod). The specific values live in the
`infra/` code as tunable parameters, not hardcoded here, and are revised as real usage data
arrives.

### What this does NOT decide

- Reserved capacity or savings plans — irrelevant at this scale and on credit.
- Automated cost-driven teardown (e.g. auto-destroying dev on a schedule). Noted as a
  possible future control, not adopted now.

## Consequences

- **Positive:** Spend is bounded by real-time caps and watched by lagging detectors — the two
  failure modes (sudden catastrophic, slow drift) are both covered. Demonstrable cost
  discipline is a genuine architecture signal, not a footnote.
- **Positive:** Per-project/per-environment cost attribution via tags makes the ADR 003
  account boundary measurable, not just conventional.
- **Negative:** Preventive caps can interrupt legitimate large operations; this is the accepted
  cost of protection and is tuned with real usage.
- **Negative:** Detective controls lag billing data by hours and cannot prevent the spend they
  report — stated plainly so the caps are understood as the real-time line of defence.
- **Supersession of scope:** The lone Budgets alert of ADR 010 is now one control within this
  set; ADR 010's mitigation requirement is satisfied and extended here.

**Extends:** ADR 010, ADR 007
