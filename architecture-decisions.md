# Architecture Decision Records — ShopLite API

**Intern:** Suraj Dhakad · **Intern ID:** RAB-2026-7AFBF77314 · RabTech Academy, Task 02
Each record follows: Status · Context · Decision · Consequences.

---

## ADR-001: Host the service in AWS ap-south-1 (Mumbai) as the single production region

- **Status:** Accepted
- **Context:** ShopLite's customers and operations team are in India. Data
  latency to Indian users, INR-denominated billing through an Indian entity,
  and keeping personal data inside India (DPDP Act 2023 expectations) all
  point to an in-country region. The team is small (one on-call rotation), so
  operating two active regions would double the operational surface.
- **Decision:** Run all production in **ap-south-1 (Mumbai)** across two AZs.
  Disaster recovery for a full regional outage is *restore-based* into
  ap-south-2 (Hyderabad) from S3 snapshot exports (see ADR-002), not active
  multi-region.
- **Consequences:**
  - (+) Lowest latency for the actual user base; data residency story is
    simple to explain to customers and auditors.
  - (+) One region keeps the cost envelope in `cost-model.csv` and the
    on-call burden realistic for a small team.
  - (−) A full ap-south-1 outage takes the service down until the manual DR
    restore completes (failure path F8). This risk is explicitly accepted at
    current scale and is revisited if revenue per hour of downtime exceeds
    the cost of active multi-region (review at Scenario 3 in the cost model).

## ADR-002: Recovery objectives — RTO 60 min / RPO 5 min for AZ & instance failure; RTO 4 h / RPO 24 h for regional loss

- **Status:** Accepted
- **Context:** The availability SLO is 99.9% (≈ 43 min of allowed downtime a
  month), so recovery from the *common* failures (AZ loss, DB instance
  failure) must be automatic and fast, or a single incident consumes the
  whole month's error budget. A full regional loss is rare and can tolerate a
  slower, manual, restore-based recovery.
- **Decision:**
  - **AZ / instance failure:** RTO **≤ 60 min** (expected: 60–120 s via RDS
    Multi-AZ automatic failover — the 60 min is the contractual ceiling
    including detection and verification), RPO **≈ 0** (synchronous standby).
  - **Data corruption / accidental deletion:** point-in-time recovery using
    RDS automated backups (5-minute WAL archiving, 7-day retention) →
    RPO **≤ 5 min**, RTO ≤ 60 min.
  - **Regional loss:** restore the weekly S3 snapshot export into ap-south-2
    → RTO **4 h**, RPO **24 h** (worst case, if the failure lands just before
    the weekly export; accepted in ADR-001).
  - Backups are only considered real if the **quarterly restore drill**
    passes (also an SLI in the SLO workbook).
- **Consequences:**
  - (+) The backup line in the cost model (7-day PITR + S3 snapshot archive)
    is directly justified by these numbers — no gold-plating, no gaps.
  - (+) Failover expectations are testable (game-days twice a year measure
    the ≤ 120 s failover).
  - (−) Regional DR is manual and slow by design; the runbook must be
    maintained and the S3 snapshot export must be monitored, or RPO 24 h
    silently becomes "never".

## ADR-003: All secrets live in AWS Secrets Manager; no secrets in code, images, or environment files

- **Status:** Accepted
- **Context:** The API needs DB credentials, a payment-gateway API key, and a
  JWT signing key. Storing them in the repo, the container image, or plain
  ECS environment variables has caused real-world breaches at far larger
  companies, and rotation would require a redeploy.
- **Decision:** Every secret is stored in **AWS Secrets Manager**, referenced
  by ARN in the ECS task definition, and injected at task start. RDS
  credentials use **automatic 30-day rotation** via the Secrets Manager RDS
  rotation integration. CI/CD holds **no long-lived AWS keys**: GitHub
  Actions assumes an IAM role via OIDC. Developers never see production
  secret values; staging has its own separate secrets.
- **Consequences:**
  - (+) Rotation is routine, not an event; a leaked secret has a 30-day
    maximum lifetime and can be force-rotated in minutes.
  - (+) Audit trail: every secret read is a CloudTrail event attributable to
    a task role.
  - (−) Small per-secret monthly cost and a hard dependency at task boot
    (mitigated: running tasks are unaffected by a Secrets Manager outage —
    see dependency path in the blueprint).
  - (−) Local development needs a documented `.env.example` + staging-secret
    workflow so developers don't invent their own shortcuts.

## ADR-004: Least-privilege IAM — one role per component, resource-scoped policies, break-glass for humans

- **Status:** Accepted
- **Context:** A single broad "app role" with `*:*` on the account would turn
  any API vulnerability (e.g. SSRF to the metadata endpoint) into a full
  account compromise. Human console access with standing admin rights has
  the same problem.
- **Decision:**
  - **ECS task role (API):** only `s3:GetObject` on the product-images bucket
    prefix, `secretsmanager:GetSecretValue` on this service's secret ARNs,
    `sqs:SendMessage` on the email queue, and CloudWatch Logs write on its
    own log group. **No RDS IAM permissions beyond network reachability**
    (DB auth is by password from Secrets Manager), no IAM, no EC2, no
    wildcard resources.
  - **Worker task role:** SQS receive/delete on the email queue + SES send on
    the verified domain only.
  - **CI/CD role (GitHub OIDC):** push to this service's ECR repo, register
    ECS task definitions, update this service only, `iam:PassRole` restricted
    to the two task roles above.
  - **Terraform role:** the only role allowed to create/modify
    infrastructure, assumed from the ops pipeline — not from laptops.
  - **Humans:** SSO login, read-only by default; production changes go
    through the pipeline. A **break-glass admin role** exists, requires MFA +
    a ticket reference, and alarms on every assumption.
  - IMDSv2 is enforced on any EC2-backed resource and Fargate tasks use
    task-role credentials only.
- **Consequences:**
  - (+) Blast radius of any single compromise is one component, not the
    account; auditors can read the policies and verify intent quickly.
  - (+) Policy-as-code: all policies live in Terraform next to the resources
    they protect, so reviews catch privilege creep in pull requests.
  - (−) More up-front design work and occasional deploy friction when a new
    feature legitimately needs a new permission — accepted as the price of
    the boundary; the fix is a small reviewed policy change, never a wildcard.

## ADR-005: Rolling deployments with automatic rollback; expand–migrate–contract for schema changes

- **Status:** Accepted
- **Context:** Deploys are the most common self-inflicted outage source. The
  service must be able to deploy during business hours without downtime and
  retreat within minutes when a release is bad (failure path F6).
- **Decision:** ECS rolling update (min healthy 100%, max 200%) of immutable
  images tagged by git SHA; ALB health gate of 3 minutes; automatic rollback
  on health-check failure or the `HighErrorRate` alarm. Database migrations
  run before the app rollout and follow expand–migrate–contract so the
  previous app version always works against the new schema.
- **Consequences:**
  - (+) Deploys are boring and reversible; rollback is a revision switch,
    not a restore.
  - (+) Schema changes can never strand the service between app and DB
    versions.
  - (−) Migrations take two or three releases to complete (add column →
    backfill/dual-write → drop column), which the team must plan for.
  - (−) Requires the discipline of backward-compatible changes — enforced in
    code review, not by tooling.

## ADR-006: Observability on CloudWatch + structured logs, with burn-rate alerting tied to the SLO workbook

- **Status:** Accepted
- **Context:** At this scale a self-hosted Prometheus/Grafana stack is
  another service to run. The SLOs need multi-window burn-rate alerts, which
  need metric queries, retention, and an on-call paging path — all available
  natively.
- **Decision:** Structured JSON logs to CloudWatch Logs (30-day retention);
  application metrics emitted as CloudWatch EMF; SLO burn-rate alarms
  (14.4×/1 h and 3×/6 h, per `slo-workbook.csv`) page via SNS → PagerDuty;
  ALB/RDS/ECS managed metrics and one dashboard per audience (on-call,
  business). Revisit a dedicated observability platform only if log volume
  at Scenario 3 makes CloudWatch ingestion the largest cost line.
- **Consequences:**
  - (+) Zero extra infrastructure to operate; alerting semantics match the
    SLO workbook exactly, so pages always correspond to user impact.
  - (+) Logging/monitoring cost is explicit in the cost model and scales
    visibly per scenario.
  - (−) CloudWatch Logs query ergonomics are weaker than dedicated tools;
    accepted at current volume, with log sampling for debug level in
    Scenario 3 to control ingestion cost.
