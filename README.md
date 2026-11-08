# Task 02 — Service Blueprint, SLOs and Cost Envelope

**Intern:** Suraj Dhakad
**Intern ID:** RAB-2026-7AFBF77314
**Programme:** Cloud Computing & DevOps Internship, RabTech Academy
**Task:** Task 02 — Build a Service Blueprint, SLOs, and a 3-Scenario Cost Envelope

## 1. What this document set is

This is the operational contract for a small production web service, designed
*before* any infrastructure is provisioned or any deployment automation is
written. It answers four questions a team must settle up front:

1. How does the service actually work, end to end? → `service-blueprint.md`
2. How good does it have to be, and how do we know when it is not? → `slo-workbook.csv`
3. What will it cost at three realistic traffic levels? → `cost-model.csv`
4. Which foundational decisions are we locking in, and why? → `architecture-decisions.md`

## 2. The reference service

All four artefacts describe the **same** example service so the numbers stay
internally consistent:

| Property | Value |
|---|---|
| Service | **ShopLite API** — a small two-tier e-commerce backend (product catalogue, cart, checkout) |
| Tier 1 (web/app) | Stateless Node.js REST API behind an Application Load Balancer, running on ECS Fargate |
| Tier 2 (data) | Amazon RDS for PostgreSQL (primary + synchronous standby), ElastiCache Redis for sessions/cart cache |
| Region | AWS **ap-south-1 (Mumbai)** — see ADR-001 |
| Traffic profile | Read-heavy (~85% GET), average response payload ≈ 8 KB, peak-to-average ratio ≈ 3× |
| Availability target | 99.9% monthly (see SLO workbook) |

Request volume scenarios used throughout: **10,000 / 100,000 / 1,000,000
requests per month** (labelled S1 "Launch", S2 "Growth", S3 "Scale").

## 3. File guide

| File | Contents |
|---|---|
| `service-blueprint.md` | Request, dependency, deployment, data and failure paths for the two-tier service, with Mermaid diagrams and per-path notes (timeouts, retries, circuit breakers). |
| `slo-workbook.csv` | SLI definitions, SLO targets, monthly error budgets, burn-rate alert thresholds and paging rules for availability, latency, freshness and durability. |
| `cost-model.csv` | Monthly cost estimate at the three traffic scenarios, broken down by compute, storage, data transfer, logging/monitoring, backup and support, with per-scenario totals. |
| `architecture-decisions.md` | ADR-001 to ADR-006: region, recovery objectives (RTO/RPO), secrets management, least-privilege IAM, deployment strategy, and observability stack — each with context, decision and consequences. |

## 4. Pricing assumptions (cost model)

- Prices are **AWS ap-south-1 (Mumbai) on-demand list prices**, rounded,
  excluding taxes (GST) and excluding any Savings Plans / Reserved Instances —
  i.e. a deliberately conservative envelope.
- Currency: **USD**, with INR equivalents at **₹84 = $1** shown in the totals
  row notes of the CSV.
- A month is **730 hours**; storage is priced per GB-month.
- Fargate: $0.04048/vCPU-hour and $0.004445/GB-hour (Linux x86, ap-south-1).
- RDS PostgreSQL Multi-AZ db.t4g.medium ≈ $0.144/hr per instance (primary +
  standby billed separately); storage gp3 $0.115/GB-month.
- Data transfer out to internet: $0.109/GB (first 10 TB tier); inter-AZ
  traffic between ALB, Fargate and RDS is included in the transfer line as an
  allowance.
- CloudWatch Logs ingestion $0.50/GB; metrics/alarms at list price.
- AWS Business Support is modelled as 10% of usage (minimum $100/month) and
  only applied from Scenario 2 upward; Scenario 1 uses Developer Support
  ($29/month flat). This is stated per-line in the CSV so a reviewer can
  re-price it.
- Free-tier effects are ignored (envelope, not a bill prediction).

## 5. Consistency notes

- The SLO error budgets in `slo-workbook.csv` are computed from the same
  99.9% / 99.95% targets referenced by the failure-path analysis in
  `service-blueprint.md`.
- The recovery objectives in ADR-002 (RTO 60 min, RPO 5 min) are what the
  backup line in `cost-model.csv` pays for (RDS automated backups + cross-AZ
  snapshots retained 7 days, plus weekly snapshot export to S3 in the same
  region).
- Scenario sizing: Fargate task counts, RDS instance class and log volumes
  grow per scenario in the cost model; the assumptions column states the
  sizing behind each quantity so the arithmetic can be audited.
