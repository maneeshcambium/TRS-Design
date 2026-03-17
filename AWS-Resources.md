# TRS Platform — AWS Resources Summary

**Version:** 1.0  
**Date:** March 2026  
**Environments:** dev · staging · prod  
**CDK Stack source:** `cdk/lib/stacks/` (11 numbered stacks)

> All resources are defined in AWS CDK (TypeScript) and deployed via `cdk deploy --all`.  
> The VPC is pre-existing — it is imported, not created. All other resources below are CDK-managed.

---

## Contents

1. [Networking](#1-networking)
2. [Compute — ECS Fargate](#2-compute--ecs-fargate)
3. [Container Registry — ECR](#3-container-registry--ecr)
4. [Data — Aurora PostgreSQL](#4-data--aurora-postgresql)
5. [Data — ClickHouse EC2](#5-data--clickhouse-ec2)
6. [Data — ElastiCache Valkey](#6-data--elasticache-valkey)
7. [Data — RDS SQL Server](#7-data--rds-sql-server)
8. [Messaging — SQS](#8-messaging--sqs)
9. [Messaging — SNS](#9-messaging--sns)
10. [Messaging — EaaaaaaassssventBridge](#10-messaging--eventbridge)
11. [Storage — S3](#11-storage--s3)
12. [API & CDN](#12-api--cdn)
13. [Security & Identity](#13-security--identity)
14. [Observability](#14-observability)
15. [Resource Count Summary](#15-resource-count-summary)
16. [Estimated Monthly Cost (Indiana MVP)](#16-estimated-monthly-cost-indiana-mvp)

---

## 1. Networking

| Resource | Name / Detail | CDK Stack | Notes |
|---|---|---|---|
| **VPC** | Existing VPC — imported via `ec2.Vpc.fromLookup` | `01-network-stack` | **Not created by CDK.** Subnet IDs, AZ assignments, and route tables locked in `cdk.context.json` |
| **Public Subnets** | 2 × existing public subnets (AZ-a + AZ-b) | imported | ALBs placed here |
| **Private Subnets** | 2 × existing private subnets (AZ-a + AZ-b) | imported | Fargate tasks placed here; NAT Gateway egress |
| **Isolated Subnets** | 2 × existing isolated subnets (AZ-a + AZ-b) | imported | Aurora, ClickHouse EC2, Valkey, RDS MSSQL — no internet, no NAT |
| **NAT Gateway** | Existing — used by Fargate for outbound calls to external APIs | imported | Not created by CDK |
| **Security Group** | `sg-alb-trs` | `01-network-stack` | ALB for TRS API — ingress :443 from API GW VPC Link CIDR only |
| **Security Group** | `sg-alb-admin` | `01-network-stack` | ALB for Admin API — ingress :443 from API GW VPC Link CIDR only |
| **Security Group** | `sg-fargate-tasks` | `01-network-stack` | All Fargate tasks — ingress :443 from ALB SGs; egress to data tier SGs + NAT |
| **Security Group** | `sg-aurora` | `01-network-stack` | Aurora cluster — ingress :5432 from `sg-fargate-tasks` only |
| **Security Group** | `sg-clickhouse` | `01-network-stack` | ClickHouse EC2 — ingress :8123/:9000 from `sg-fargate-tasks` only |
| **Security Group** | `sg-valkey` | `01-network-stack` | ElastiCache Valkey — ingress :6379 from `sg-fargate-tasks` only |
| **Security Group** | `sg-mssql` | `01-network-stack` | RDS SQL Server — ingress :1433 from `sg-fargate-tasks` only |
| **ACM Certificate** | `*.trs.example.com` (DNS validated) | `02-security-stack` | TRS SPA + TRS API domains |
| **ACM Certificate** | `*.admin-trs.example.com` (DNS validated) | `02-security-stack` | Admin SPA + Admin API domains |
| **Route 53 Hosted Zone** | `trs.example.com` | `02-security-stack` | A/ALIAS records for TRS API + TRS SPA CloudFront |
| **Route 53 Hosted Zone** | `admin-trs.example.com` | `02-security-stack` | A/ALIAS records for Admin API + Admin SPA CloudFront |

---

## 2. Compute — ECS Fargate

| Service | Image | vCPU | Memory | Min Tasks | Max Tasks | CDK Stack | Trigger |
|---|---|---|---|---|---|---|---|
| **TRS Reporting API** | `trs-api:latest` | 1 | 2 GB | 2 | auto-scale | `06-api-stack` | ALB RequestCount |
| **Admin API** | `admin-api:latest` | 0.5 | 1 GB | 1 | 2 | `06-api-stack` | ALB RequestCount |
| **Score Processor** | `score-processor:latest` | 0.5 | 1 GB | 1 | auto-scale | `06-api-stack` | SQS `score-ingest` queue depth |
| **PeerDB OSS** | `peerdb:latest` (OSS image) | 0.25 | 0.5 GB | 1 | 1 | `09-cdc-stack` | Continuous — WAL reader |
| **Sign-Pair Transformer** | `cdc-transformer:latest` | 0.5 | 1 GB | 1 | 1 | `09-cdc-stack` | SQS `cdc-events` queue depth |
| **RtsSyncWorker** | `rts-sync:latest` | 0.25 | 0.5 GB | 0 | 1 | `10-rts-stack` | EventBridge scheduled — hourly |
| **Aggregate Refresh** | `aggregate-refresh:latest` | 0.5 | 1 GB | 0 | 1 | `06-api-stack` | EventBridge scheduled — 15min/30min/nightly |

**ECS Fargate Cluster:** `trs-platform` — shared across all services above. Defined in `05-ecs-stack`.

---

## 3. Container Registry — ECR

| Repository Name | Used By | CDK Stack |
|---|---|---|
| `trs-api` | TRS Reporting API Fargate service | `05-ecs-stack` |
| `admin-api` | Admin API Fargate service | `05-ecs-stack` |
| `score-processor` | Score Processor Fargate service | `05-ecs-stack` |
| `cdc-transformer` | Sign-Pair Transformer Fargate service | `05-ecs-stack` |
| `rts-sync` | RtsSyncWorker Fargate service | `05-ecs-stack` |
| `aggregate-refresh` | Aggregate Refresh Fargate service | `05-ecs-stack` |

All repositories: image scan on push enabled; lifecycle policy retains last 10 images.

---

## 4. Data — Aurora PostgreSQL

| Attribute | Value |
|---|---|
| **Resource** | Aurora PostgreSQL Serverless v2 Cluster |
| **Engine** | `aurora-postgresql16` |
| **Capacity** | min 0.5 ACU · max 16 ACU (auto-scales; Indiana MVP sizing) |
| **Multi-AZ** | Yes — writer + reader endpoint |
| **Subnet** | Isolated subnets |
| **Security Group** | `sg-aurora` — ingress :5432 from `sg-fargate-tasks` only |
| **Encryption** | KMS CMK `trs-aurora-key` (SSE at rest + in transit via SSL) |
| **Parameter Group** | Custom: `wal_level=logical`, `max_replication_slots=5`, `max_slot_wal_keep_size=10240` (10 GB) |
| **Backup** | Continuous automated point-in-time recovery (AWS managed) |
| **CDK Stack** | `03-data-stack` |
| **Credentials** | Stored in Secrets Manager — `/trs/{env}/aurora/password` |

**Purpose:** Primary and authoritative store for all TRS data — scores, membership, config, cache, idempotency log.

---

## 5. Data — ClickHouse EC2

| Attribute | Value |
|---|---|
| **Resource** | EC2 Instance |
| **Instance Type** | `r6i.xlarge` (32 GB RAM, 4 vCPU) — Indiana/small scale |
| **Storage** | 200 GB `gp3` EBS data volume |
| **OS/AMI** | Amazon Linux 2023 |
| **Software** | ClickHouse Server 24.x (installed via EC2 user-data script) |
| **Subnet** | Isolated subnet (AZ-a) |
| **Security Group** | `sg-clickhouse` — ingress :8123 (HTTP) / :9000 (native) from `sg-fargate-tasks` only |
| **Backup** | Daily S3 snapshot to `trs-ch-backups` (rebuildable from Aurora via PeerDB Initial Load) |
| **CDK Stack** | `03-data-stack` |
| **Credentials** | ClickHouse user/password in Secrets Manager — `/trs/{env}/clickhouse/password` |

**Purpose:** Aggregation-only analytical layer. Medallion Architecture: Bronze → Silver → Gold. Not used for per-student reads. Optional by design — system degrades gracefully without it.

**Scale-up path:**

| Scale | Students | Upgrade To |
|---|---|---|
| Indiana MVP | < 1.3M | `r6i.xlarge` (32 GB) |
| Medium (IL) | ~1.85M | `r6i.xlarge` (32 GB) |
| Large (TX) | ~5.5M | `r6i.2xlarge` (64 GB) |

---

## 6. Data — ElastiCache Valkey

| Attribute | Value |
|---|---|
| **Resource** | ElastiCache for Valkey |
| **Instance Type** | `cache.r7g.large` (predictable cost) — or ElastiCache Serverless |
| **Subnet** | Isolated subnets |
| **Security Group** | `sg-valkey` — ingress :6379 from `sg-fargate-tasks` only |
| **Encryption** | At-rest (KMS CMK) + in-transit (TLS) |
| **Auth** | Auth token stored in Secrets Manager — `/trs/{env}/valkey/auth-token` |
| **CDK Stack** | `03-data-stack` |

**Cache key namespaces:**

| Key Pattern | TTL | Invalidated By |
|---|---|---|
| `school:{tenantId}:{schoolId}:{year}:{testAliasId}` | 15 min | Aggregate Refresh Job |
| `district:{tenantId}:{districtId}:{year}:{testAliasId}` | 5 min | Aggregate Refresh Job |
| `testconfig:{tenantId}:{testAliasId}` | 10 min | Admin API on config change → SNS |
| `embargo:{tenantId}:{testAliasId}` | 60 s | Admin API on embargo change |
| `embargo_roles:{tenantId}:{testAliasId}` | 5 min | Admin API on embargo change |

**Purpose:** Tier-0 cache in the TRS API fallback chain. Also backs EmbargoService and test config lookups.

---

## 7. Data — RDS SQL Server

| Attribute | Value |
|---|---|
| **Resource** | RDS SQL Server |
| **Instance Type** | `db.t3.medium` |
| **License** | License-included (Express or Standard Edition) |
| **Subnet** | Isolated subnets |
| **Security Group** | `sg-mssql` — ingress :1433 from `sg-fargate-tasks` only |
| **Encryption** | KMS CMK at rest |
| **CDK Stack** | `03-data-stack` / `10-rts-stack` |
| **Credentials** | Secrets Manager — `/trs/{env}/mssql/password` |

**Purpose:** Local TRS replica of the external Master RTS MSSQL. Receives data via SQL Server transactional replication (master RTS = Publisher, this instance = Subscriber). RtsSyncWorker queries this replica hourly to delta-sync membership tables into Aurora.

---

## 8. Messaging — SQS

| Queue Name | Visibility Timeout | maxReceiveCount | DLQ | CDK Stack | Producer | Consumer |
|---|---|---|---|---|---|---|
| `score-ingest` | 300 s | 5 | `score-ingest-dlq` | `04-messaging-stack` | S3 Event Notification | Score Processor |
| `score-ingest-dlq` | 300 s | — | — | `04-messaging-stack` | `score-ingest` on exceed | CloudWatch alarm |
| `cdc-events` | 300 s | 5 | `cdc-dlq` | `04-messaging-stack` | PeerDB OSS | Sign-Pair Transformer |
| `cdc-dlq` | 300 s | — | — | `04-messaging-stack` | `cdc-events` on exceed | CloudWatch alarm |
| `rts-sync-trigger` | 60 s | 3 | — | `04-messaging-stack` | EventBridge (hourly) | RtsSyncWorker |

All queues: SSE-SQS encryption enabled; CloudWatch alarm on DLQ `ApproximateNumberOfMessagesVisible > 0`.

---

## 9. Messaging — SNS

| Topic Name | CDK Stack | Publishers | Subscribers |
|---|---|---|---|
| `trs-alerts` | `04-messaging-stack` | CloudWatch Alarms, CDC Transformer (schema errors) | Slack webhook / email (on-call) |
| `trs-rescore-events` | `04-messaging-stack` | Admin API (on config/embargo change) | Valkey cache invalidation listener (Fargate or Lambda) |

---

## 10. Messaging — EventBridge

| Rule Name | Schedule | CDK Stack | Target |
|---|---|---|---|
| `trs-school-mv-refresh` | Every 15 minutes | `04-messaging-stack` | Aggregate Refresh Fargate task |
| `trs-district-mv-refresh` | Every 30 minutes | `04-messaging-stack` | Aggregate Refresh Fargate task |
| `trs-state-nightly-refresh` | Daily at 02:00 UTC | `04-messaging-stack` | Aggregate Refresh Fargate task |
| `trs-rts-sync-hourly` | Every 60 minutes | `04-messaging-stack` | RtsSyncWorker Fargate task (via `rts-sync-trigger` SQS) |

---

## 11. Storage — S3

| Bucket Name | Purpose | Versioning | Encryption | CDK Stack |
|---|---|---|---|---|
| `trs-scores-raw` | Landing zone for inbound score JSON files from Scoring System | On | SSE-KMS (`trs-s3-key`) | `07-cdn-stack` |
| `trs-scores-rejected` | Rejected score files (UNKNOWN_TEST_CONFIG / VALIDATION_FAILED / DATA_CONFLICT) | On | SSE-KMS | `07-cdn-stack` |
| `trs-spa-trs` | TRS SPA static assets (React build output) | On | SSE-KMS | `07-cdn-stack` |
| `trs-spa-admin` | Admin SPA static assets (React build output) | On | SSE-KMS | `07-cdn-stack` |
| `trs-ch-backups` | Daily ClickHouse EC2 snapshots | On | SSE-KMS | `07-cdn-stack` |

All buckets: `BlockPublicAccess` fully enabled; S3 Object Ownership = BucketOwnerEnforced; access logging to a separate log bucket.

**S3 Event Notification:** `trs-scores-raw` → filter prefix `scores/` → SQS `score-ingest`.

---

## 12. API & CDN

### API Gateway

| Resource | Type | CDK Stack | Backend |
|---|---|---|---|
| `trs-reporting-api` | HTTP API | `06-api-stack` | VPC Link → `ALB: TRS API` → TRS Reporting API Fargate |
| `admin-api` | HTTP API | `06-api-stack` | VPC Link → `ALB: Admin API` → Admin API Fargate |

### Application Load Balancers (ALB)

| ALB Name | Scheme | Listener | Target Group | CDK Stack |
|---|---|---|---|---|
| `trs-api-alb` | Internal | HTTPS :443 | TRS Reporting API Fargate tasks | `06-api-stack` |
| `admin-api-alb` | Internal | HTTPS :443 | Admin API Fargate tasks | `06-api-stack` |

### CloudFront Distributions

| Distribution | Domain | Origin | OAC | CDK Stack |
|---|---|---|---|---|
| TRS SPA | `*.trs.example.com` | S3 `trs-spa-trs` | Yes — signed requests only | `07-cdn-stack` |
| Admin SPA | `*.admin-trs.example.com` | S3 `trs-spa-admin` | Yes — signed requests only | `07-cdn-stack` |

Both distributions: ACM certificate (DNS validated), managed cache policy for static assets, custom error response for SPA client-side routing (404 → `/index.html` with 200).

---

## 13. Security & Identity

### KMS Customer-Managed Keys (CMKs)

| Key Alias | Protects | CDK Stack |
|---|---|---|
| `alias/trs-aurora-key` | Aurora cluster storage + Secrets Manager entries | `02-security-stack` |
| `alias/trs-s3-key` | All S3 buckets (scores, SPA, backups) | `02-security-stack` |
| `alias/trs-valkey-key` | ElastiCache Valkey at-rest encryption | `02-security-stack` |
| `alias/trs-mssql-key` | RDS SQL Server storage encryption | `02-security-stack` |

### IAM Roles (ECS Task Roles — least-privilege)

| Role Name | Attached To | Key Permissions |
|---|---|---|
| `trs-api-task-role` | TRS Reporting API Fargate tasks | `secretsmanager:GetSecretValue` (own secrets), `xray:PutTraceSegments` |
| `admin-api-task-role` | Admin API Fargate tasks | `secretsmanager:GetSecretValue`, `sns:Publish` (rescore events), `xray:PutTraceSegments` |
| `score-processor-task-role` | Score Processor Fargate tasks | `sqs:ReceiveMessage/DeleteMessage/ChangeMessageVisibility` (score-ingest), `s3:GetObject` (trs-scores-raw), `s3:PutObject` (trs-scores-rejected), `secretsmanager:GetSecretValue` |
| `cdc-transformer-task-role` | Sign-Pair Transformer Fargate tasks | `sqs:ReceiveMessage/DeleteMessage/ChangeMessageVisibility` (cdc-events), `cloudwatch:PutMetricData`, `secretsmanager:GetSecretValue` |
| `rts-sync-task-role` | RtsSyncWorker Fargate tasks | `ssm:GetParameter/PutParameter` (/trs/rts/last_sync_date), `secretsmanager:GetSecretValue`, `cloudwatch:PutMetricData` |
| `aggregate-refresh-task-role` | Aggregate Refresh Fargate tasks | `secretsmanager:GetSecretValue`, `cloudwatch:PutMetricData` |
| `ecs-task-execution-role` | All Fargate tasks (execution role) | `ecr:GetAuthorizationToken`, `ecr:BatchGetImage`, `logs:CreateLogStream`, `logs:PutLogEvents`, `secretsmanager:GetSecretValue` |

### Secrets Manager

| Secret Path | Contents | Rotated? |
|---|---|---|
| `/trs/{env}/aurora/password` | Aurora master password | AWS managed rotation |
| `/trs/{env}/aurora/readonly-password` | Aurora read-only user password | AWS managed rotation |
| `/trs/{env}/clickhouse/password` | ClickHouse `trs_app` user password | Manual |
| `/trs/{env}/valkey/auth-token` | ElastiCache Valkey auth token | Manual |
| `/trs/{env}/mssql/password` | RDS SQL Server SA password | Manual |
| `/trs/{env}/sso/jwks-url` | SSO OIDC JWKS endpoint URL | n/a (URL, not secret) |
| `/trs/{env}/external/flightplan-api-key` | FlightPlan API key | Manual |
| `/trs/{env}/external/its-api-key` | ITS PLD API key | Manual |
| `/trs/{env}/external/csr-api-key` | CSR standards API key | Manual |
| `/trs/{env}/external/reportshub-api-key` | ReportsHub ISR PDF API key | Manual |
| `/trs/{env}/external/crs-api-key` | CRS SDF Excel API key | Manual |

### SSM Parameter Store

| Parameter Path | Contents | Used By |
|---|---|---|
| `/trs/{env}/rts/last_sync_date` | Timestamp of last successful RTS sync | RtsSyncWorker |
| `/trs/{env}/network/vpc-id` | VPC ID (cross-stack reference) | All stacks |
| `/trs/{env}/network/private-subnet-ids` | Comma-separated private subnet IDs | Fargate stacks |
| `/trs/{env}/network/isolated-subnet-ids` | Comma-separated isolated subnet IDs | Data stacks |
| `/trs/{env}/network/sg-fargate-tasks` | Security group ID | Fargate task definitions |
| `/trs/{env}/data/aurora-cluster-endpoint` | Aurora writer endpoint | API + Worker services |
| `/trs/{env}/data/aurora-reader-endpoint` | Aurora reader endpoint | TRS API (read queries) |
| `/trs/{env}/data/clickhouse-endpoint` | ClickHouse EC2 private IP / DNS | Fargate services |
| `/trs/{env}/data/valkey-endpoint` | ElastiCache Valkey endpoint | API services |

---

## 14. Observability

### CloudWatch Log Groups

| Log Group | Retention | CDK Stack |
|---|---|---|
| `/trs/fargate/trs-api` | 30d dev · 90d staging/prod | `11-monitoring-stack` |
| `/trs/fargate/admin-api` | 30d dev · 90d staging/prod | `11-monitoring-stack` |
| `/trs/fargate/score-processor` | 30d dev · 90d staging/prod | `11-monitoring-stack` |
| `/trs/fargate/cdc-transformer` | 30d dev · 90d staging/prod | `11-monitoring-stack` |
| `/trs/fargate/rts-sync` | 30d dev · 90d staging/prod | `11-monitoring-stack` |
| `/trs/fargate/aggregate-refresh` | 30d dev · 90d staging/prod | `11-monitoring-stack` |
| `/trs/fargate/peerdb` | 30d dev · 90d staging/prod | `11-monitoring-stack` |

### CloudWatch Alarms

| Alarm Name | Metric | Threshold | Severity | Action |
|---|---|---|---|---|
| `trs-cdc-lag` | PeerDB LSN lag | > 5 min | P1 | SNS → `trs-alerts` |
| `trs-cdc-dlq-depth` | `cdc-dlq` ApproximateNumberOfMessages | > 0 | P1 | SNS → `trs-alerts` |
| `trs-api-5xx-rate` | TRS API 5xx errors | > 1% | P1 | SNS → `trs-alerts` |
| `trs-ch-fallback-rate` | `TRS/API/ClickHouseFallback` custom metric | > 10% | P1 | SNS → `trs-alerts` |
| `trs-cdc-schema-error` | `TRS/CDC/SchemaError` custom metric | > 0 | P1 | SNS → `trs-alerts` |
| `trs-rts-sync-failure` | `TRS/RTS/SyncFailure` custom metric | > 0 | P2 | SNS → `trs-alerts` |
| `trs-rts-sync-duration` | `TRS/RTS/SyncDurationSeconds` | > 1800 s (30 min) | P2 | SNS → `trs-alerts` |
| `trs-score-dlq-depth` | `score-ingest-dlq` ApproximateNumberOfMessages | > 0 | P2 | SNS → `trs-alerts` |
| `trs-score-rejection-spike` | `TRS/ScoreProcessor/RejectionCount` | > 100/hr | P2 | SNS → `trs-alerts` |
| `trs-aurora-cpu` | Aurora CPUUtilization | > 80% | P2 | SNS → `trs-alerts` |
| `trs-valkey-memory` | ElastiCache DatabaseMemoryUsagePercentage | > 80% | P2 | SNS → `trs-alerts` |
| `trs-clickhouse-cpu` | EC2 CPUUtilization (ClickHouse instance) | > 75% | P2 | SNS → `trs-alerts` |

### CloudWatch Dashboards

| Dashboard Name | Key Widgets |
|---|---|
| `TRS-ScoreIngestion` | Files/hr, rejection count by type, batch duration p95, score-ingest queue depth |
| `TRS-CDCPipeline` | PeerDB LSN lag (minutes), Transformer queue depth, Bronze insert rate per table, DLQ depth |
| `TRS-ReportingAPI` | p50/p95/p99 latency by endpoint, fallback tier breakdown, active Fargate task count, Valkey hit ratio |
| `TRS-ClickHouseEC2` | Custom: `CH_BackgroundMergeQueue`, `CH_QueryLatencyP95` (emitted by CloudWatch agent cron) |

### X-Ray

| Resource | CDK Stack |
|---|---|
| X-Ray tracing group `trs-platform` | `11-monitoring-stack` |
| Sampling rules — all Fargate services + API Gateway | `11-monitoring-stack` |

---

## 15. Resource Count Summary

| Category | Resource Type | Count |
|---|---|---|
| Networking | Security Groups (custom, CDK-created) | 7 |
| Networking | ACM Certificates | 2 |
| Networking | Route 53 Hosted Zones | 2 |
| Compute | ECS Fargate Cluster | 1 |
| Compute | ECS Fargate Services | 7 |
| Compute | EC2 Instance (ClickHouse) | 1 |
| Container Registry | ECR Repositories | 6 |
| Data | Aurora PostgreSQL Cluster | 1 |
| Data | ElastiCache Valkey Node | 1 |
| Data | RDS SQL Server Instance | 1 |
| Messaging | SQS Queues (incl. DLQs) | 5 |
| Messaging | SNS Topics | 2 |
| Messaging | EventBridge Scheduled Rules | 4 |
| Storage | S3 Buckets | 5 |
| API | API Gateway HTTP APIs | 2 |
| API | Application Load Balancers | 2 |
| CDN | CloudFront Distributions | 2 |
| Security | KMS CMKs | 4 |
| Security | IAM Roles | 7 |
| Security | Secrets Manager Secrets | 10 |
| Observability | CloudWatch Log Groups | 7 |
| Observability | CloudWatch Alarms | 12 |
| Observability | CloudWatch Dashboards | 4 |
| **Total managed resources** | | **~107** |

---

## 16. Estimated Monthly Cost (Indiana MVP)

> All figures are approximate on-demand USD. Reserved pricing (1-yr) shown where applicable.  
> Region: `us-east-1`. Does not include data transfer or support plan.

| Resource | Specification | Est. $/month |
|---|---|---|
| Aurora PostgreSQL Serverless v2 | avg 2 ACU · 1 writer + 1 reader | ~$100 |
| ClickHouse EC2 `r6i.xlarge` | On-demand | ~$181 |
| ClickHouse EC2 `r6i.xlarge` | 1-yr Reserved | ~$105 |
| ClickHouse EBS 200GB gp3 | — | ~$16 |
| ElastiCache Valkey `cache.r7g.large` | On-demand | ~$104 |
| RDS SQL Server `db.t3.medium` | License-included | ~$80 |
| ECS Fargate (all 7 services) | avg 2 vCPU steady-state | ~$60 |
| API Gateway (HTTP) × 2 | 1M requests/mo | ~$2 |
| CloudFront × 2 | 10GB transfer/mo | ~$5 |
| S3 × 5 buckets | 50GB total | ~$2 |
| SQS + SNS + EventBridge | 1M msgs/mo | ~$2 |
| Secrets Manager × 10 | — | ~$4 |
| CloudWatch (logs, metrics, alarms) | 20GB logs/mo | ~$20 |
| ALB × 2 | — | ~$20 |
| **Total (on-demand ClickHouse)** | | **~$580/mo** |
| **Total (reserved ClickHouse)** | | **~$504/mo** |

> ⚠️ Aurora and ElastiCache scale with traffic — costs above assume Indiana MVP load (~100 concurrent users). Texas scale (5.5M students) would increase Aurora and ClickHouse costs significantly.

---

*Document version 1.0 · March 2026*  
*Cross-reference: [architecture.md](architecture.md) · [TRS_Project_Plan_WBS.md](TRS_Project_Plan_WBS.md) Phase 1*
