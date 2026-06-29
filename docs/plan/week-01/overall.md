# Week 1 — Overall Plan

**Theme:** Cross the bootstrap boundary and get the first real infrastructure change deployed
to AWS through a policy-gated CI/CD pipeline. By end of Week 1, no human ever runs
`tofu apply` from a laptop again.

**Companion docs:** [`day-by-day.md`](day-by-day.md) ·
[`../../../roadmap.md`](../../../roadmap.md) ·
[`../../../platform-plan.md`](../../../platform-plan.md)

**ADRs this week delivers:** 013, 014, 015, 016, 019 (partial — foundation + cost guardrails)

---

## The dependency spine

```
Day 1 — repo skeleton + local gates
  ↓
Day 2 — manual bootstrap (OIDC + state bucket) ← the one ClickOps moment
  ↓
Day 3 — s3-bucket module + 00-foundation stack (plan locally)
  ↓
Day 4 — SSM exports + plan.yml workflow
  ↓
Day 5 — policy gates in CI (PR goes green)
  ↓
Day 6 — apply.yml + cost guardrails → foundation applied via CI
```

Every day produces something the next day builds on. Days cannot be reordered. If a day
slips, everything after it slips — useful signal that the day was under-scoped.

---

## Week exit criterion

- Manual bootstrap recorded in `docs/runbooks/bootstrap.md`. No static admin keys anywhere.
- A merge to `main` triggers GitHub Actions → OIDC assume → `tofu apply` on `dev/00-foundation`.
- Every PR runs `tofu fmt` + `tofu validate` + tflint + trivy + conftest/OPA + terraform-docs
  and **blocks** on failure.
- `cerberus-cli` (`aws --profile cerberus`) can list buckets and SSM params but gets
  `AccessDenied` on any infrastructure-mutating call — verified by a live denied call, not
  just policy inspection.
- All six S3 buckets, the Glue database, CloudWatch log groups, and SSM parameter exports
  exist in `ap-south-1`.
- AWS Budgets alert + Cost Anomaly Detection monitor live.

---

## The two hard problems this week

**1. The OIDC trust policy `sub` condition**

If the `sub` string is wrong, the assume-role silently denies with a generic credentials
error. This is the most failure-prone step of the week. Budget 30 extra minutes on Day 2 to
verify the trust with a throwaway test workflow run before building anything on top of it.

The exact `sub` values for this repo:
- `cerberus-ci-plan`: `repo:ChiragVenkateshaiah/cerberus:pull_request`
- `cerberus-ci-apply`: `repo:ChiragVenkateshaiah/cerberus:ref:refs/heads/main`

**2. `plan.yml` requires explicit `id-token: write` permission**

The `pull_request` event (not `pull_request_target` — ADR 015) does not expose secrets to
fork PRs, but the workflow token has limited repo permissions by default. OIDC still works
only if the workflow YAML explicitly includes:

```yaml
permissions:
  id-token: write
  contents: read
  pull-requests: write
```

Missing `id-token: write` silently produces an empty OIDC token and a confusing AWS
credentials error. This is the single most common OIDC + GitHub Actions failure mode.

---

## Deliverables checklist

| Day | Deliverable | Exit signal |
|-----|-------------|-------------|
| 1 | Repo skeleton + pre-commit config | `pre-commit run --all-files` green |
| 2 | Bootstrap runbook + OIDC roles + state bucket | Test workflow shows CI role ARN |
| 3 | `s3-bucket` module + `00-foundation` stack | `tofu plan` clean locally |
| 4 | SSM exports + `plan.yml` workflow | Plan comment appears on PR |
| 5 | OPA policies + trivy + terraform-docs in CI | PR fully green, all gates pass |
| 6 | `apply.yml` + Budgets + Anomaly Detection | Foundation applied via CI; live deny verified |

---

## Code review workflow

Include `/code-review ultra` at the end of each day's PR, before merging.

### Why

For a portfolio project where IAM policies, OIDC trust conditions, and IaC security posture
are the point, cold multi-agent review is not ceremony — it catches the mistakes that cost
credibility. A wrong `sub` condition or a policy Rego bug that `conftest` misses is exactly
what the cold review will catch because the agents haven't seen your reasoning.

### Cadence

| Day | Run `/code-review ultra`? | Reason |
|-----|--------------------------|--------|
| 1 | Optional | Low security surface — scaffolding only |
| 2 | **Yes** | Bootstrap runbook + OIDC trust policies are security-critical |
| 3 | Optional | Module structure review useful but not urgent |
| 4 | **Yes** | CI workflow with OIDC credentials |
| 5 | **Yes** | Policy gates are the guardrail — verify they actually gate |
| 6 | **Yes** | Apply workflow + IAM verification + cost controls |

### Workflow per day

```
Code → open PR → CI plan runs (automatic) → /code-review ultra → address findings → merge → CI apply
```

### Model guidance

- **`/code-review ultra`** — end-of-day PR review; cold multi-agent, best for IaC and IAM
  security surface
- **Opus inline** — architectural questions arising from review findings (IAM policy
  implications, OIDC trust reasoning)
- **Sonnet inline** — acting on findings and implementing fixes

### How to invoke

```
/code-review ultra
```

Run with the PR open. It reviews the current branch diff vs `main`. For a specific GitHub
PR: `/code-review ultra <PR#>`.
