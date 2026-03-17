# TRS Platform — AWS Architecture & Infrastructure Diagrams

**Version:** 1.0  
**Date:** March 2026  
**Stack:** AWS CDK (TypeScript) · .NET 8 Fargate · Aurora PostgreSQL Serverless v2 · ClickHouse EC2 · ElastiCache Valkey

---

## Contents

1. [TRS — Full Platform Architecture](#1-trs--full-platform-architecture)
2. [VPC Network Topology & Security Groups](#2-vpc-network-topology--security-groups)
3. [Score Ingestion Pipeline](#3-score-ingestion-pipeline)
4. [CDC Pipeline — Postgres to ClickHouse](#4-cdc-pipeline--postgres-to-clickhouse)
5. [TRS API Read Path & Fallback Chain](#5-trs-api-read-path--fallback-chain)
6. [RTS Replication Pipeline](#6-rts-replication-pipeline)
7. [Frontend Delivery & SSO Auth Flow](#7-frontend-delivery--sso-auth-flow)
8. [CDK Stack Deployment Order](#8-cdk-stack-deployment-order)

---

## 1. TRS — Full Platform Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────────────┐
│  TRS — Architecture                                                                     │
├──────────────────────────────────┬──────────────────────────────────────────────────────┤
│  EXTERNAL                        │  AWS EDGE                                            │
│                                  │                                                      │
│  ┌─────────────────┐             │  ┌───────────────────┐   ┌───────────────────┐      │
│  │  Browser        │─────────────┼─▶│  CloudFront       │   │  CloudFront       │      │
│  │  Teachers       │             │  │  TRS SPA          │   │  Admin SPA        │      │
│  │  Admins         │─────────────┼─▶│  *.trs.example    │   │  *.admin-trs.*    │      │
│  └─────────────────┘             │  └────────┬──────────┘   └────────┬──────────┘      │
│                                  │           │ OAC                   │ OAC             │
│  ┌─────────────────┐             │  ┌────────▼──────────────────────▼──────────┐       │
│  │  Score Delivery │             │  │  S3  trs-spa-trs  /  trs-spa-admin       │       │
│  │  System         │             │  │  SSE-KMS · versioning · block public      │       │
│  └────────┬────────┘             │  └───────────────────────────────────────────┘       │
│           │ PUT file             │                                                      │
│           │                      │  ┌───────────────────┐   ┌───────────────────┐      │
│           │                      │  │  API Gateway HTTP  │   │  API Gateway HTTP │      │
│  ┌────────▼────────┐             │  │  trs-reporting-api │   │  admin-api        │      │
│  │  RTS (external) │             │  │  VPC Link → ALB    │   │  VPC Link → ALB   │      │
│  │  Master MSSQL   │             │  └────────┬──────────┘   └────────┬──────────┘      │
│  └────────┬────────┘             │           │                       │                 │
│           │ SQL Server           └───────────┼───────────────────────┼─────────────────┘
│           │ replication                      │                       │
└───────────┼──────────────────────────────────┼───────────────────────┼─────────────────┘
            │                                  │                       │
            │           ┌──────────────────────┼───────────────────────┼──────────────────┐
            │           │  EXISTING VPC                                │                  │
            │           │                      │                       │                  │
            │           │  PUBLIC SUBNETS (AZ-a + AZ-b)               │                  │
            │           │  ┌───────────────────▼───────────────────────▼───────────────┐  │
            │           │  │  ALB: TRS API (sg-alb-trs)  ALB: Admin API (sg-alb-admin) │  │
            │           │  │  HTTPS :443 inbound from API Gateway VPC Link only        │  │
            │           │  └─────────────────┬────────────────────┬───────────────────┘  │
            │           │                    │                    │                       │
            │           │  PRIVATE SUBNETS — NAT Gateway egress                                                │
            │           │  ┌─────────────────▼────────────────────▼────────────────────────────────────────┐  │
            │           │  │  ECS Fargate Cluster  (sg-fargate-tasks)                                       │  │
            │           │  │                                                                                │  │
            │           │  │  ◄─────── READ PATH ──────────────────────────────────────────────────────►   │  │
            │           │  │  ┌────────────────────────────────────────────────────────────────────────┐   │  │
            │           │  │  │                                                                        │   │  │
            │           │  │  │  ┌──────────────────────────┐   ┌──────────────────────────┐          │   │  │
            │           │  │  │  │  TRS Reporting API        │   │  Admin API               │          │   │  │
            │           │  │  │  │  .NET 8 · 1vCPU/2GB       │   │  .NET 8 · 0.5vCPU/1GB   │          │   │  │
            │           │  │  │  │  auto-scale on ALB        │   │  1 task                  │          │   │  │
            │           │  │  │  │                           │   │                          │          │   │  │
            │           │  │  │  │  reads: Valkey → Aurora   │   │  reads/writes: Aurora    │          │   │  │
            │           │  │  │  │    → ClickHouse Gold      │   │  reads: Valkey           │          │   │  │
            │           │  │  │  │  (fallback chain — §5)    │   │  calls: external APIs    │          │   │  │
            │           │  │  │  └──────────────────────────┘   └──────────────────────────┘          │   │  │
            │           │  │  │                                                                        │   │  │
            │           │  │  └────────────────────────────────────────────────────────────────────────┘   │  │
            │           │  │                                                                                │  │
            │           │  │  ◄─────── WRITE PATH  (independent of read path — no shared arrows) ──────►   │  │
            │           │  │  ┌────────────────────────────────────────────────────────────────────────┐   │  │
            │           │  │  │                                                                        │   │  │
            │           │  │  │  ┌──────────────────────┐   ┌──────────────────────┐                  │   │  │
            │           │  │  │  │  Score Processor      │   │  PeerDB OSS          │                  │   │  │
            │           │  │  │  │  .NET 8 Fargate        │   │  Fargate · CDC Engine │                  │   │  │
            │           │  │  │  │  SQS consumer          │   │  Binary WAL reader   │                  │   │  │
            │           │  │  │  │  → UPSERT Aurora       │   │  → SQS cdc-events    │                  │   │  │
            │           │  │  │  └──────────────────────┘   │  → CH membership direct│                  │   │  │
            │           │  │  │                             └──────────┬───────────┘                  │   │  │
            │           │  │  │                                        │ SQS cdc-events               │   │  │
            │           │  │  │  ┌──────────────────────┐             ▼                              │   │  │
            │           │  │  │  │  Aggregate Refresh    │   ┌──────────────────────┐                  │   │  │
            │           │  │  │  │  .NET 8 · Fargate     │   │  Sign-Pair Transformer│                  │   │  │
            │           │  │  │  │  reads ClickHouse Gold │   │  .NET 8 Fargate       │                  │   │  │
            │           │  │  │  │  → writes report_cache │   │  INSERT  → sign=+1    │                  │   │  │
            │           │  │  │  │  15min/30min/nightly  │   │  UPDATE  → -1 then +1 │                  │   │  │
            │           │  │  │  └──────────────────────┘   │  DELETE  → sign=-1    │                  │   │  │
            │           │  │  │                             │  → Bronze INSERT CH   │                  │   │  │
            ▼           │  │  │  ┌──────────────────────┐   └──────────────────────┘                  │   │  │
  ┌──────────────────┐  │  │  │  │  RtsSyncWorker        │                                             │   │  │
  │  TRS RDS MSSQL   │──┼──┼──┼─▶│  .NET 8 · hourly     │                                             │   │  │
  │  SQL Server      │  │  │  │  │  EventBridge trigger  │                                             │   │  │
  │  db.t3.medium    │  │  │  │  │  → UPSERT membership  │                                             │   │  │
  └──────────────────┘  │  │  │  └──────────────────────┘                                             │   │  │
                        │  │  │                                                                        │   │  │
                        │  │  └────────────────────────────────────────────────────────────────────────┘   │  │
                        │  └────────────────────────────────────────────────────────────────────────────────┘  │
                        │               │                         │                       │
                        │  ISOLATED SUBNETS — no internet, no NAT                         │
                        │  ┌────────────▼─────────────────────────▼──────────────────┐   │
                        │  │                                                           │   │
                        │  │  ┌──────────────────────────────────────────────────┐    │   │
                        │  │  │  Aurora PostgreSQL Serverless v2  (sg-aurora)     │    │   │
                        │  │  │  aurora-postgresql16 · 0.5–16 ACU · Multi-AZ     │    │   │
                        │  │  │  ingress :5432 from sg-fargate only               │    │   │
                        │  │  │                                                   │    │   │
                        │  │  │  PRIMARY STORE — Source of Truth                  │    │   │
                        │  │  │  • student_opportunities  (partitioned)           │    │   │
                        │  │  │  • student_component_scores  (partitioned)        │    │   │
                        │  │  │  • roster_student / school_student                │    │   │
                        │  │  │  • district_student / district_school             │    │   │
                        │  │  │  • teacher_roster / students                      │    │   │
                        │  │  │  • tenants / test_aliases / test_keys             │    │   │
                        │  │  │  • test_alias_groups / embargo_roles              │    │   │
                        │  │  │  • report_cache  (pre-computed aggregates)        │    │   │
                        │  │  │  • mv_school_overall / mv_district_overall        │    │   │
                        │  │  │  • score_ingest_log  (idempotency)                │    │   │
                        │  │  │                                                   │    │   │
                        │  │  │            WAL (logical replication slot)         │    │   │
                        │  │  └──────────────────────────┬────────────────────────┘    │   │
                        │  │                             │                             │   │
                        │  │  ┌──────────────────────────▼────────────────────────┐    │   │
                        │  │  │  ElastiCache Valkey  (sg-valkey)                   │    │   │
                        │  │  │  cache.r7g.large · auth token via Secrets Manager  │    │   │
                        │  │  │  ingress :6379 from sg-fargate only                │    │   │
                        │  │  │                                                    │    │   │
                        │  │  │  TIER 0 — Cache Layer                               │    │   │
                        │  │  │  • school aggregates  (TTL 15min)                  │    │   │
                        │  │  │  • district aggregates  (TTL 5min)                 │    │   │
                        │  │  │  • test config  (TTL 10min · invalidated on Admin) │    │   │
                        │  │  │  • embargo status  (TTL 60s / 5min for roles)      │    │   │
                        │  │  └───────────────────────────────────────────────────┘    │   │
                        │  │                                                           │   │
                        │  │  ┌───────────────────────────────────────────────────┐    │   │
                        │  │  │  ClickHouse EC2  r6i.xlarge  (sg-clickhouse)       │    │   │
                        │  │  │  200GB gp3 · Amazon Linux 2023 · v24.x             │    │   │
                        │  │  │  ingress :8123/:9000 from sg-fargate only          │    │   │
                        │  │  │                                                    │    │   │
                        │  │  │  MEDALLION ARCHITECTURE — Aggregation Only         │    │   │
                        │  │  │                                                    │    │   │
                        │  │  │  [Bronze]                                          │    │   │
                        │  │  │  student_scores_bronze                             │    │   │
                        │  │  │  VersionedCollapsingMergeTree                      │    │   │
                        │  │  │  (raw CDC sign pairs)                              │    │   │
                        │  │  │         │ Silver MV (auto on insert)               │    │   │
                        │  │  │         ▼                                          │    │   │
                        │  │  │  [Silver]                                          │    │   │
                        │  │  │  student_scores_silver                             │    │   │
                        │  │  │  AggregatingMergeTree                              │    │   │
                        │  │  │  (latest per student)                              │    │   │
                        │  │  │         │ Gold MVs (auto on insert)                │    │   │
                        │  │  │         ▼                                          │    │   │
                        │  │  │  [Gold]                                            │    │   │
                        │  │  │  school_aggregates_gold                            │    │   │
                        │  │  │  district_aggregates_gold                          │    │   │
                        │  │  │  component_aggregates_gold                         │    │   │
                        │  │  │  AggregatingMergeTree                              │    │   │
                        │  │  │  (school/district cubes · avgMerge / countMerge)   │    │   │
                        │  │  │                                                    │    │   │
                        │  │  │  [Membership Mirrors]                              │    │   │
                        │  │  │  membership_school_mirror                          │    │   │
                        │  │  │  membership_district_mirror                        │    │   │
                        │  │  │  ReplacingMergeTree  (PeerDB direct write)         │    │   │
                        │  │  └───────────────────────────────────────────────────┘    │   │
                        │  └───────────────────────────────────────────────────────────┘   │
                        │                                                                   │
                        │  CROSS-CUTTING — all Fargate services                             │
                        │  ┌─────────────────────────────────────────────────────────────┐ │
                        │  │  Secrets Manager  (DB passwords · API keys · SSO JWKS URL)  │ │
                        │  │  SSM Parameter Store  (/trs/rts/last_sync_date)             │ │
                        │  │  CloudWatch  Logs · Metrics · Dashboards · Alarms           │ │
                        │  │  X-Ray  Distributed tracing · custom span attributes        │ │
                        │  └─────────────────────────────────────────────────────────────┘ │
                        └───────────────────────────────────────────────────────────────────┘

  SCORE INGEST PATH:
  ┌──────────────┐   PUT    ┌──────────────────┐  S3 Event  ┌───────────────────────────┐
  │ Score System │ ───────▶ │ S3 trs-scores-raw│ ─────────▶ │ SQS score-ingest          │
  └──────────────┘          │ SSE-KMS          │            │ visibility=300s  DLQ(x5)  │
                            └──────────────────┘            └─────────────┬─────────────┘
                                                                           │
                                                                           ▼ Score Processor

  EXTERNAL APIS (outbound from Admin API & TRS API via NAT Gateway):
  FlightPlan (test import)  ·  ITS (PLD/cut scores)  ·  CSR (standards)
  ReportsHub (ISR PDF)  ·  CRS (SDF Excel download)
```

---

## 2. VPC Network Topology & Security Groups

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  EXISTING VPC  (imported via ec2.Vpc.fromLookup — no CDK creation)           │
│  2 Availability Zones  ·  parameters locked in cdk.context.json              │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐    │
│  │  PUBLIC SUBNETS  (AZ-a + AZ-b)                                      │    │
│  │                                                                     │    │
│  │  ┌──────────────────────────────┐  ┌──────────────────────────────┐ │    │
│  │  │  ALB: TRS API                │  │  ALB: Admin API              │ │    │
│  │  │  sg-alb-trs                  │  │  sg-alb-admin                │ │    │
│  │  │  Ingress: :443 from          │  │  Ingress: :443 from          │ │    │
│  │  │    API GW VPC Link CIDR only │  │    API GW VPC Link CIDR only │ │    │
│  │  │  Egress:  :443 to Fargate SG │  │  Egress:  :443 to Fargate SG │ │    │
│  │  └──────────────┬───────────────┘  └──────────────┬───────────────┘ │    │
│  └─────────────────┼──────────────────────────────────┼────────────────┘    │
│                    │ :443                              │ :443                │
│  ┌─────────────────▼──────────────────────────────────▼────────────────┐    │
│  │  PRIVATE SUBNETS  (AZ-a + AZ-b)  —  NAT Gateway egress              │    │
│  │                                                                      │    │
│  │  sg-fargate-tasks                                                    │    │
│  │  Ingress: :443 from sg-alb-trs / sg-alb-admin                       │    │
│  │  Egress:  :5432 → sg-aurora                                          │    │
│  │           :8123 → sg-clickhouse                                      │    │
│  │           :9000 → sg-clickhouse  (native protocol)                   │    │
│  │           :6379 → sg-valkey                                          │    │
│  │           :1433 → sg-mssql                                           │    │
│  │           :443  → NAT → external APIs (FlightPlan, ITS, CSR, …)     │    │
│  │                                                                      │    │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐ ┌────────────┐  │    │
│  │  │ TRS API      │ │ Admin API    │ │ Score        │ │ PeerDB OSS │  │    │
│  │  │ Fargate      │ │ Fargate      │ │ Processor    │ │ Fargate    │  │    │
│  │  └──────────────┘ └──────────────┘ └──────────────┘ └────────────┘  │    │
│  │  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐                 │    │
│  │  │ Sign-Pair    │ │ RtsSyncWorker│ │ Aggregate    │                 │    │
│  │  │ Transformer  │ │ Fargate      │ │ Refresh      │                 │    │
│  │  └──────────────┘ └──────────────┘ └──────────────┘                 │    │
│  └──────────────────────────────────────────────────────────────────────┘    │
│                    │ :5432          │ :8123/:9000    │ :6379   │ :1433       │
│  ┌─────────────────▼──────┐ ┌──────▼─────────┐ ┌───▼──────┐ ┌▼──────────┐  │
│  │  ISOLATED SUBNETS  (AZ-a + AZ-b)  —  no internet, no NAT               │  │
│  │                        │         │               │         │           │  │
│  │  ┌─────────────────────┴──┐  ┌───┴────────────┐  │         │           │  │
│  │  │  Aurora PG             │  │ ClickHouse EC2 │  │         │           │  │
│  │  │  sg-aurora             │  │ sg-clickhouse  │  │         │           │  │
│  │  │  Ingress: :5432 from   │  │ Ingress: :8123 │  │         │           │  │
│  │  │    sg-fargate only     │  │   :9000 from   │  │         │           │  │
│  │  │  Multi-AZ writer +     │  │   sg-fargate   │  │         │           │  │
│  │  │    reader endpoint     │  │   only         │  │         │           │  │
│  │  └────────────────────────┘  └────────────────┘  │         │           │  │
│  │                                                   │         │           │  │
│  │  ┌────────────────────────────────────────────────┴─┐  ┌────┴────────┐  │  │
│  │  │  ElastiCache Valkey                              │  │ RDS MSSQL  │  │  │
│  │  │  sg-valkey                                       │  │ sg-mssql   │  │  │
│  │  │  Ingress: :6379 from sg-fargate only             │  │ Ingress:   │  │  │
│  │  │  Auth token via Secrets Manager                  │  │   :1433 from│  │  │
│  │  └──────────────────────────────────────────────────┘  │   sg-fargate│  │  │
│  │                                                         └────────────┘  │  │
│  └─────────────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────┘

  OUTSIDE VPC (regional AWS endpoints — no SG required):
  ┌────────────────────────────┐   ┌───────────────────────────────────────────┐
  │  API Gateway HTTP API      │   │  S3 · SQS · SNS · EventBridge             │
  │  trs-reporting-api         │   │  Secrets Manager · SSM · CloudWatch        │
  │  admin-api                 │   │  X-Ray · ECR · ACM · Route 53             │
  │  → VPC Link → ALB          │   │  (accessed via VPC endpoints or NAT GW)   │
  └────────────────────────────┘   └───────────────────────────────────────────┘
```

---

## 3. Score Ingestion Pipeline

```
  ┌──────────────────┐
  │  Score Delivery  │
  │  System          │
  └────────┬─────────┘
           │  PUT s3://trs-scores-raw/scores/<file>.json
           ▼
  ┌──────────────────────────────────────────────────────┐
  │  S3: trs-scores-raw                                  │
  │  versioning on · SSE-KMS · block public access       │
  └────────┬─────────────────────────────────────────────┘
           │  S3 Event Notification  (prefix filter: scores/)
           ▼
  ┌──────────────────────────────────────────────────────┐
  │  SQS: score-ingest                                   │
  │  visibility timeout = 300s                           │
  │  maxReceiveCount = 5  →  score-ingest-dlq on exceed  │
  └────────┬─────────────────────────────────────────────┘
           │  poll (BackgroundService)
           ▼
  ┌──────────────────────────────────────────────────────┐
  │  Score Processor  —  ECS Fargate (.NET 8)            │
  │                                                      │
  │  1 · PARSE + VALIDATE                                │
  │      JSON schema · required fields check             │
  │      OppStatus enum · ConditionCode enum             │
  │      UTF-8 encoding validation                       │
  │                  │ valid                             │
  │  2 · NORMALIZE                                       │
  │      TestKey → testalias_id  (via test_keys table)   │
  │      field mapping to Postgres column names          │
  │                  │                                   │
  │  3 · ELIGIBILITY FLAG                                │
  │      compute is_aggregate_eligible                   │
  │      OppStatus + ConditionCode rules                 │
  │                  │                                   │
  │  4 · IDEMPOTENCY CHECK                               │
  │      SELECT score_ingest_log WHERE s3_key + etag     │
  │      duplicate → SQS DeleteMessage (no-op)           │
  │      new file  → continue                            │
  │                  │                                   │
  │  5 · BATCH UPSERT                                    │
  │      student_opportunities                           │
  │      student_component_scores                        │
  │      ON CONFLICT WHERE EXCLUDED.date_scored > old    │
  │                  │                                   │
  │  REJECTION HANDLER  (any step can fork here)         │
  │  ┌───────────────────────────────────────────────┐   │
  │  │  UNKNOWN_TEST_CONFIG  → testalias_id not found │   │
  │  │  VALIDATION_FAILED    → schema / enum error    │   │
  │  │  DATA_CONFLICT        → newer score exists     │   │
  │  │  → write score_ingest_rejections               │   │
  │  │  → copy file to S3 trs-scores-rejected         │   │
  │  │  → SQS DeleteMessage (ACK)                     │   │
  │  └───────────────────────────────────────────────┘   │
  └────────┬─────────────────────────────────────────────┘
           │  successful UPSERT
           ▼
  ┌──────────────────────────────────────────────────────┐
  │  Aurora PostgreSQL                                   │
  │  student_opportunities  (partitioned)                │
  │  student_component_scores  (partitioned)             │
  │  REPLICA IDENTITY FULL  (for PeerDB CDC)             │
  └──────────────────────────────────────────────────────┘

  CloudWatch Custom Metrics (emitted by Score Processor):
  · TRS/ScoreProcessor/FilesProcessedPerHour
  · TRS/ScoreProcessor/RejectionCount  (by rejection type)
  · TRS/ScoreProcessor/BatchDurationP95
```

---

## 4. CDC Pipeline — Postgres to ClickHouse

```
  ┌──────────────────────────────────────────────────────┐
  │  Aurora PostgreSQL                                   │
  │  wal_level = logical                                 │
  │  max_replication_slots = 5                           │
  │  REPLICA IDENTITY FULL on score tables               │
  └──────────┬───────────────────────────────────────────┘
             │  WAL logical replication slot
             ▼
  ┌──────────────────────────────────────────────────────┐
  │  PeerDB OSS  —  ECS Fargate                          │
  │                                                      │
  │  CDC Job 1: student_opportunities                    │
  │             student_component_scores                 │
  │             → before + after row images              │
  │             → SQS cdc-events                         │
  │                          │                           │
  │  CDC Job 2: school_student                           │
  │             district_student                         │
  │             → ClickHouse membership mirrors DIRECT   │
  │               (bypasses Sign-Pair Transformer)       │
  └──────────┬───────────────────────────────────────────┘
             │
             ▼ CDC Job 1 output
  ┌──────────────────────────────────────────────────────┐
  │  SQS: cdc-events                                     │
  │  before + after row images per event                 │
  │  maxReceiveCount = 5  →  cdc-dlq                     │
  └──────────┬───────────────────────────────────────────┘
             │  poll (BackgroundService)
             ▼
  ┌──────────────────────────────────────────────────────┐
  │  Sign-Pair Transformer  —  ECS Fargate (.NET 8)      │
  │                                                      │
  │  INSERT event   →  emit sign=+1 Bronze row           │
  │                    single HTTP batch INSERT          │
  │                                                      │
  │  UPDATE / rescore  →  emit atomic  -1 / +1  pair     │
  │                       single HTTP batch INSERT       │
  │                       (both rows in one request)     │
  │                                                      │
  │  DELETE event   →  emit sign=-1 · is_deleted=1       │
  │                    DateTime.UtcNow as date_scored     │
  │                                                      │
  │  Error Handling:                                     │
  │    exp backoff  1s → 2s → 4s → 30s  on CH error     │
  │    4xx schema error → alarm TRS/CDC/SchemaError      │
  │    unrecoverable   → SQS cdc-dlq                     │
  └──────────┬───────────────────────────────────────────┘
             │  Bronze INSERT (batch HTTP :8123)
             ▼
  ┌──────────────────────────────────────────────────────┐
  │  ClickHouse EC2  r6i.xlarge  (Medallion Architecture)│
  │                                                      │
  │  [Bronze]                                            │
  │  student_scores_bronze                               │
  │  student_component_scores_bronze                     │
  │  VersionedCollapsingMergeTree                        │
  │  (raw CDC sign pairs — no reads happen here)         │
  │          │                                           │
  │          │  Silver Materialized View  (auto on insert)│
  │          ▼                                           │
  │  [Silver]                                            │
  │  student_scores_silver                               │
  │  AggregatingMergeTree                                │
  │  (latest per student — intermediate rollup)          │
  │          │                                           │
  │          │  Gold Materialized Views  (auto on insert) │
  │          ▼                                           │
  │  [Gold]                                              │
  │  school_aggregates_gold                              │
  │  district_aggregates_gold                            │
  │  component_aggregates_gold                           │
  │  AggregatingMergeTree                                │
  │  avgMerge / countMerge / stateMerge functions        │
  │  ← PRIMARY READ TARGET for TRS Reporting API         │
  │                                                      │
  │  [Membership Mirrors]  ← PeerDB CDC Job 2 direct     │
  │  membership_school_mirror   ReplacingMergeTree       │
  │  membership_district_mirror ReplacingMergeTree       │
  └──────────────────────────────────────────────────────┘

  Alarm:  CloudWatch  TRS/CDC/SchemaError
          → SNS trs-alerts → Slack / email  (P1 severity)
  DLQ monitor:  CloudWatch alarm on cdc-dlq depth > 0
```

---

## 5. TRS API Read Path & Fallback Chain

```
  Browser / Client
       │
       │  GET /v1/{tenantId}/school/{schoolId}/report
       │  Authorization: Bearer <JWT RS256>
       ▼
  ┌─────────────────────────────────┐
  │  API Gateway HTTP API           │
  │  trs-reporting-api              │
  └────────────────┬────────────────┘
                   │  VPC Link
                   ▼
  ┌─────────────────────────────────┐
  │  Internal ALB  (sg-alb-trs)     │
  │  HTTPS :443                     │
  └────────────────┬────────────────┘
                   │
                   ▼
  ┌─────────────────────────────────────────────────────────────┐
  │  TRS Reporting API  —  ECS Fargate (.NET 8)                  │
  │                                                             │
  │  1 · JWT VALIDATION                                         │
  │      RS256 · JWKS endpoint cached 60min                     │
  │      extract tenant_id + roles claims                       │
  │      401 on invalid / expired token                         │
  │                                                             │
  │  2 · EMBARGO CHECK  (Valkey-backed EmbargoService)          │
  │      embargo_until  TTL 60s                                 │
  │      embargo_roles  TTL 5min                                │
  │      → if embargoed for caller's role:                       │
  │        return 404 Not Found  (silent — test invisible)       │
  │                                                             │
  │  3 · FIVE-TIER FALLBACK CHAIN                               │
  │                                                             │
  │  ┌─────────────────────────────────────────────────────┐    │
  │  │  TIER 0 — Valkey (ElastiCache)                      │    │
  │  │  GET {scope}:{tenantId}:{scopeId}:{year}:{testId}   │    │
  │  │  HIT  → return 200  servedFrom: "valkey"            │    │
  │  │  MISS → proceed to Tier 1                           │    │
  │  └─────────────────────────────────────────────────────┘    │
  │                   │ MISS                                     │
  │  ┌────────────────▼────────────────────────────────────┐    │
  │  │  TIER 1 — report_cache  (Aurora PG table)           │    │
  │  │  SELECT * FROM report_cache WHERE scope=school …    │    │
  │  │  HIT  → backfill Valkey (TTL 15min)                 │    │
  │  │         return 200  servedFrom: "report_cache"      │    │
  │  │  MISS → proceed to Tier 2                           │    │
  │  └─────────────────────────────────────────────────────┘    │
  │                   │ MISS                                     │
  │  ┌────────────────▼────────────────────────────────────┐    │
  │  │  TIER 2 — ClickHouse Gold                           │    │
  │  │  GET /ping  (200ms timeout probe)                   │    │
  │  │  AVAILABLE:                                         │    │
  │  │    SELECT FROM school_aggregates_gold …             │    │
  │  │    avgMerge / countMerge functions                  │    │
  │  │    backfill Valkey  ·  return 200                   │    │
  │  │    servedFrom: "clickhouse_gold"                    │    │
  │  │  UNAVAILABLE / TIMEOUT:  proceed to Tier 3          │    │
  │  └─────────────────────────────────────────────────────┘    │
  │                   │ CH down                                  │
  │  ┌────────────────▼────────────────────────────────────┐    │
  │  │  TIER 3 — Postgres Materialized View                │    │
  │  │  SELECT * FROM mv_school_overall WHERE …            │    │
  │  │  AVAILABLE:  return 200  servedFrom: "postgres_mv"  │    │
  │  │  UNAVAILABLE / STALE: proceed to Tier 4             │    │
  │  └─────────────────────────────────────────────────────┘    │
  │                   │ MV stale                                 │
  │  ┌────────────────▼────────────────────────────────────┐    │
  │  │  TIER 4 — Postgres Live Query                       │    │
  │  │  SELECT aggregate(score_value) FROM                 │    │
  │  │    student_opportunities JOIN roster_student …      │    │
  │  │  return 200  servedFrom: "postgres_live"            │    │
  │  └─────────────────────────────────────────────────────┘    │
  │                                                             │
  │  Response envelope:  { data: {…}, servedFrom: "<tier>" }    │
  └─────────────────────────────────────────────────────────────┘

  BACKGROUND — Aggregate Refresh Job  (Fargate · EventBridge schedule)
  ┌──────────────────────────────────────────────────────┐
  │  Reads: ClickHouse Gold school/district/state cubes  │
  │  Writes: report_cache table (Aurora PG)              │
  │  Schedule:  school  15min  ·  district  5min         │
  │             state   nightly                          │
  └──────────────────────────────────────────────────────┘

  CloudWatch metric:  TRS/API/ClickHouseFallback
  (incremented every time Tier 2 is bypassed)
```

---

## 6. RTS Replication Pipeline

```
  ┌──────────────────────────────────────────────────────┐
  │  Master RTS MSSQL  (external — owned by RTS Team)    │
  │  SQL Server  —  Publisher                            │
  │  Tables: roster, school, district relationships      │
  └──────────────────────┬───────────────────────────────┘
                         │  SQL Server transactional replication
                         │  (relevant tables only)
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  TRS RDS SQL Server                                  │
  │  db.t3.medium  ·  isolated subnet  ·  sg-mssql       │
  │  Ingress :1433 from sg-fargate only                  │
  │  SQL Server  —  Subscriber                           │
  └──────────────────────┬───────────────────────────────┘
                         │  delta query  (hourly)
       EventBridge ──────┘  scheduled rule → trigger
       hourly rule
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  RtsSyncWorker  —  ECS Fargate (.NET 8)              │
  │                                                      │
  │  1 · DELTA QUERY                                     │
  │      SELECT * FROM rts.roster                        │
  │        WHERE modified_date > @last_sync_date         │
  │      first run (no last_sync_date) = full load       │
  │                                                      │
  │  2 · TRANSFORM                                       │
  │      RTS schema → TRS Postgres schema                │
  │      roster_student  ·  school_student               │
  │      district_student  ·  district_school            │
  │      teacher_roster                                  │
  │                                                      │
  │  3 · UPSERT Aurora                                   │
  │      ON CONFLICT DO UPDATE                           │
  │      soft-delete handling                            │
  │                                                      │
  │  4 · UPDATE SSM Parameter Store                      │
  │      /trs/rts/last_sync_date ← success timestamp    │
  │      CloudWatch alarm on failure or duration > 30min │
  └──────────────────────┬───────────────────────────────┘
                         │  UPSERT membership tables
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  Aurora PostgreSQL                                   │
  │  roster_student  ·  school_student                   │
  │  district_student  ·  district_school                │
  │  teacher_roster                                      │
  │  REPLICA IDENTITY FULL                               │
  └──────────────────────┬───────────────────────────────┘
                         │  WAL (logical replication)
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  PeerDB CDC Job 2  —  ECS Fargate                    │
  │  watches school_student + district_student           │
  │  WAL → ClickHouse membership mirrors DIRECT          │
  │  (bypasses Sign-Pair Transformer)                    │
  └──────────────────────┬───────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────┐
  │  ClickHouse                                          │
  │  membership_school_mirror   ReplacingMergeTree       │
  │  membership_district_mirror ReplacingMergeTree       │
  └──────────────────────────────────────────────────────┘

  Alarm:  trs-rts-sync-failure  →  SNS trs-alerts  →  Slack / email
  State:  SSM  /trs/rts/last_sync_date
```

---

## 7. Frontend Delivery & SSO Auth Flow

```
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  CI/CD — GitHub Actions                                                  │
  │                                                                          │
  │  on: push to main                                                        │
  │  ┌────────────────┐     ┌───────────────────┐     ┌────────────────────┐ │
  │  │  npm run build │────▶│  aws s3 sync       │────▶│  CloudFront        │ │
  │  │  Vite + React  │     │  → trs-spa-trs     │     │  create-invalidation│ │
  │  │  TypeScript    │     │    or trs-spa-admin │     │  cache bust        │ │
  │  │  Tailwind CSS  │     │  SSE-KMS upload    │     │  on each deploy    │ │
  │  └────────────────┘     └───────────────────┘     └────────────────────┘ │
  └──────────────────────────────────────────────────────────────────────────┘
                  │ deployed assets served via
                  ▼
  ┌──────────────────────────────┐   ┌──────────────────────────────┐
  │  CloudFront Distribution     │   │  CloudFront Distribution     │
  │  TRS SPA                     │   │  Admin SPA                   │
  │  *.trs.example.com           │   │  *.admin-trs.example.com     │
  │  ACM cert (DNS validated)    │   │  ACM cert (DNS validated)    │
  │  OAC signed requests to S3   │   │  OAC signed requests to S3   │
  └──────────┬───────────────────┘   └──────────┬───────────────────┘
             │ OAC                              │ OAC
             ▼                                  ▼
  ┌───────────────────────┐           ┌───────────────────────┐
  │  S3: trs-spa-trs      │           │  S3: trs-spa-admin    │
  │  SSE-KMS · versioning │           │  SSE-KMS · versioning │
  │  block public access  │           │  block public access  │
  └───────────────────────┘           └───────────────────────┘

  ─────────────────────────────────────────────────────────
  BROWSER AUTH + API CALL FLOW:
  ─────────────────────────────────────────────────────────

  Browser loads SPA from CloudFront
       │
       │  1 · OIDC login redirect
       ▼
  ┌──────────────────────────────────────────────────────┐
  │  SSO / OIDC Provider  (external)                     │
  │  RS256 JWT  ·  tenant_id claim  ·  roles claim array │
  │  JWKS endpoint (auto-rotation, cached 60min by API)  │
  └──────────────────────┬───────────────────────────────┘
                         │  2 · JWT RS256  returned to browser
                         ▼
  Browser  (JWT stored in memory only — NOT localStorage)
       │
       │  3 · API call with  Authorization: Bearer <JWT>
       ▼
  ┌──────────────────────────────┐   ┌──────────────────────────────┐
  │  API Gateway HTTP            │   │  API Gateway HTTP            │
  │  trs-reporting-api           │   │  admin-api                   │
  │  VPC Link → TRS ALB          │   │  VPC Link → Admin ALB        │
  └──────────┬───────────────────┘   └──────────┬───────────────────┘
             │                                  │
             ▼                                  ▼
  ┌──────────────────────┐           ┌──────────────────────┐
  │  TRS Reporting API   │           │  Admin API           │
  │  Fargate (.NET 8)    │           │  Fargate (.NET 8)    │
  │  validate JWT RS256  │           │  validate JWT RS256  │
  │  check roles claim   │           │  check admin role    │
  └──────────────────────┘           └──────────────────────┘
```

---

## 8. CDK Stack Deployment Order

```
  cdk deploy --all   (deploys in numbered order — dependencies shown below)
  Cross-stack references via SSM Parameter Store (no hard CF export coupling)

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  01-network-stack                                                        │
  │  VPC import  ec2.Vpc.fromLookup  (no VPC creation)                       │
  │  Custom Security Groups:                                                 │
  │    sg-fargate-tasks · sg-aurora · sg-clickhouse                          │
  │    sg-valkey · sg-mssql · sg-alb-trs · sg-alb-admin                     │
  │  Parameters → SSM: VPC ID, subnet IDs, SG IDs                           │
  └───────┬──────────────────────────────────┬──────────────────────────────┘
          │                                  │
          ▼                                  ▼
  ┌───────────────────────┐       ┌──────────────────────────────────────────┐
  │  02-security-stack    │       │  04-messaging-stack                      │
  │  IAM task roles (x6)  │       │  SQS score-ingest + DLQ                  │
  │  KMS CMKs             │       │  SQS cdc-events + DLQ                    │
  │    Aurora · S3        │       │  SQS rts-sync-trigger                    │
  │    Valkey · Secrets   │       │  SNS trs-alerts + trs-rescore-events     │
  │  Secrets Manager      │       │  EventBridge rules (15min/30min/nightly/ │
  │    path hierarchy     │       │    hourly)                               │
  └───────┬───────────────┘       └──────────────────────┬───────────────────┘
          │                                              │
          ▼                                              │
  ┌──────────────────────────────────────────┐           │
  │  03-data-stack                           │           │
  │  Aurora PG Serverless v2                 │           │
  │    aurora-postgresql16                   │           │
  │    0.5–16 ACU · Multi-AZ                 │           │
  │    param group: wal_level=logical        │           │
  │    max_replication_slots=5               │           │
  │  ClickHouse EC2  r6i.xlarge              │           │
  │    200GB gp3 · Amazon Linux 2023         │           │
  │    user-data: install ClickHouse 24.x    │           │
  │  ElastiCache Valkey  cache.r7g.large     │           │
  │  RDS SQL Server  db.t3.medium            │           │
  └───────┬──────────────────────────────────┘           │
          │                                              │
          ▼                                              │
  ┌──────────────────────────────────────────┐           │
  │  05-ecs-stack                            │           │
  │  ECS Fargate cluster                     │           │
  │  ECR repositories (x6):                  │           │
  │    trs-api · admin-api                   │           │
  │    score-processor · cdc-transformer     │           │
  │    rts-sync · aggregate-refresh          │           │
  └───────┬──────────────────────────────────┘           │
          │                          ◄────────────────────┘
          │   (03 + 04 + 05 all ready)
          ▼
  ┌─────────────────────┐  ┌─────────────────────┐  ┌─────────────────────┐
  │  06-api-stack       │  │  09-cdc-stack        │  │  10-rts-stack       │
  │  API GW HTTP (x2)   │  │  PeerDB Fargate      │  │  RDS MSSQL config   │
  │  VPC Links (x2)     │  │  Sign-Pair Transform │  │  RtsSyncWorker      │
  │  Internal ALBs (x2) │  │  Fargate             │  │  Fargate            │
  │  TRS API Fargate    │  │  ECR image refs      │  │  EventBridge rule   │
  │  Admin API Fargate  │  └─────────┬───────────┘  │  (hourly)           │
  │  Score Processor    │            │               └──────────┬──────────┘
  └─────────┬───────────┘            │                          │
            │                        └──────────────────────────┘
            │                                    │
            └────────────────────────────────────┘
                              │
                              ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  11-monitoring-stack                                                     │
  │  CloudWatch Log Groups  (30d dev · 90d staging/prod)                    │
  │  CloudWatch Dashboards (x4):                                             │
  │    Score Ingestion · CDC Pipeline · TRS API · ClickHouse EC2            │
  │  SLO Alarms:                                                             │
  │    CDC lag > 5min · cdc-dlq depth > 0 · 5xx rate > 1%                  │
  │    CH fallback rate > 10% · Aurora CPU > 80% · Valkey mem > 80%        │
  │  X-Ray tracing group + sampling rules                                   │
  │  SNS → Slack / email  (P1/P2 alarms)                                   │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  07-cdn-stack  (independent — no Fargate dependency)                    │
  │  CloudFront TRS SPA  ·  CloudFront Admin SPA                            │
  │  S3 SPA buckets  ·  S3 score buckets  ·  S3 backup bucket               │
  └──────────────────────┬───────────────────────────────────────────────────┘
                         │
                         ▼
  ┌──────────────────────────────────────────────────────────────────────────┐
  │  08-cicd-stack                                                           │
  │  GitHub Actions workflows                                                │
  │  on push → cdk synth  ·  on merge main → cdk deploy dev                 │
  │  manual approval gate for staging  ·  manual approval gate for prod     │
  │  React build → s3 sync → CloudFront invalidation                        │
  └──────────────────────────────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────────────────────────────┐
  │  02-security-stack  (also contains ACM + Route 53)                      │
  │  ACM certificates:  *.trs.example.com  ·  *.admin-trs.example.com      │
  │  Route 53 hosted zones + A/ALIAS records                                │
  └──────────────────────────────────────────────────────────────────────────┘
```

---

*Document version 1.0 · March 2026*
*Source of truth: `cdk/lib/stacks/` in the TRS monorepo · Cross-reference: `TRS_Project_Plan_WBS.md` Phase 1*
