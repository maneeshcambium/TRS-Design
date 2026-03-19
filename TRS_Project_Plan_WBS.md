# TRS — Execution-Ready Project Plan & Work Breakdown Structure (WBS)

**Author:** Senior Project Manager / Systems Architect  
**Date:** March 13, 2026  
**Deadline:** May 20, 2026 (start Mar 18 · 9 sprints)  
**Team:** 1 Architect · 3 Full-Stack Devs · 1 PM  
**Methodology:** Iterative / Sprint-based with hard go-live gate  

---

## Table of Contents

1. [CDK Architecture Recommendation](#1-cdk-architecture-recommendation)
2. [Capacity Reality Check](#2-capacity-reality-check)
3. [MVP Scope Definition — What's IN vs. DEFERRED](#3-mvp-scope-definition)
4. [Risk Register](#4-risk-register)
5. [Phase Overview Table](#5-phase-overview-table)
6. [Resource Allocation Matrix](#6-resource-allocation-matrix)
7. [Sprint Plan (9 Sprints + Go-Live · Mar 18 – May 20)](#7-sprint-plan)
8. [Detailed WBS — All Phases](#8-detailed-wbs)
   - Phase 0: Project Setup & Architecture
   - Phase 1: CDK Infrastructure Foundation
   - Phase 2: Database Schemas & Migrations
   - Phase 3: Score Ingestion Pipeline
   - Phase 4: CDC Pipeline (PeerDB + Sign-Pair Transformer)
   - Phase 5: RTS Replication Pipeline
   - Phase 6: Valkey Distributed Cache
   - Phase 7: TRS Reporting API (Fargate)
   - Phase 8: Admin API (Fargate)
   - Phase 9: TRS Frontend SPA
   - Phase 10: Admin Frontend SPA
   - Phase 11: Observability, Monitoring & Alerting
   - Phase 12: Testing & QA
   - Phase 13: Production Readiness & Go-Live
9. [Critical Path Diagram](#9-critical-path-diagram)
10. [Post-MVP Backlog](#10-post-mvp-backlog)

---

## 1. CDK Architecture Recommendation

### Recommendation: Single Monorepo · Multiple CDK Stacks · Per-Concern Stacks

```
trs-platform/                       ← Git monorepo root
├── cdk/                            ← All infrastructure as code
│   ├── bin/
│   │   └── trs.ts                  ← CDK App entry — instantiates all stacks
│   ├── lib/
│   │   ├── stacks/
│   │   │   ├── 01-network-stack.ts       ← VPC import (existing), custom SGs
│   │   │   ├── 02-security-stack.ts      ← IAM roles, KMS keys, Secrets Manager paths
│   │   │   ├── 03-data-stack.ts          ← Aurora, ClickHouse EC2, Valkey, RDS MSSQL
│   │   │   ├── 04-messaging-stack.ts     ← SQS queues, SNS topics, EventBridge rules
│   │   │   ├── 05-ecs-stack.ts           ← ECS Fargate cluster, ECR repos
│   │   │   ├── 06-api-stack.ts           ← API Gateway, TRS API Fargate, Admin API Fargate, ALBs
│   │   │   ├── 07-cdn-stack.ts           ← CloudFront (TRS SPA + Admin SPA), S3 buckets
│   │   │   ├── 08-cicd-stack.ts          ← CodePipeline / GitHub Actions integration
│   │   │   ├── 09-cdc-stack.ts           ← PeerDB Fargate, Sign-Pair Transformer Fargate
│   │   │   ├── 10-rts-stack.ts           ← RTS MSSQL replica, RtsSyncWorker Fargate
│   │   │   └── 11-monitoring-stack.ts    ← CloudWatch dashboards, alarms, SNS alerts
│   │   └── constructs/
│   │       ├── fargate-api-service.ts    ← Reusable: ALB + Fargate service + autoscaling
│   │       ├── fargate-worker.ts         ← Reusable: scheduled Fargate task
│   │       └── secure-bucket.ts          ← Reusable: S3 + OAC + encryption
│   ├── cdk.json
│   └── package.json
├── src/
│   ├── trs-api/                    ← .NET 8 TRS Reporting API (Fargate)
│   ├── admin-api/                  ← .NET 8 Admin API (Fargate)
│   ├── score-processor/            ← .NET 8 Score Processor (Fargate)
│   ├── cdc-transformer/            ← .NET 8 Sign-Pair CDC Transformer (Fargate)
│   ├── rts-sync/                   ← .NET 8 RTS Sync worker (Fargate)
│   ├── aggregate-refresh/          ← .NET 8 Aggregate Refresh job (Fargate)
│   └── frontend/
│       ├── trs-app/                ← React TRS SPA
│       └── admin-app/              ← React Admin SPA
└── infra/
    ├── sql/
    │   ├── postgres/               ← Liquibase changelogs (changelog.xml + changesets/*.sql — local & cloud)
    │   └── clickhouse/             ← Versioned ClickHouse DDL scripts (001_bronze.sql, 002_silver.sql, …)
    └── scripts/                    ← Operational: PeerDB initial load, CH rebuild, etc.
```

### Pros vs. Cons

| Criterion | Monorepo + Multiple CDK Stacks ✅ RECOMMENDED | Separate Repos per concern |
|---|---|---|
| **Cross-stack dependencies** | Easy — CloudFormation exports / SSM lookups within same app | Requires shared parameter store conventions across repos |
| **Team size fit** | Excellent — 3 devs, no ownership silos needed | Overhead of coordinating 5+ repos for a 4-person team |
| **Shared constructs** | Single `constructs/` directory — reuse `FargateApiService` across TRS + Admin | Must publish a private npm package or copy-paste |
| **CI/CD** | One pipeline deploys everything; environment promotion (dev → staging → prod) trivial | N pipelines to maintain |
| **Deployment risk** | A broken stack can affect the deploy run — mitigated by stack ordering (numbered prefixes) | Truly isolated deploys |
| **Scale signal** | Right for now; split into sub-repos only if team exceeds 10+ with separate ownership | Jump-to for large orgs |
| **CDK synth time** | Slightly longer as stacks grow (acceptable at this scale) | Faster per-repo synth |

**Stack ordering rule:** Numbered prefix (`01-` through `11-`) ensures `cdk deploy --all` deploys in dependency order. Cross-stack references use SSM Parameter Store (avoids hard CloudFormation export coupling).

---

## 2. Capacity Reality Check

| Resource | Available Person-Days (10 wks) |
|---|---|
| Architect | 50 days |
| Dev 1 (Backend focus) | 50 days |
| Dev 2 (Backend/Data focus) | 50 days |
| Dev 3 (Frontend/Full-stack focus) | 50 days |
| **Total** | **200 person-days** |

| | Days Estimated | Notes |
|---|---|---|
| **Total WBS effort** | ~197 person-days | See phase breakdown below |
| **Overhead (meetings, reviews, PR cycles)** | ~30 days (~15%) | Built into estimates |
| **Buffer / risk float** | ~3 days | ⚠️ Near zero — see Risk #1 |

> **⚠️ CRITICAL WARNING:** This timeline has essentially no slack. Any external integration delay 
> (ReportsHub, CRS) or scope creep will require immediate de-scoping to protect the May 20 date.
> The MVP scope definition below is the safety valve.

---

## 3. MVP Scope Definition

### IN SCOPE — Must ship by May 20, 2026

| Domain | Included |
|---|---|
| Infrastructure | All CDK stacks, networking, data layer, compute |
| Score Ingestion | Score Processor Fargate, validation, UPSERT to Postgres, rejection handling |
| CDC Pipeline | PeerDB + Sign-Pair Transformer → ClickHouse Bronze/Silver/Gold |
| RTS Sync | Hourly delta sync MSSQL → Postgres relationship tables |
| Caching | Valkey integration (aggregates, test configs, embargo service) |
| TRS Reporting API | Roster, School, District aggregate endpoints; three-tier fallback; embargo gate |
| Admin API | Test config CRUD, embargo management, FlightPlan import |
| TRS Frontend | Class/roster view, school report, district report, SSO login |
| Admin Frontend | Test configuration UI, embargo management UI |
| SSO | OIDC JWT integration for both sites |
| Observability | CloudWatch dashboards, critical alarms, structured logging |
| Testing | Unit tests per component; integration tests for critical paths |
| External: FlightPlan | Admin imports test list — IN (needed for test config to work) |
| External: ITS | PLD import — IN (needed to configure performance levels) |
| External: CSR | Standards association — IN (needed for standards reports) |

### DEFERRED — Post-MVP Phase 2

| Domain | Deferred Reason |
|---|---|
| **ReportsHub (ISR PDF)** | ~~Deferred~~ — **Promoted to MVP.** Full client required. Contract must be locked by Apr 7 (see Risk R2). |
| **CRS (SDF Excel)** | ~~Deferred~~ — **Promoted to MVP.** Full client required. Contract must be locked by Apr 7 (see Risk R2). |
| **State-level aggregates** | Nightly pre-computation; low priority for Indiana MVP |
| **Component/Standards fallback (district scope)** | HTTP 503 path acceptable for MVP |
| **Playwright E2E tests** | Full suite post-MVP; critical paths done in MVP |
| **Load testing at Indiana scale** | Staging validation only; full load test post-MVP |
| **Student demographics / student_attributes table** | Marked "Future" in design doc |
| **ClickHouse Silver (roster reads)** | Postgres is primary path; Silver is future feature |

---

## 4. Risk Register

| # | Risk | Likelihood | Impact | Mitigation Strategy |
|---|---|---|---|---|
| **R1** | **10-week timeline with zero slack** — Any delay in one phase cascades to all downstream phases | HIGH | CRITICAL | (1) Enforce MVP scope ruthlessly — no new features after Apr 1. (2) Weekly PM risk review. (3) Pre-identify which features get cut if timeline slips: ReportsHub/CRS are first candidates for de-scope back to stubs if contracts slip past Apr 14. |
| **R2** | **External API contracts not finalized** (ReportsHub, CRS) — Dev cannot code against a moving spec; both integrations are now **full MVP requirements** | HIGH | **CRITICAL** | (1) PM escalates immediately — hard contract lock deadline **Apr 7** for both. Missing this date directly threatens May 20 go-live. (2) If contract is not locked by Apr 7: de-scope back to stub and move full integration to post-MVP Phase 2. (3) Until spec is locked, Dev 1 builds behind a typed `IReportsHubClient` / `ICrsClient` interface so the internals can be swapped without touching the endpoints. |
| **R3** | **RTS MSSQL Replication Complexity** — SQL Server transactional replication requires DBA access to master RTS, schema mapping can be far more complex than estimated | MEDIUM | HIGH | (1) Spike task in Week 1 (P0.10) to validate access and confirm delta tracking column. (2) If replication access is blocked, fall back to **read replica via JDBC over VPN** polled hourly. (3) RTS team SLA for read access by Mar 20. |
| **R4** | **PeerDB CDC + ClickHouse Gold correctness** — Sign-pair collapse logic is subtle; incorrect -1/+1 pairs cause silent aggregate corruption | MEDIUM | HIGH | (1) Dedicated validation sprint (P4.10-P4.11): INSERT/UPDATE/DELETE scenarios validated before any API work begins. (2) Gold query results cross-checked against Postgres ground truth in integration tests. (3) CloudWatch alarm on `TRS/CDC/SchemaError`. |
| **R5** | **Greenfield frontend integration latency** — With no frontend and no API existing simultaneously, UI devs are blocked until API stubs are available | MEDIUM | MEDIUM | (1) API stubs / OpenAPI spec generated from .NET Minimal API in Week 3 (before full implementation). (2) Dev 3 uses MSW (Mock Service Worker) to mock all API responses locally. (3) Architect publishes API contract document by end of Week 3. |

---

## 5. Phase Overview Table

| Phase | Name | MVP? | Duration | Weeks | T-Shirt Size | Primary Owner |
|---|---|---|---|---|---|---|
| 0 | Project Setup & Architecture Kickoff | ✅ | 7 days | 1 | S | Architect + Dev 2 + PM |
| 1 | CDK Infrastructure Foundation | ✅ | 12 days | 1–3 | **L** | Architect |
| 2 | Database Schemas & Migrations | ✅ | 10 days | **1–3** _(scripts local W1; cloud deploy W2)_ | M | Dev 2 + Dev 1 |
| 3 | **Opp / Score Ingestion Pipeline** (S3 → SQS → Score Processor → Postgres + `DevDataSeeder` full implementation) | ✅ | 13 days | 3–4 | M | Dev 1 + Dev 2 |
| 4 | CDC Pipeline (PeerDB + Sign-Pair Transformer → ClickHouse Bronze/Silver/Gold) | ✅ | 14 days | 3–5 | **L** | Dev 1 + Architect |
| 5 | **RTS Replication Pipeline** (MSSQL replica → Fargate hourly delta sync → Postgres membership tables) | ✅ | 13 days | 3–5 | M | Dev 2 |
| 6 | Valkey Distributed Cache (shared library + key conventions) | ✅ | 2 days | 4 | S | Dev 2 |
| 7 | TRS Reporting API (Fargate — roster/school/district/standards; three-tier fallback; embargo) | ✅ | 25 days | 4–7 | **XL** | Dev 1 |
| 8 | Admin API (Fargate — test config CRUD; embargo management; external integrations) | ✅ | 16 days | 4–7 | **L** | Dev 2 |
| 9 | TRS Frontend SPA (React — roster/school/district/standards reports + SSO) | ✅ | 25 days | 5–8 | **L** | Dev 3 |
| 10 | Admin Frontend SPA (React — test config, standards, PLDs, embargo + SSO) | ✅ | 12 days | 5–8 | M | Dev 3 |
| 11 | **Nightly Aggregate Refresh Job** (Fargate — ClickHouse Gold → `report_cache`; school 15min, district 5min, state nightly) | ✅ | 3 days | 6–7 | S | Dev 1 |
| 12 | **ReportsHub Integration — ISR PDF** (full client: serialize score payload → call ReportsHub → stream PDF URL back; circuit breaker via Polly) | ✅ | 4 days | 6–7 | M | Dev 1 |
| 13 | **CRS Integration — SDF Excel Download** (full client: pass `studentKey` + `testKey` → call CRS → stream Excel file back; circuit breaker via Polly) | ✅ | 4 days | 6–7 | M | Dev 1 |
| 14 | **CSR Integration — Publications & Standards Browser** (Admin API client: browse publications, fetch standards under a publication; used in Admin UI to associate standards to a test alias) | ✅ | 2 days | 6–7 | S | Dev 2 |
| 15 | Observability & Monitoring | ✅ | 9 days | 7–9 | M | Dev 1 + Architect |
| 15 | Testing & QA | ✅ | 19 days | 8–10 | **L** | All |
| 16 | Production Readiness & Go-Live | ✅ | 11 days | 9–10 | M | Architect + PM |

> **Legend:** ✅ = In MVP scope (target May 20)

---

## 6. Resource Allocation Matrix

| Phase | Architect | Dev 1 (Backend) | Dev 2 (Backend/Data) | Dev 3 (Frontend) |
|---|---|---|---|---|
| P0: Setup | Lead | Support | **P0.11-12** | — |
| P1: CDK Infra | **Primary** | Support | —  | Support (P1.18-20) |
| P2: DB Schemas | Support | P2.9-P2.13 (CH) | **Primary** (PG) | — |
| P3: Score Processor | CDK tasks | **Primary** | — | — |
| P4: CDC Pipeline | Lead PeerDB | **Primary** (Transformer) | — | — |
| P5: RTS Sync | Design support | — | **Primary** | — |
| P6: Valkey | Design | Support | **Primary** | — |
| P7: TRS API | Review/CDK | **Primary** | — | — |
| P8: Admin API | Review/CDK | — | **Primary** | — |
| P9: TRS Frontend | API contract | — | — | **Primary** |
| P10: Admin Frontend | — | — | — | **Primary** |
| P11: Observability | Lead | Support | Support | — |
| P12: Testing | Security review | Backend tests | Backend tests | E2E/Frontend |
| P13: Go-Live | **Lead** | Support | Support | Support |

---

## 7. Sprint Plan (9 Sprints + Go-Live · Mar 18 – May 20)

> Parallel workstreams are shown per role for each sprint. All four team members are active every sprint from S1 onward.

---

### S1 — Mar 18–24: Kickoff & Infrastructure Bootstrap

| Role | This Sprint |
|---|---|
| **Architect** | P0.1–P0.10: monorepo init, CDK bootstrap, AWS account setup, naming conventions, CI/CD skeleton, Secrets Manager paths, ADRs, SSO + RTS alignment spike; begin P1.1 (VPC import & subnet discovery) |
| **Dev 1** | P0.5 CI/CD support; P0.9 SSO alignment; **P2.9–P2.11**: ClickHouse DDL scripting locally — Bronze (`VersionedCollapsingMergeTree`), Silver (`AggregatingMergeTree` + MV), Gold layer + Bronze-to-Gold MVs; all scripts versioned under `infra/sql/clickhouse/` and validated against local Docker ClickHouse |
| **Dev 2** | **P0.11–P0.12**: local dev scaffold — `docker-compose.local.yml`, Postgres + ClickHouse init script directories, WireMock stub skeleton, `DevDataSeeder` .NET project scaffold; shared AWS dev resource docs + Valkey `KeyPrefix` convention; **P2.1–P2.6**: Liquibase project setup + Postgres DDL changesets — core schema, score tables, membership tables, config tables, operational tables; all validated against local Docker Postgres |
| **Dev 3** | Monorepo orientation + local env; P1.19–P1.20 CloudFront SPA distribution design |

**Exit criteria:** Monorepo committed; CDK bootstrapped; existing VPC imported into CDK (`cdk.context.json` VPC/subnet parameters locked); local dev scaffold in repo; all team members running the local stack; **Liquibase Postgres changesets (P2.1–P2.6) apply cleanly against local Docker Postgres**; **ClickHouse Bronze/Silver/Gold DDL scripts (P2.9–P2.11) execute cleanly against local Docker ClickHouse**

---

### S2 — Mar 25–31: CDK Infrastructure + Database Schemas

| Role | This Sprint |
|---|---|
| **Architect** | P1.1–P1.18: full CDK stack suite — VPC import + custom SGs, ACM/R53, IAM/KMS, Aurora PG, ClickHouse EC2, Valkey ElastiCache, RDS MSSQL, ECS/ECR, ALBs, SQS/SNS/EventBridge, API Gateway, S3 event notifications; P1.22–P1.24 CloudWatch log groups, X-Ray, log schema |
| **Dev 1** | **P2.12–P2.13**: complete ClickHouse membership mirrors + seed validation script; **apply all CH DDL scripts to dev EC2** (once P1.9 provisioned); begin P4.1 PeerDB Fargate spike |
| **Dev 2** | **P2.7–P2.8**: complete Liquibase changesets — materialized views + unique indexes + pg_cron jobs; **run `liquibase update` against dev Aurora** (once P1.7 provisioned); begin P3.1–P3.2 Score Processor scaffold |
| **Dev 3** | P1.19–P1.21: CloudFront distributions for TRS SPA + Admin SPA (S3 OAC origins, SPA error page routing); CI/CD SPA deploy stage (React build → S3 sync → cache invalidation) |

**Exit criteria:** All CDK stacks synthesize and deploy to dev; **Liquibase `liquibase update` applies all Postgres changesets to Aurora dev cleanly**; **all ClickHouse DDL scripts applied to dev EC2**; CI/CD pipeline green end-to-end

---

### S3 — Apr 1–7: Score Processor + CDC Bootstrap + RTS Access

| Role | This Sprint |
|---|---|
| **Architect** | P4.1–P4.3: PeerDB Fargate setup, CDC Job #1 (`student_opportunities` + `student_component_scores` → SQS `cdc-events`), CDC Job #2 (membership → CH mirrors direct); P3.10 CDK Fargate task definition for Score Processor |
| **Dev 1** | P3.1–P3.7: Score Processor — BackgroundService scaffold, score JSON parse + validate, data normalization (`TestKey → testalias_id`), `is_aggregate_eligible` computation, idempotency check against `score_ingest_log`, batch UPSERT, rejection handling (3 rejection types → S3 rejected bucket + SQS ACK) |
| **Dev 2** | P5.1–P5.3: confirm RTS MSSQL access + delta-tracking column; configure SQL Server transactional replication → TRS MSSQL RDS; validate replication lag; begin P3.12 — DevDataSeeder config/roster/embargo seed steps + fake-opp field generation logic (matching Score Processor rules) |
| **Dev 3** | P9.1–P9.2: TRS SPA React scaffold (Vite + TypeScript), routing structure, SSO auth flow with OIDC redirect + MSW mock token for local dev |

**Exit criteria:** Score files S3 → SQS → Postgres end-to-end; PeerDB + Sign-Pair Transformer scaffolded; RTS replication confirmed; DevDataSeeder seeds reference data + full roster hierarchy

---

### S4 — Apr 8–14: CDC Validated + RTS Worker + DevDataSeeder Complete

| Role | This Sprint |
|---|---|
| **Architect** | P4.10–P4.14: CDC E2E validation — INSERT/UPDATE/DELETE all correct in Gold layer; Sign-Pair Transformer Fargate CDK finalized; P5.11 RtsSyncWorker CDK Fargate + EventBridge hourly rule; publish TRS API OpenAPI contract (P7.1 scaffold + spec) |
| **Dev 1** | P3.8–P3.11: Score Processor CloudWatch metrics, unit tests, E2E smoke test; P4.4–P4.9 + P4.13: Sign-Pair Transformer — SQS consumer, INSERT/UPDATE/DELETE Bronze emission, error handling + exponential backoff + DLQ, unit tests |
| **Dev 2** | P5.4–P5.10: RtsSyncWorker — BackgroundService scaffold, delta query logic, RTS→Postgres schema transform, UPSERT relationship tables, SSM `last_sync_date`, failure alarm, integration tests; P3.12 complete — DevDataSeeder ClickHouse Bronze direct-write + all CLI flags (`--reset`, `--skip-opps`, `--skip-postgres`) |
| **Dev 3** | P9.3–P9.5: TRS SPA — class/roster list view (students + teacher assignments table), student score drill-down stub; all screens wired to MSW mocks |

**Exit criteria:** CDC E2E correct for all 3 event types (no double-counting); RTS sync running hourly in dev; `DevDataSeeder` fully operational (seeds Postgres + ClickHouse Bronze → Gold auto-populated via MVs); API OpenAPI spec published for Dev 3 consumption

---

### S5 — Apr 15–21: Valkey Cache + TRS API Core + Admin API Core

| Role | This Sprint |
|---|---|
| **Architect** | P6.2 Valkey key-prefix convention doc (`Valkey__KeyPrefix`) + cache key naming spec; Admin API scaffold review (P8.1); JWT RS256 design review; security threat model pass on auth surfaces |
| **Dev 1** | P6.1 + P6.3 + P6.5: `TRS.Caching` shared library (`IValkeyCacheService`, `StackExchange.Redis`), EmbargoService Valkey integration (60s/5min TTLs), Valkey wired as Tier-0 in fallback chain; P7.2–P7.5: JWT RS256 middleware, embargo gate middleware, test config loader, three-tier fallback infrastructure |
| **Dev 2** | P6.4 + P6.6–P6.7: test config Valkey TTL (10min), cache invalidation on Admin change (SNS → key eviction), Valkey unit tests (hit/miss/TTL/eviction/failover); P8.1–P8.5: Admin API scaffold, JWT middleware, test alias CRUD endpoints |
| **Dev 3** | P9.6–P9.8: TRS SPA — school aggregate report view, district aggregate report view (both wired to MSW-mocked API); P10.1 Admin SPA scaffold + SSO auth flow |

**Exit criteria:** Valkey Tier-0 integrated and tested; Roster/School/District aggregate endpoints live with three-tier fallback; Admin test alias CRUD working; `EmbargoService` cache-backed

---

### S6 — Apr 22–28: External Integrations + Frontend Feature-Complete

| Role | This Sprint |
|---|---|
| **Architect** | OWASP Top 10 security review pass (auth, injection, SSRF, structured logging); P11 Nightly Aggregate Refresh CDK (Fargate + EventBridge schedules for school 15min, district 30min, state nightly); final API contract review |
| **Dev 1** | P7.6–end: remaining TRS API endpoints — standards fallback, component scores, `report_cache` write-through; P11: Nightly Aggregate Refresh Fargate implementation; P12: ReportsHub ISR PDF client (Polly circuit breaker + response streaming); P13: CRS SDF Excel download client |
| **Dev 2** | P8.6–end: Admin API — embargo management endpoints, FlightPlan test import, ITS PLD + cut-score import; P14: CSR publications browser + standards association client; P8 unit + integration tests |
| **Dev 3** | P9.9–P9.13: TRS SPA — standards report view, ISR/SDF download buttons wired to real API, real OIDC SSO token flow; P10.2–P10.5: Admin SPA — test configuration management UI, PLD management, standards association browser |

**Exit criteria:** All external service clients wired (FlightPlan, ITS, CSR, ReportsHub, CRS); TRS SPA all views complete; Admin SPA test config + embargo UI complete

---

### S7 — Apr 29–May 5: Integration Testing + Observability

| Role | This Sprint |
|---|---|
| **Architect** | P15: CloudWatch dashboards (TRS API latency, CDC lag, RTS sync health, Score Processor throughput), SLO alarms, SNS alert routing; CDK monitoring stack deploy; integration test results triage |
| **Dev 1** | TRS API integration tests (full endpoint suite + all fallback-chain paths covered); structured JSON logging + X-Ray tracing wired in TRS API + Score Processor + Sign-Pair Transformer; regression fixes |
| **Dev 2** | Admin API integration tests (config CRUD + embargo + all external integrations); structured logging in Admin API + RtsSyncWorker; CloudWatch metric alarms for all worker services; regression fixes |
| **Dev 3** | Playwright E2E scaffold — critical path scripts (login → roster → school report → Admin config CRUD → embargo toggle); Admin SPA embargo UI complete; frontend unit tests for all components |

**Exit criteria:** All integration test suites green; dashboards + alarms deployed; both SPAs feature-complete with unit tests passing

---

### S8 — May 6–12: Staging Deployment + E2E Testing

| Role | This Sprint |
|---|---|
| **Architect** | CDK deploy all stacks to staging; PeerDB initial data load to staging; performance baseline (API p95 latency, ClickHouse Gold query times); final security penetration review |
| **Dev 1** | TRS API staging smoke tests; Score Processor staging end-to-end (drop real file → S3 → SQS → Postgres verified); staging regression fixes; draft Score Processor + CDC pipeline runbook |
| **Dev 2** | Admin API staging smoke tests; RtsSyncWorker full initial load validation on staging Postgres; staging regression fixes; draft RTS sync + Admin API runbook |
| **Dev 3** | Playwright E2E — all critical paths green on staging environment; cross-browser checks (Chrome, Edge, Safari); accessibility audit; fix staging-specific SPA issues |

**Exit criteria:** Staging fully deployed and stable; all E2E critical paths green; PeerDB initial load validated; all runbooks drafted

---

### S9 — May 13–19: Production Hardening & Go-Live Prep

| Role | This Sprint |
|---|---|
| **Architect** | Production CDK deploy (all stacks, final review gate); PeerDB production initial data load; on-call runbook finalized; rollback procedure documented and rehearsed |
| **Dev 1** | Production TRS API + Score Processor smoke tests; Score Processor + CDC runbook finalized; on-call rotation setup |
| **Dev 2** | Production Admin API + RTS sync smoke tests; RTS full initial load to production Postgres membership tables; Admin API + RTS runbook finalized |
| **Dev 3** | Production SPA deploy + CloudFront cache warm-up; final UX + accessibility pass; SPA error monitoring configured |

**Exit criteria:** Production stack fully deployed and verified; runbooks complete; on-call schedule confirmed; team ready for go-live

---

### May 20 — 🚀 Go-Live

| Role | Activity |
|---|---|
| **Architect** | War-room lead; DNS cutover; PeerDB production sync enabled; go/no-go decision authority |
| **Dev 1** | TRS API + Score Processor production validation; monitor CDC lag metrics |
| **Dev 2** | Admin API + RTS sync production validation; monitor RTS sync health |
| **Dev 3** | SPA production smoke + CDN monitoring; stakeholder demo support |

**Go-Live complete when:** All smoke tests pass; first real score file processed end-to-end through production; stakeholder sign-off received.

---

## 8. Detailed WBS

---

### Phase 0: Project Setup & Architecture Kickoff
**Duration:** Week 1 | **T-shirt:** S | **Effort:** ~10 days

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P0.1 | Initialize Git monorepo structure (`cdk/`, `src/`, `infra/sql/`, `scripts/`) with README and conventions doc | 0.5 | Architect | — |
| P0.2 | CDK TypeScript project bootstrap — `cdk init`, `cdk bootstrap` for dev/staging/prod environments | 1 | Architect | P0.1 |
| P0.3 | AWS account/environment setup: confirm 3 environments (dev/staging/prod), IAM admin access, cost budget alerts | 1 | Architect + PM | — |
| P0.4 | Define CDK stack naming conventions, environment config pattern (`cdk.context.json`), resource tagging strategy | 0.5 | Architect | P0.2 |
| P0.5 | CI/CD pipeline skeleton — GitHub Actions workflows: `on push → cdk synth`, `on merge → cdk deploy dev`, manual gate for staging/prod | 1.5 | Architect + Dev 1 | P0.2 |
| P0.6 | Secrets Manager: define all secret paths (DB passwords, API keys, SSO JWKS URL, external system credentials) | 0.5 | Architect | P0.3 |
| P0.7 | Produce Architecture Decision Records (ADRs): Fargate API choice, Valkey placement, CDK monorepo, RTS sync strategy | 0.5 | Architect | — |
| P0.8 | External integration kickoff emails: ReportsHub, CRS, CSR, FlightPlan, ITS — request API specs, sandbox URLs, POCs | 1 | PM | — |
| P0.9 | SSO alignment: confirm OIDC discovery URL, `tenant_id` claim name, `roles` claim array format, JWKS rotation policy with SSO team | 0.5 | Dev 1 + PM | — |
| P0.10 | RTS spike: confirm read access to Master RTS MSSQL; identify delta-tracking column; validate SQL Server replication feasibility | 1 | Architect + PM | — |
| P0.11 | Local dev environment scaffold: `docker-compose.local.yml` (Postgres + ClickHouse + WireMock), `local.env.example`, Postgres + ClickHouse SQL init script directories (`infra/sql/postgres/init/`, `infra/sql/clickhouse/`), WireMock stub skeleton (`dev/wiremock/mappings/`), `DevDataSeeder` .NET project scaffold at `src/tools/DevDataSeeder/` | 1.5 | Architect + Dev 2 | P0.1 |
| P0.12 | Local dev shared AWS resources: one-time setup docs for shared `trs-scores-dev-raw` S3 bucket + `score-ingest-dev` SQS queue in TRS dev account (per-developer `AWS__ScoresKeyPrefix`); document shared Valkey key-prefix convention (`Valkey__KeyPrefix=<machine-name>`) in team wiki / README | 0.5 | Architect | P0.3 |

---

### Phase 1: CDK Infrastructure Foundation
**Duration:** Weeks 1–3 | **T-shirt:** L | **Effort:** ~22 days

#### 1A: Network & Security

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P1.1 | CDK: Import existing VPC using `ec2.Vpc.fromLookup` — confirm and lock subnet IDs (public/private/isolated), AZ distribution, NAT Gateway presence, and route table associations; store VPC parameters in `cdk.context.json` for deterministic synth | 0.5 | Architect | P0.4 |
| P1.2 | CDK: Security Groups — audit existing default SGs in the VPC; create custom SGs for each service tier: Aurora (port 5432 from Fargate SGs only), ClickHouse EC2 (port 8123/9000 from Fargate SGs only), Valkey/ElastiCache (port 6379 from Fargate SGs only), Fargate task SGs (egress to data tier), ALB SGs (HTTPS 443 inbound from API Gateway VPC link). No reliance on default SGs for production traffic. | 1.5 | Architect | P1.1 |
| P1.3 | CDK: ACM certificates for `*.trs.example.com` and `*.admin-trs.example.com` (DNS validation) | 0.5 | Architect | P1.1 |
| P1.4 | CDK: Route 53 hosted zones + A/ALIAS records for API and CDN domains | 0.5 | Architect | P1.3 |
| P1.5 | CDK: IAM roles for ECS task execution + task roles (scoped per service: TRS API, Admin API, Score Processor, RTS sync, etc.) | 1 | Architect | P1.2 |
| P1.6 | CDK: KMS customer-managed keys — Aurora encryption, S3 SSE, Valkey at-rest, Secrets Manager | 0.5 | Architect | P1.2 |

#### 1B: Data Infrastructure

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P1.7 | CDK: Aurora PostgreSQL Serverless v2 cluster (Indiana: min 0.5 ACU, max 16 ACU; `aurora-postgresql16`; Multi-AZ) | 1.5 | Architect | P1.1, P1.2, P1.6 |
| P1.8 | CDK/config: Aurora parameter group — `wal_level=logical`, `max_replication_slots=5`, `max_slot_wal_keep_size=10240` (10GB) | 0.5 | Architect | P1.7 |
| P1.9 | CDK: ClickHouse EC2 (`r6i.xlarge`, 200GB `gp3` EBS, Amazon Linux 2023 AMI, user-data installs ClickHouse server 24.x) | 2 | Architect | P1.1, P1.2 |
| P1.10 | CDK: Valkey on ElastiCache Serverless (or `cache.r7g.large` for predictable cost) — in isolated subnet, auth token via Secrets Manager | 1 | Architect | P1.1, P1.2, P1.6 |
| P1.11 | CDK: RDS SQL Server instance (`db.t3.medium`, license-included Express or SE, isolated subnet) — TRS RTS replica | 1.5 | Architect | P1.1, P1.2 |
| P1.12 | CDK: S3 buckets — `trs-scores-raw`, `trs-scores-rejected`, `trs-spa-trs`, `trs-spa-admin`, `trs-ch-backups` (all: versioning on, SSE-KMS, block public access) | 0.5 | Architect | P1.6 |

#### 1C: Compute & Messaging

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P1.13 | CDK: ECS Fargate cluster + ECR repos (`trs-api`, `admin-api`, `score-processor`, `cdc-transformer`, `rts-sync`, `aggregate-refresh`) | 1 | Architect | P1.5 |
| P1.14 | CDK: ALB for TRS API (internal) + ALB for Admin API (internal) — HTTPS listener, target groups, health check paths | 1 | Architect | P1.1, P1.13 |
| P1.15 | CDK: SQS queues — `score-ingest` (visibility 300s), `score-ingest-dlq`, `cdc-events`, `cdc-dlq`, `rts-sync-trigger` | 0.5 | Architect | — |
| P1.16 | CDK: SNS topics (`trs-alerts`, `trs-rescore-events`) + EventBridge scheduled rules (school MV refresh 15min, district MV 30min, nightly state, hourly RTS sync) | 0.5 | Architect | P1.15 |
| P1.17 | CDK: API Gateway HTTP API → VPC Link → ALB for TRS API; second API Gateway → Admin ALB | 1 | Architect | P1.14 |
| P1.18 | CDK: S3 event notification on `trs-scores-raw` → SQS `score-ingest` (prefix filter: `scores/`) | 0.5 | Architect | P1.12, P1.15 |

#### 1D: Frontend CDN

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P1.19 | CDK: CloudFront distribution for TRS SPA (S3 OAC origin, managed cache policy for assets, error page for SPA routing) | 1 | Dev 3 + Architect | P1.3, P1.12 |
| P1.20 | CDK: CloudFront distribution for Admin SPA (same pattern as TRS; separate domain) | 0.5 | Dev 3 | P1.19 |
| P1.21 | CDK/Pipeline: CI/CD stage for SPA deployment — React build → `aws s3 sync` → `create-invalidation` | 1 | Dev 3 | P0.5, P1.19 |

#### 1E: Observability Foundation

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P1.22 | CDK: CloudWatch Log Groups with retention policies (30d dev, 90d staging/prod) for all services | 0.5 | Architect | — |
| P1.23 | CDK: X-Ray tracing group + sampling rules for Fargate services + API Gateway | 0.5 | Architect | P1.13 |
| P1.24 | Define structured log schema (JSON: `correlationId`, `tenantId`, `userId`, `endpoint`, `durationMs`, `servedFrom`, `level`) | 0.5 | Architect | — |

---

### Phase 2: Database Schemas & Migrations
**Duration:** Weeks 1–3 | **T-shirt:** M | **Effort:** ~13 days

> **Local-first strategy:** All Postgres changesets (Liquibase) and ClickHouse DDL scripts are authored and validated **locally in Week 1** against the Docker Compose containers from P0.11 — no AWS dependency for script development. Once CDK provisions Aurora (P1.7) and ClickHouse EC2 (P1.9) in Week 2, the same scripts are applied to AWS with zero rework.

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P2.1 | Set up Liquibase project for Postgres schema versioning — `changelog.xml` root, per-changeset SQL files under `infra/sql/postgres/changesets/`; run `liquibase update` against local Docker Compose Postgres to confirm baseline applies cleanly | 0.5 | Dev 2 | P0.11 _(cloud apply: P1.7)_ |
| P2.2 | Postgres DDL: `trs` schema + `student_opportunities` partitioned table (LIST tenant_id → LIST school_year) + all indexes (`idx_so_roster`, `idx_so_student`) | 1.5 | Dev 2 | P2.1 |
| P2.3 | Postgres DDL: `student_component_scores` partitioned table + indexes; `REPLICA IDENTITY FULL` on both score tables | 1 | Dev 2 | P2.1 |
| P2.4 | Postgres DDL: membership tables (`roster_student`, `school_student`, `district_student`, `district_school`, `teacher_roster`) with tenant_id partitioning | 1 | Dev 2 | P2.1 |
| P2.5 | Postgres DDL: config tables (`test_aliases`, `test_keys`, `test_alias_groups`, `tenants`) + indexes | 1 | Dev 2 | P2.1 |
| P2.6 | Postgres DDL: operational tables (`embargo_roles`, `score_ingest_log`, `score_ingest_rejections`, `report_cache`) + expiry index + cleanup pg_cron jobs | 1 | Dev 2 | P2.1 |
| P2.7 | Postgres DDL: materialized views (`mv_school_overall`, `mv_district_overall`, `mv_school_standards_current`) + unique indexes | 1 | Dev 2 | P2.2, P2.4 |
| P2.8 | Postgres: pg_cron extension enable + cron jobs (report_cache cleanup, score_ingest_log cleanup, MV refresh schedule aligned with EventBridge rules) | 0.5 | Dev 2 | P2.6, P2.7 |
| P2.9 | ClickHouse DDL: Bronze layer — `student_scores_bronze` (VersionedCollapsingMergeTree) + `student_component_scores_bronze`; script versioned as `infra/sql/clickhouse/001_bronze.sql`; validated locally | 1 | Dev 1 | P0.11 _(cloud apply: P1.9)_ |
| P2.10 | ClickHouse DDL: Silver layer — `student_scores_silver` (AggregatingMergeTree) + `student_scores_silver_mv` MV | 1 | Dev 1 | P2.9 |
| P2.11 | ClickHouse DDL: Gold layer — `school_aggregates_gold` + `district_aggregates_gold` + `component_aggregates_gold` + all Bronze-to-Gold MVs | 2 | Dev 1 | P2.9 |
| P2.12 | ClickHouse DDL: membership mirrors (`membership_school_mirror`, `membership_district_mirror` — ReplacingMergeTree) + diagnostic tables | 0.5 | Dev 1 | P2.9 |
| P2.13 | Schema validation: seed test data scripts (10 students, 2 schools, 1 district) — verify inserts, cascades, sign-pair collapse, MV population | 1 | Dev 1 + Dev 2 | P2.9–P2.12 |

---

### Phase 3: Score Ingestion Pipeline
**Duration:** Weeks 3–4 | **T-shirt:** M | **Effort:** ~13 days

> **Blocker note:** P1.18 (S3 → SQS event notification) and P2.2/P2.3 (score tables) must be complete before this phase can be fully tested.

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P3.1 | .NET 8 `BackgroundService` project scaffold for Score Processor — SQS consumer loop, Dockerfile, `/health` endpoint | 1 | Dev 1 | P1.13 |
| P3.2 | Score Processor: parse + validate score JSON (required fields, OppStatus enum, ConditionCode enum, UTF-8 encoding) | 1.5 | Dev 1 | P3.1 |
| P3.3 | Score Processor: data normalization — `TestKey → testalias_id` lookup via `test_keys` table; field mapping to Postgres column names | 1.5 | Dev 1 | P3.2, P2.5 |
| P3.4 | Score Processor: `is_aggregate_eligible` flag computation (OppStatus + ConditionCode rules) | 0.5 | Dev 1 | P3.3 |
| P3.5 | Score Processor: idempotency check against `score_ingest_log` (s3_key + etag); no-op on duplicate | 0.5 | Dev 1 | P3.4, P2.6 |
| P3.6 | Score Processor: batch UPSERT `student_opportunities` + `student_component_scores` (ON CONFLICT WHERE EXCLUDED.date_scored > existing) | 1.5 | Dev 1 | P3.5, P2.2, P2.3 |
| P3.7 | Score Processor: rejection handling — `UNKNOWN_TEST_CONFIG`, `VALIDATION_FAILED`, `DATA_CONFLICT` → write `score_ingest_rejections`, copy file to rejection S3 bucket, ACK SQS | 1 | Dev 1 | P3.6, P2.6 |
| P3.8 | Score Processor: CloudWatch custom metrics (files processed/hr, rejection count, batch duration p95) | 0.5 | Dev 1 | P3.7 |
| P3.9 | Unit tests: parse/validate, normalization, UPSERT idempotency, all rejection paths, eligibility flag logic | 2 | Dev 1 | P3.2–P3.7 |
| P3.10 | CDK: Fargate task definition for Score Processor (ECR image, CPU/mem, env vars from Secrets Manager, SQS event source) | 0.5 | Architect + Dev 1 | P3.1, P1.13 |
| P3.11 | End-to-end smoke test: drop sample score file in S3 → verify row appears in `student_opportunities` within 60s | 0.5 | Dev 1 | P3.10 |
| P3.12 | `DevDataSeeder` full implementation: seed config + roster hierarchy + embargo (step 1–3); fake opps using same field rules as Score Processor (OppStatus distribution, ConditionCode probabilities, `is_aggregate_eligible`, cut-score PL derivation, `date_scored` millisecond precision); direct-write to Postgres AND ClickHouse Bronze in parallel; `--reset`, `--skip-opps`, `--skip-postgres` CLI flags; verify ClickHouse MVs auto-populate Silver/Gold after seed | 2 | Dev 1 + Dev 2 | P3.3, P3.4, P2.9 |

---

### Phase 4: CDC Pipeline (PeerDB + Sign-Pair Transformer)
**Duration:** Weeks 3–5 | **T-shirt:** L | **Effort:** ~15 days

> **Critical:** This phase produces the ClickHouse Bronze data that all Gold aggregates depend on.  
> **Blocker:** P1.8 (`wal_level=logical`) and P2.9 (Bronze DDL) must be complete.

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P4.1 | PeerDB OSS: Fargate task definition + config volume (CDK); confirm PeerDB version compatible with Aurora PG 16 replication slots | 1.5 | Architect | P1.9, P1.13, P1.8 |
| P4.2 | PeerDB CDC Job #1: `student_opportunities` + `student_component_scores` → SQS `cdc-events` queue (before+after row images, REPLICA IDENTITY FULL confirmed) | 1.5 | Architect | P4.1, P2.2 |
| P4.3 | PeerDB CDC Job #2: `school_student` + `district_student` → ClickHouse `membership_school_mirror` / `membership_district_mirror` (direct, bypasses Transformer) | 1 | Architect | P4.1, P2.12 |
| P4.4 | .NET 8 Sign-Pair Transformer: `BackgroundService` scaffold — SQS consumer, ClickHouse HTTP client (`ClickHouse.Client` NuGet), retry policy | 1 | Dev 1 | P1.13, P1.15 |
| P4.5 | Transformer: INSERT event → emit single `sign=+1` Bronze row batch to ClickHouse | 1 | Dev 1 | P4.4, P2.9 |
| P4.6 | Transformer: UPDATE (rescore) event → emit atomic `-1/+1` pair in a **single** HTTP batch INSERT | 1.5 | Dev 1 | P4.5 |
| P4.7 | Transformer: DELETE event → emit `sign=-1, is_deleted=1` Bronze row; use `DateTime.UtcNow` as synthetic `date_scored` | 1 | Dev 1 | P4.5 |
| P4.8 | Transformer: component scores variant — same logic routing to `student_component_scores_bronze` | 1 | Dev 1 | P4.6, P2.9 |
| P4.9 | Transformer: error handling — exponential backoff (1→2→4→30s) on ClickHouse error; 4xx → CloudWatch alarm `TRS/CDC/SchemaError`; bad events → SQS DLQ | 1 | Dev 1 | P4.5–P4.8 |
| P4.10 | End-to-end validation: INSERT score in Postgres → verify Bronze row → Silver aggregation → Gold `avgMerge` returns correct value | 1.5 | Dev 1 + Architect | P4.2–P4.9 |
| P4.11 | End-to-end validation: UPDATE (rescore) in Postgres → verify Bronze sign-pair appears → Gold aggregate corrected; no double counting | 1 | Dev 1 | P4.10 |
| P4.12 | End-to-end validation: DELETE in Postgres → verify Bronze `-1` collapses the original `+1` → Gold aggregate decremented | 0.5 | Dev 1 | P4.11 |
| P4.13 | Unit tests: Transformer INSERT/UPDATE/DELETE emission logic, batch atomicity, retry/DLQ logic | 1.5 | Dev 1 | P4.5–P4.9 |
| P4.14 | CDK: containerize Transformer, push to ECR, Fargate service definition; PeerDB Fargate task finalized | 1 | Architect + Dev 1 | P4.4, P1.13 |

---

### Phase 5: RTS Replication Pipeline
**Duration:** Weeks 3–5 | **T-shirt:** M | **Effort:** ~13 days

> **Architecture:** Master RTS MSSQL (external, owned by RTS team) → TRS RDS MSSQL replica (SQL Server transactional replication) → Fargate `RtsSyncWorker` (hourly, delta-based) → Aurora Postgres relationship tables.

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P5.1 | Confirm with RTS team: read credentials to Master RTS MSSQL, identify delta-tracking column (e.g., `modified_date`), list of tables to replicate | 1 | PM + Architect | P0.10 |
| P5.2 | Configure SQL Server transactional replication: Master RTS as Publisher → TRS MSSQL RDS as Subscriber (relevant tables only: roster, school, district relationships) | 2 | Dev 2 + Architect | P5.1, P1.11 |
| P5.3 | Validate replication: confirm TRS MSSQL replica receives changes from Master within expected latency | 1 | Dev 2 | P5.2 |
| P5.4 | .NET 8 `RtsSyncWorker`: Fargate `BackgroundService` scaffold — hourly EventBridge trigger, `last_sync_date` stored in SSM Parameter Store | 1.5 | Dev 2 | P1.13, P1.16 |
| P5.5 | RtsSyncWorker: delta query logic — `SELECT * FROM rts.roster WHERE modified_date > @last_sync_date` (per table); handle first run (no last_sync_date = full load) | 2 | Dev 2 | P5.4, P5.3 |
| P5.6 | RtsSyncWorker: transformation queries — map RTS schema to Postgres `trs.roster_student`, `trs.school_student`, `trs.district_student`, `trs.district_school`, `trs.teacher_roster` | 2 | Dev 2 | P5.5, P2.4 |
| P5.7 | RtsSyncWorker: UPSERT to Postgres relationship tables (ON CONFLICT DO UPDATE); handle soft-deletes | 1 | Dev 2 | P5.6 |
| P5.8 | RtsSyncWorker: update `last_sync_date` in SSM on success; CloudWatch alarm on sync failure or duration > 30min | 0.5 | Dev 2 | P5.7 |
| P5.9 | Validate full flow: run full initial load, confirm Indiana roster sample data appears in Postgres membership tables | 1 | Dev 2 | P5.7 |
| P5.10 | Unit + integration tests: delta detection, schema transformation, UPSERT idempotency, first-run full-load path | 1.5 | Dev 2 | P5.5–P5.8 |
| P5.11 | CDK: Fargate task definition + EventBridge scheduled rule (hourly) for RtsSyncWorker | 0.5 | Architect + Dev 2 | P5.4, P1.13 |

---

### Phase 6: Valkey Distributed Cache Integration
**Duration:** Weeks 4–5 | **T-shirt:** S | **Effort:** ~7.5 days

> **Architecture:** Valkey sits as the **new Tier 0** ahead of `report_cache`. Also used for test config lookups, translations, and EmbargoService in-line cache (replacing pure in-process Dictionary).

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P6.1 | .NET shared library `TRS.Caching`: `IValkeyCacheService` implementation using `StackExchange.Redis`; connection string from Secrets Manager | 1 | Dev 2 | P1.10 |
| P6.2 | Cache key conventions document + implementation: `{scope}:{tenantId}:{scopeId}:{schoolYear}:{testAliasId}`, `testconfig:{tenantId}:{testAliasId}`, `embargo:{tenantId}:{testAliasId}` | 0.5 | Dev 2 + Architect | P6.1 |
| P6.3 | Integrate Valkey into `EmbargoService`: replace in-process Dictionary with Valkey (60s TTL for embargo_until, 5min TTL for embargo_roles) | 1 | Dev 1 | P6.1, P6.2 |
| P6.4 | Integrate Valkey into test config lookups (`test_aliases`) and S3 config cache — 10min TTL; Admin API invalidates on config change | 1 | Dev 2 | P6.1, P6.2 |
| P6.5 | Wire Valkey as Tier 0 in the fallback chain (Valkey → `report_cache` → ClickHouse Gold → Postgres MV → Postgres live) | 1.5 | Dev 1 | P6.1, P6.2 — wired during P7 API work |
| P6.6 | Cache invalidation: Admin API emits SNS rescore notification → Lambda/Fargate listener evicts targeted Valkey keys by pattern | 1 | Dev 2 | P6.5, P1.16 |
| P6.7 | Unit tests: cache hit/miss path, TTL expiry simulation, key eviction, connection failure graceful degradation (miss → proceed to next tier) | 1.5 | Dev 2 | P6.3–P6.6 |

---

### Phase 7: TRS Reporting API (Fargate / .NET 8)
**Duration:** Weeks 4–7 | **T-shirt:** XL | **Effort:** ~25 days

> **Blocker chain:** P2 (schemas) → P4 (CDC + Gold data) → P6 (Valkey) must be done before this phase produces fully tested results.  
> **Parallel:** Scaffold + auth endpoints can start in Week 4 while CDC is still being validated.

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P7.1 | .NET 8 ASP.NET Core Minimal API project scaffold — Dockerfile, `GET /health`, environment config from env vars/Secrets Manager | 1 | Dev 1 | P1.13 |
| P7.2 | Shared middleware: JWT validation (RS256, JWKS endpoint cached 60min) → extract `tenant_id`, `roles` claims; 401 on invalid token | 1.5 | Dev 1 | P0.9, P7.1 |
| P7.3 | Repository layer: `IPostgresRepository` — `report_cache` read/write, `mv_school_overall` query, `mv_district_overall` query, per-student live query, embargo reads | 2 | Dev 1 | P2.5, P2.6, P2.7 |
| P7.4 | Repository layer: `IClickHouseRepository` — Gold school query, Gold district query, Gold component query, `/ping` health check | 2 | Dev 1 | P2.11 |
| P7.5 | `IFallbackChain` orchestrator: Valkey → report_cache → ClickHouse health check (200ms probe) → Gold query → MV query → live query; `servedFrom` envelope | 2 | Dev 1 | P7.3, P7.4, P6.5 |
| P7.6 | `EmbargoService`: embargo check + embargo_roles check (Valkey-backed); throw `EmbargoException` → 404 response | 1.5 | Dev 1 | P7.2, P6.3, P2.5, P2.6 |
| P7.7 | `GET /v1/{tenantId}/tests?schoolYear={Y}` — test discovery endpoint with embargo filter (omit embargoed for non-privileged callers) | 1 | Dev 1 | P7.6, P7.3 |
| P7.8 | `GET /v1/{tenantId}/roster/{rosterId}/students` — per-student score list (live Postgres, embargo check, pagination) | 1.5 | Dev 1 | P7.6, P7.3 |
| P7.9 | `GET /v1/{tenantId}/roster/{rosterId}/report` — roster aggregate (always live Postgres; no cache; small N) | 1.5 | Dev 1 | P7.8, P7.5 |
| P7.10 | `GET /v1/{tenantId}/school/{schoolId}/report` — school aggregate via full fallback chain (cache TTL 15min) | 2 | Dev 1 | P7.5, P7.4 |
| P7.11 | `GET /v1/{tenantId}/district/{districtId}/report` — district aggregate via fallback chain (cache TTL 5min) | 1.5 | Dev 1 | P7.10 |
| P7.12 | `GET /v1/{tenantId}/school/{schoolId}/standards` — school component aggregates (Gold component query; MV fallback) | 1.5 | Dev 1 | P7.10, P7.4 |
| P7.13 | `GET /v1/{tenantId}/district/{districtId}/standards` — district component; HTTP 503 if Valkey/CH both unavailable (no live Postgres fallback per design) | 1 | Dev 1 | P7.12 |
| P7.14 | Aggregate Refresh Fargate job: reads ClickHouse Gold → writes `report_cache` (school 15min, district 5min, state nightly) | 1.5 | Dev 1 | P7.3, P7.4 |
| P7.15 | `POST /v1/{tenantId}/student/{studentId}/isr` — Full ReportsHub client: validate embargo, serialize score payload (`student_opportunities` + component scores), `POST` to ReportsHub endpoint, handle `202 Accepted` async or `200` sync PDF URL response, return URL to caller; Polly circuit breaker + retry (max 2); 503 on open circuit | 4 | Dev 1 | P7.8; **ReportsHub API contract must be locked by Apr 7** |
| P7.16 | `GET /v1/{tenantId}/student/{studentId}/sdf?testKey={K}` — Full CRS client: pass `studentKey` + `testKey` to CRS, stream Excel binary response back to caller with `Content-Disposition: attachment`; Polly circuit breaker + retry (max 2); 503 on open circuit | 4 | Dev 1 | P7.7; **CRS API contract must be locked by Apr 7** |
| P7.17 | `servedFrom` response envelope + CloudWatch custom metric `TRS/API/ClickHouseFallback` count | 0.5 | Dev 1 | P7.10 |
| P7.18 | Unit tests: embargo logic (all edge cases), fallback chain routing (mock CH down), all endpoint behaviors, ISR/SDF client stubs | 3 | Dev 1 | P7.6–P7.16 |
| P7.19 | Integration tests: real Postgres + real ClickHouse (local Docker Compose) — school/district queries, rescore correction, fallback | 2 | Dev 1 | P7.18 |
| P7.20 | CDK: Fargate service definition — TRS API (1 vCPU / 2GB, 2 min tasks, auto-scale on ALB RequestCount); ALB target group | 1 | Architect + Dev 1 | P7.1, P1.13, P1.14 |

---

### Phase 8: Admin API (Fargate / .NET 8)
**Duration:** Weeks 4–7 | **T-shirt:** L | **Effort:** ~16 days

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P8.1 | .NET 8 Admin API project scaffold — Dockerfile, `GET /health`, shared JWT middleware (NuGet from shared library) | 1 | Dev 2 | P1.13, P7.2 |
| P8.2 | Tenant management: `GET/POST /v1/admin/tenants` | 0.5 | Dev 2 | P8.1, P2.5 |
| P8.3 | Test alias CRUD: `GET/POST/PUT /v1/admin/{tenantId}/test-aliases` + `test_keys`, `test_alias_groups` | 2.5 | Dev 2 | P8.1, P2.5 |
| P8.4 | Test-alias config upload: `PUT /v1/admin/{tenantId}/test-aliases/{id}/config` — validates JSON schema, uploads to `s3://trs-config/{tenantId}/{testAliasId}.json` (performance levels, measures, score bands, standards) | 1.5 | Dev 2 | P8.3 |
| P8.5 | Test-alias config download: `GET /v1/admin/{tenantId}/test-aliases/{id}/config` — reads from S3 config bucket | 0.5 | Dev 2 | P8.3 |
| P8.6 | Embargo management: `PUT /v1/admin/{tenantId}/test-aliases/{id}/embargo` (set/clear `embargo_until`); `PUT .../embargo-roles` (manage `embargo_roles` table) | 1.5 | Dev 2 | P8.3, P2.6 |
| P8.7 | Embargo change → Valkey cache invalidation: on embargo update, publish SNS event → Valkey evicts `embargo:*` keys | 1 | Dev 2 | P8.6, P6.6 |
| P8.8 | FlightPlan integration: `IFlightPlanClient` — `GET /tests?client={X}&schoolYear={Y}`; Admin endpoint to import and create test aliases | 2 | Dev 2 | P8.3 — FlightPlan contract |
| P8.9 | CSR integration: `ICsrClient` — browse publications, fetch standards for a publication; Admin endpoint to associate standards to test alias | 2 | Dev 2 | P8.5 — CSR contract |
| P8.10 | ITS integration: `IItsClient` — fetch PLD measures + cut scores for a test; generate test-alias config JSON and upload to S3 config bucket | 2 | Dev 2 | P8.4 — ITS contract |
| P8.11 | Score rejection management: `GET /v1/admin/{tenantId}/rejections` + `POST .../replay` (re-queue to `score-ingest` SQS) | 1 | Dev 2 | P8.1, P3.7 |
| P8.12 | Unit tests: all CRUD endpoints, embargo logic, cache invalidation, FlightPlan/CSR/ITS client stubs | 2.5 | Dev 2 | P8.3–P8.11 |
| P8.13 | CDK: Fargate service definition for Admin API (0.5 vCPU / 1GB, 1 task); ALB target group | 0.5 | Architect + Dev 2 | P8.1, P1.14 |

---

### Phase 9: TRS Frontend SPA (React + TypeScript)
**Duration:** Weeks 5–8 | **T-shirt:** L | **Effort:** ~19 days

> **Parallel strategy:** Dev 3 uses Mock Service Worker (MSW) to mock all TRS API endpoints locally until Phase 7 is production-ready.

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P9.1 | React + TypeScript project setup: Vite, React Router v6, TanStack Query, Tailwind CSS + component library (e.g. shadcn/ui), ESLint, Vitest | 1 | Dev 3 | — |
| P9.2 | OIDC authentication flow: `oidc-client-ts` / `react-oidc-context` SSO login/logout; JWT stored in memory (not localStorage); protected route guard; tenant_id extraction | 2 | Dev 3 | P0.9 |
| P9.3 | MSW mock layer: mock all TRS API endpoints for local development | 1 | Dev 3 | — concurrent with P7 |
| P9.4 | Typed API client layer: `axios` wrappers matching TRS API OpenAPI spec; auto-attach Bearer token; handle 401 redirect, 404 embargo response | 1 | Dev 3 | P9.1 |
| P9.5 | Test selection view: `GET /tests?schoolYear={Y}` → filterable list by subject/grade; navigate to report | 1.5 | Dev 3 | P9.4, P7.7 |
| P9.6 | Roster/Class report view: student score list table (scale score, perf level, component scores); aggregate summary header bar | 2.5 | Dev 3 | P9.5, P7.8, P7.9 |
| P9.7 | School aggregate report view: avg score, PL distribution chart (bar), student count; `servedFrom` badge for observability | 2 | Dev 3 | P9.5, P7.10 |
| P9.8 | District aggregate report view: school comparison table + district-level metrics | 2 | Dev 3 | P9.7, P7.11 |
| P9.9 | Standards/component report view (school scope): per-standard breakdown table with PL distribution | 2 | Dev 3 | P9.7, P7.12 |
| P9.10 | ISR PDF button: call stub ISR endpoint → open PDF URL in new tab (no-op if stub returns 503 in MVP) | 1 | Dev 3 | P9.6, P7.15 |
| P9.11 | SDF Excel download button: call stub SDF endpoint → trigger file download | 0.5 | Dev 3 | P9.6, P7.16 |
| P9.12 | Global: loading skeletons, error boundaries, empty states, 404 handling for embargoed test (no "Access Denied" message — silent 404 per design) | 1 | Dev 3 | P9.6–P9.9 |
| P9.13 | Accessibility pass: WCAG 2.1 AA — keyboard nav, ARIA labels, color contrast, screen reader test on report tables | 1 | Dev 3 | P9.6–P9.9 |
| P9.14 | Unit tests (Vitest + React Testing Library): all views, auth flow, embargo 404 handling, API error states | 2 | Dev 3 | P9.1–P9.12 |
| P9.15 | CDK pipeline integration: build script (`npm run build`) → S3 sync → CloudFront invalidation on deploy | 0.5 | Dev 3 + Architect | P1.19, P1.21 |

---

### Phase 10: Admin Frontend SPA (React + TypeScript)
**Duration:** Weeks 5–8 | **T-shirt:** M | **Effort:** ~12 days

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P10.1 | React project setup: shared Vite config + shared component library config with TRS app; separate build target | 0.5 | Dev 3 | P9.1 |
| P10.2 | OIDC auth flow: same pattern as TRS app; verify admin role claim before rendering — redirect unauthorized users | 1 | Dev 3 | P0.9, P8.1 |
| P10.3 | Typed API client layer for Admin API | 0.5 | Dev 3 | P10.1 |
| P10.4 | Test configuration UI: import tests from FlightPlan (select from datatable), create/edit test aliases, manage test keys + groups | 3 | Dev 3 | P10.3, P8.3, P8.8 |
| P10.5 | Standards association UI: browse CSR publications tree, select standards, save associations to test | 2 | Dev 3 | P10.4, P8.9 |
| P10.6 | Performance levels UI: view ITS PLD import, display cut scores per level, allow manual override | 1.5 | Dev 3 | P10.4, P8.10 |
| P10.7 | Embargo management UI: date picker for `embargo_until`, role picker from `embargo_roles`, clear embargo button | 1.5 | Dev 3 | P10.4, P8.6 |
| P10.8 | Score rejection dashboard: paginated list of rejections, reason display, "Replay" button with confirmation dialog | 1 | Dev 3 | P10.2, P8.11 |
| P10.9 | Unit tests: config forms, embargo UI, FlightPlan import flow, CSR standards picker, rejection replay | 1.5 | Dev 3 | P10.4–P10.8 |
| P10.10 | CDK pipeline integration: build → S3 sync → CloudFront invalidation | 0.5 | Dev 3 | P1.20, P1.21 |

---

### Phase 11: Observability, Monitoring & Alerting
**Duration:** Weeks 7–9 | **T-shirt:** M | **Effort:** ~9.5 days

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P11.1 | Implement structured log format in all .NET services (Serilog JSON sink) per schema defined in P1.24; `correlationId` header propagated from API Gateway | 1.5 | Dev 1 + Dev 2 | P1.24, P7.1, P8.1 |
| P11.2 | OpenTelemetry SDK: traces in TRS API + Admin API → AWS X-Ray; custom span attributes (`tenantId`, `servedFrom`, `fallbackTier`) | 1.5 | Dev 1 | P11.1 |
| P11.3 | CloudWatch alarms — critical (SNS → email/Slack): CDC lag > 5min, `cdc-dlq` depth > 0, 5xx rate > 1%, ClickHouse fallback rate > 10% | 1 | Architect | P4.14 |
| P11.4 | CloudWatch alarms — warning: RTS sync failure, sync duration > 30min, Score Processor rejection rate spike, `score-ingest-dlq` depth > 0 | 0.5 | Dev 2 | P5.11, P3.11 |
| P11.5 | CloudWatch alarms — infrastructure: Aurora CPU > 80%, Valkey memory > 80%, ClickHouse EC2 CPU > 75% | 0.5 | Architect | P1.7, P1.9, P1.10 |
| P11.6 | CloudWatch dashboard "Score Ingestion": files/hr, validation failures, rejection count/type, successful UPSERT rate | 1 | Dev 1 | P3.11 |
| P11.7 | CloudWatch dashboard "CDC Pipeline": PeerDB LSN lag (minutes behind), Transformer queue depth, Bronze insert rate per table | 1 | Dev 1 | P4.14 |
| P11.8 | CloudWatch dashboard "TRS API": p50/p95/p99 latency by endpoint, fallback tier breakdown pie, active Fargate task count, Valkey hit ratio | 1 | Dev 1 | P7.20 |
| P11.9 | ClickHouse EC2 custom metrics: CloudWatch agent + cron script to emit `CH_BackgroundMergeQueue`, `CH_QueryLatencyP95` → CloudWatch | 1 | Dev 2 | P1.9 |
| P11.10 | SNS → Slack webhook (or email) for all P1/P2 alarms; confirm on-call rotation contacts | 0.5 | Architect + PM | P11.3 |
| P11.11 | Runbook doc stubs: ClickHouse rebuild procedure, PeerDB initial load, RTS sync manual trigger, score replay | 1 | Architect + PM | — |

---

### Phase 12: Testing & QA
**Duration:** Weeks 8–10 | **T-shirt:** L | **Effort:** ~20 days

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P12.1 | Integration test suite: Score Processor → Postgres → CDC → ClickHouse Gold (fully automated, Docker Compose with real PG + CH containers) | 2.5 | Dev 1 | All of Phase 3, Phase 4 |
| P12.2 | Integration test suite: RTS sync → Postgres membership tables → ClickHouse membership mirrors | 1.5 | Dev 2 | Phase 5 complete |
| P12.3 | Integration test suite: TRS API three-tier fallback (inject CH unreachable → verify Postgres MV path; inject CH + MV stale → verify live query path) | 2 | Dev 1 | Phase 7 complete |
| P12.4 | Integration test suite: Admin API — create test config → embargo it → TRS API returns 404 for non-privileged user → returns data for privileged user | 1.5 | Dev 2 | Phase 8 complete |
| P12.5 | Integration test: rescore — UPDATE `student_opportunities.score_value` → wait for CDC → verify Gold aggregate changes correctly (+/- delta only) | 2 | Dev 1 | P4.11 |
| P12.6 | Playwright E2E: TRS app — login via SSO, select Indiana test, view roster report, view school report, view district report | 2 | Dev 3 | Phase 9 complete |
| P12.7 | Playwright E2E: Admin app — login, import test from FlightPlan, add standards, set embargo date; verify TRS app hides test | 2 | Dev 3 | Phase 10 complete |
| P12.8 | Playwright E2E: embargo scenario — confirm regular user gets no test in list; confirm embargo-role user sees test | 1 | Dev 3 | P12.6, P12.7 |
| P12.9 | Security review: JWT edge cases (expired, tampered signature, wrong tenant_id, missing roles claim); SQL injection prevention audit; input validation review | 2 | Architect | Phase 7, Phase 8 |
| P12.10 | Performance validation: seed Indiana-scale data (1.3M students, 13M score rows) in staging; verify school aggregate < 30ms from ClickHouse Gold | 2 | Dev 1 + Architect | Phase 13 staging deploy |
| P12.11 | Regression buffer: fix issues identified during testing (estimated) | 3 | All devs | P12.1–P12.9 |

---

### Phase 13: Production Readiness & Go-Live
**Duration:** Weeks 9–10 | **T-shirt:** M | **Effort:** ~11 days

| ID | Task | Est. Days | Role(s) | Blocked By |
|---|---|---|---|---|
| P13.1 | CDK: production environment config (`cdk.context.json` prod overrides) — Aurora sizing for Indiana, ClickHouse `r6i.xlarge`, Valkey prod tier | 1 | Architect | Phase 1 complete |
| P13.2 | Production Secrets Manager: populate all production secrets — DB passwords, API keys for FlightPlan/CSR/ITS, SSO JWKS URL | 0.5 | Architect + PM | P0.6 |
| P13.3 | Deploy all CDK stacks to **staging** environment; verify stack outputs, cross-stack references, VPC connectivity | 1.5 | Architect | P12.11, P13.1 |
| P13.4 | Staging: PeerDB Initial Load — snapshot Aurora staging → ClickHouse; verify Bronze → Gold row counts match expected | 1 | Architect | P13.3 |
| P13.5 | Staging smoke tests: deploy TRS + Admin SPAs → login via SSO → create test config → view report → verify `servedFrom=clickhouse_gold` | 1.5 | All devs | P13.4 |
| P13.6 | Staging performance spot check: run school + district aggregate queries; verify latency targets (school < 30ms from CH Gold, < 200ms total) | 0.5 | Dev 1 | P13.5 |
| P13.7 | Deploy all CDK stacks to **production** (gated approval step in pipeline) | 1 | Architect | P13.6 |
| P13.8 | Production: PeerDB Initial Load — populate ClickHouse from production Aurora; switch to streaming CDC mode | 1 | Architect | P13.7 |
| P13.9 | Production smoke tests: score file ingestion → verify report visible in TRS app; Admin embargo toggle → verify TRS hides test | 1 | All devs | P13.8 |
| P13.10 | Go-Live runbook: cutover checklist, rollback procedure (CDK rollback + PeerDB pause), on-call contacts, escalation path | 0.5 | Architect + PM | — |
| P13.11 | Hotfix buffer (May 18–20): reserved for last-minute production issues | 2 | All devs | P13.9 |

---

## 9. Critical Path Diagram

```
Week 1          Week 2-3           Week 3-5         Week 4-7        Week 8-10
─────────────────────────────────────────────────────────────────────────────
[P0: Setup]
     │
     ▼
[P1: CDK Infra] ──────────────────────────────────────────────» all services depend on this
     │
     ├──[P2: DB Schemas] ──────►[P3: Score Processor]──────►[P4 validation]
     │        │                                                     │
     │        └──[P4: CDC Pipeline] ──────────────────────────────►│
     │        │                                                     │
     │        └──[P5: RTS Sync] ───────────────────►[P7.9 roster data available]
     │
     ├──[P2 CH DDL] ──►[P4: Transformer] ──►[P4 E2E validation] ──►[P7: TRS API Gold queries]
     │
     ├──[P6: Valkey] ─────────────────────────────────────────────►[P7.5 Tier-0 fallback]
     │
     ├──[P7: TRS API] ───────────────────────────────────────────►[P9.4 API client]──►[P9: TRS SPA]
     │       │
     │       └── CRITICAL: P4 Gold data must be ready before P7.10 school endpoint validates
     │
     ├──[P8: Admin API] ─────────────────────────────────────────►[P10.3]──►[P10: Admin SPA]
     │
     ├──[P11: Observability] ──────────────────────────────────────────────►[P12.10]
     │
     └──[P12: Testing] ───────────────────────────────────────────────────►[P13: Go-Live]

HARD BLOCKERS (nothing downstream can proceed without these):
══════════════════════════════════════════════════════════════
  P1.7 (Aurora UP)              → P2, P3, P4, P5, P7, P8
  P1.9 (ClickHouse EC2 UP)      → P2.9, P4, P7.4
  P2.2 (PG score tables)        → P3, P4.2, P7.3
  P4.10 (E2E CDC validated)     → P7.10 (trustworthy Gold data)
  P7.2 (JWT middleware)         → P7.6, P8.1, P9.2, P10.2
  P0.9 (SSO contract confirmed) → P7.2 (JWT implementation)
  P5.1 (RTS access confirmed)   → entire Phase 5
```

---

## 10. Post-MVP Backlog

Items confirmed out of MVP scope. Plan Phase 2 sprint after go-live stabilization (2 weeks post May 20).

| Item | Depends On | Priority |
|---|---|---|
| ReportsHub ISR — async PDF generation (if API supports polling/webhook pattern) | Post go-live optimization | P2 |
| CRS SDF — batch/multi-student export (if CRS adds bulk API) | Post go-live optimization | P3 |
| State-level aggregate reports (nightly pre-compute) | MVP go-live stable | P2 |
| Full Playwright E2E test coverage | Phase 2 sprint | P2 |
| Indiana-scale load test (1.3M students, 100 concurrent) | Post go-live prod data | P1 |
| ClickHouse Silver roster reads (replace Postgres path) | Post go-live | P3 |
| Student demographics (`student_attributes` table) | TBD data source | P3 |
| District-scope component aggregates live Postgres fallback | Post go-live | P2 |
| ClickHouse Daily S3 backup Lambda | Post go-live | P2 |
| Reserved EC2 pricing (ClickHouse) | 1 month post go-live | P2 |
| Multi-tenant rollout (TX, VA, IL) | Indiana MVP validated | P1 |

---

*Document version 1.0 — March 12, 2026*  
*Next review: End of Sprint 1 (March 18, 2026)*
