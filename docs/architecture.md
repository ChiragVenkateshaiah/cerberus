# Cerberus — Architecture

Visual reference for the Cerberus serverless lakehouse, built as a platform-engineering
lab. Three views: the overall platform, the IaC layering, and the CI/CD + orchestration
flow. These diagrams are sourced into the project README.

Companion docs: [`../roadmap.md`](../roadmap.md), [`../platform-plan.md`](../platform-plan.md),
and the decision trail in [`adr/`](adr/).

---

## Diagram A — Overall Platform Architecture

How provisioning, the data plane, orchestration, and observability fit together.

```mermaid
flowchart TD
    src["NYC TLC Yellow Taxi<br/>(public Parquet)"]

    subgraph provision["Provisioning and delivery"]
        gh["GitHub Actions CI-CD"]
        oidc["IAM OIDC role<br/>(cerberus-ci, no static keys)"]
        tofu["OpenTofu<br/>(modules + envs/dev)"]
        gh --> oidc --> tofu
    end

    subgraph aws["AWS — Project=cerberus (ap-south-1)"]
        subgraph storage["S3 — Apache Iceberg"]
            bronze[("bronze")]
            silver[("silver")]
            gold[("gold")]
        end
        glue["Glue Data Catalog<br/>(cerberus_lakehouse)"]
        emr["EMR Serverless<br/>(Spark)"]
        athena["Athena<br/>(engine v3)"]
        sfn["Step Functions<br/>(medallion pipeline)"]
        eb["EventBridge<br/>(schedule)"]
        cw["CloudWatch<br/>(logs, metrics, dashboard)"]
        cost["Budgets + Cost Anomaly Detection"]
        cli["cerberus-cli<br/>(runtime identity)"]
    end

    tofu -->|provisions| aws
    src -->|ingest| emr
    eb -->|trigger| sfn
    sfn -->|orchestrates| emr
    emr -->|write Iceberg| bronze
    bronze -->|clean + DQ gate| silver
    silver -->|aggregate| gold
    gold -->|query| athena
    emr <-->|catalog| glue
    athena <-->|catalog| glue
    cli -->|submit jobs| emr
    cli -->|run queries| athena
    emr -->|logs + metrics| cw
    athena -->|bytes scanned| cw
    sfn -->|execution health| cw
    cw --> cost
```

---

## Diagram B — IaC Layering (ADR 012/013)

Reusable modules feed per-environment stacks. Cross-stack values flow through SSM
Parameter Store, never another stack's state file. One manual bootstrap step enables
everything else.

```mermaid
flowchart TD
    bootstrap["Manual bootstrap (ADR 011/014)<br/>OIDC provider + cerberus-ci role + state bucket<br/>— the one non-IaC step"]

    subgraph modules["infra/modules (reusable, tofu-tested)"]
        m1["s3-bucket"]
        m2["iam-role"]
        m3["emr-serverless-app"]
        m4["athena-workgroup"]
        m5["step-function-pipeline"]
    end

    subgraph dev["infra/envs/dev — apply order"]
        f["00-foundation<br/>(S3 + Glue + log groups + cost)"]
        i["10-iam<br/>(EMR exec role + cerberus-cli grant)"]
        c["20-compute<br/>(EMR Serverless + Athena)"]
        o["30-orchestration<br/>(Step Functions + EventBridge)"]
    end

    prod["infra/envs/prod<br/>(structure ready, applied later)"]

    bootstrap -->|enables tofu apply| f
    modules -->|consumed by| dev
    f -->|"/cerberus/dev/foundation/*-arn"| i
    i -->|"/cerberus/dev/iam/*-arn"| c
    c -->|"emr app id + role arn"| o
    dev -.->|same modules| prod
```

---

## Diagram C — CI/CD and Orchestration Flow (ADR 015/017)

Left: how a change reaches AWS — policy-gated, OIDC-applied. Right: how the medallion
pipeline runs on a schedule with data-quality gates between layers.

```mermaid
flowchart LR
    subgraph cicd["CI-CD pipeline (ADR 015/016)"]
        pr["Pull Request"]
        plan["tofu plan<br/>tflint · trivy · conftest(OPA)"]
        review["Review + approve"]
        merge["Merge to main"]
        apply["tofu apply (dev)<br/>via OIDC role"]
        pr --> plan --> review --> merge --> apply
    end

    subgraph orch["Scheduled orchestration (ADR 017/018)"]
        eb["EventBridge<br/>(schedule)"]
        b["Bronze ingest"]
        dq1["PyDeequ gate"]
        s["Silver build"]
        dq2["PyDeequ gate"]
        g["Gold aggregate"]
        eb --> b --> dq1 --> s --> dq2 --> g
    end

    apply -.->|deploys state machine| eb
```
