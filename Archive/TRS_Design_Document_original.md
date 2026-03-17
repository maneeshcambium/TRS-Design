# Teacher Reporting System (TRS) — Engineering Design Document v4

**Author:** Principal Software Architect
**Date:** 2026-03-03
**Status:** Draft — Supersedes v3 for implementation
**Audience:** Engineering team; AI implementation agents

> **Document purpose:** This is the authoritative design document for the TRS analytical
> reporting pipeline. Aurora PostgreSQL is the primary source of truth for all data. ClickHouse
> serves aggregation-only queries via a CDC-driven Medallion Architecture (Bronze → Silver → Gold).
> Postgres provides full query fallback at all times — ClickHouse is optional by design.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Infrastructure & Hosting Decisions](#3-infrastructure--hosting-decisions)
4. [CDC Pipeline — PeerDB + Sign-Pair Transformer](#4-cdc-pipeline--peerdb--sign-pair-transformer)
5. [Data Models — Inputs & Events](#5-data-models--inputs--events)
6. [Database Schema — Aurora PostgreSQL](#6-database-schema--aurora-postgresql)
7. [Database Schema — ClickHouse Medallion Architecture](#7-database-schema--clickhouse-medallion-architecture)
8. [Data Flow](#8-data-flow)
   - [8.5 Score File Rejection Path](#85-score-file-rejection-path)
9. [API Fallback Logic](#9-api-fallback-logic)
10. [Query Patterns](#10-query-patterns)
11. [Aggregate Cache & Materialized Views](#11-aggregate-cache--materialized-views)
12. [Resilience & Failure Modes](#12-resilience--failure-modes)
13. [Scale & Capacity](#13-scale--capacity)
14. [Cost Analysis](#14-cost-analysis)
15. [Key Engineering Decisions](#15-key-engineering-decisions)
16. [Operational Prerequisites](#16-operational-prerequisites)
17. [MVP Scope & Future Enhancements](#17-mvp-scope--future-enhancements)
18. [Tech Stack Reference](#18-tech-stack-reference)

---

## 1. System Overview

### 1.1 Purpose

The **Teacher Reporting System (TRS)** receives student test scores from an upstream Scoring
System and presents aggregated analytical reports to teachers and administrators. It is a
**read-heavy analytical system** with infrequent bursts of writes (during test windows) and heavy
aggregations over large student populations (up to 5.5 million students for Texas).

### 1.2 Architectural Philosophy

| Decision | Architecture |
|----------|--------------|
| Score storage | Aurora PostgreSQL — primary and authoritative store |
| ClickHouse writes | CDC-only via PeerDB + Sign-Pair Transformer (no direct write from application code) |
| ClickHouse schema | Medallion: Bronze → Silver → Gold (`VersionedCollapsingMergeTree` → `AggregatingMergeTree`) |
| Rescore in ClickHouse | Sign-Pair (-1/+1) atomic correction in Bronze; `avgState(score * sign)` in Gold |
| Score CDC transport | PeerDB on Fargate — WAL replication slot; before+after images → Sign-Pair Transformer |
| Membership CDC transport | PeerDB on Fargate — direct CDC from `school_student` / `district_student` → ClickHouse membership mirrors |
| ClickHouse sizing | Single `r6i.2xlarge` (aggregation-only, no replica) |
| Aggregate caching | Postgres `report_cache` table (keyed by scope/tenant/year/test) |
| Aggregate pre-computation | ClickHouse Gold layer live query → `report_cache` |
| Fallback when ClickHouse is down | Tier 1: Postgres `report_cache` → Tier 3: Postgres MV / live |
| Multi-enrollment attribution | Membership mirror `JOIN FINAL` (`ReplacingMergeTree` supports N rows per student) |

### 1.3 Core Inputs

| Input | Description | Source |
|-------|-------------|--------|
| Student test scores | JSON events per student opportunity | Upstream Scoring System |
| Roster / school / district relationships | Hierarchical membership data | External Relationship System (RTS) |
| Test configuration | Test family, subject, grade, standards metadata | Config systems |

### 1.4 Core Outputs

| Report | Aggregation Level | Latency Target |
|--------|-------------------|----------------|
| Class (roster) aggregates | All students in a teacher's roster | < 1 hour |
| Per-student score list | Individual student scores within a class | < 1 hour |
| Standard-level class aggregates | Per standard, within a class | < 1 hour |
| School aggregates | All students in a school | < 5 minutes (cached) |
| District aggregates | All students in a district | < 5 minutes (cached) |
| State aggregates | All students in a state | Nightly pre-computed |

### 1.5 Users & Concurrent Load

- **Teachers** — class-level (roster) reports.
- **School / District Administrators** — school and district aggregate reports.
- **MVP target concurrency:** 100 concurrent users per client; 1,000 per client per hour.

---

## 2. Architecture

### 2.1 High-Level Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────────────┐
│                                  TRS — Architecture v4                                        │
│                                                                                              │
│  ┌─────────────┐  ┌──────┐  ┌──────────────────┐  ┌──────────────────────────────────────┐  │
│  │  Scoring    │─▶│  S3  │─▶│  SQS Score Queue │─▶│  Lambda: Score Processor             │  │
│  │  System     │  │  Raw │  │  (+ DLQ)         │  │  Writes to Postgres ONLY             │  │
│  └─────────────┘  └──────┘  └──────────────────┘  └──────────────┬───────────────────────┘  │
│                                                                    │ UPSERT (primary write)   │
│  ┌─────────────┐  ┌──────────────────────┐                        │                          │
│  │  RTS        │─▶│  Lambda: RTS         │    ┌───────────────────▼───────────────────────┐  │
│  │  (external) │  │  Membership Sync     │───▶│                                           │  │
│  └─────────────┘  └──────────────────────┘    │  Aurora PostgreSQL Serverless v2          │  │
│                                               │  ─────────────────────────────────────── │  │
│                                               │  PRIMARY STORE — Source of Truth          │  │
│                                               │  • student_opportunities (scores)         │  │
│                                               │  • student_component_scores               │  │
│                                               │  • rosters / schools / districts          │  │
│                                               │  • student_attributes (demographics)      │  │
│                                               │  • report_cache (pre-computed aggregates) │  │
│                                               │  • mv_school_overall / mv_district_overall│  │
│                                               │  • score_ingest_log (idempotency)         │  │
│                                               └──────────┬────────────────────────────────┘  │
│                                                          │                                    │
│                                                    WAL (logical replication slot)             │
│                                                          │                                    │
│                                               ┌──────────▼────────────┐                      │
│                                               │  PeerDB Engine        │                      │
│                                               │  (Fargate 0.25vCPU)   │                      │
│                                               │  Binary WAL reader    │                      │
│                                               └──────────┬────────────┘                      │
│                                                          │ [score CDC] before+after images    │
│                                                          │ [membership CDC] direct→CH mirrors │
│                                                          │                                    │
│                                               ┌──────────▼────────────┐                      │
│                                               │  C# Sign-Pair         │                      │
│                                               │  Transformer          │                      │
│                                               │  (Fargate 0.5vCPU)    │                      │
│                                               │  INSERT → +1          │                      │
│                                               │  UPDATE → -1 then +1  │                      │
│                                               │  DELETE → -1 only     │                      │
│                                               └──────────┬────────────┘                      │
│                                                          │ Bronze INSERT (batch, HTTP)        │
│                                                          │                                    │
│  ┌──────────────────────────────────────────────────────▼─────────────────────────────────┐  │
│  │                  ClickHouse Single Node (r6i.2xlarge, 64GB RAM)                         │  │
│  │  ─────────────────────────────────────────────────────────────────────────────────────  │  │
│  │  MEDALLION ARCHITECTURE — Aggregation Only (no per-student reads, no replica)           │  │
│  │                                                                                          │  │
│  │  [Bronze]                    [Silver]                    [Gold]                          │  │
│  │  student_scores_bronze  ──▶  student_scores_silver  ──▶  school_aggregates_gold         │  │
│  │  VersionedCollapsingMT       AggregatingMT              AggregatingMT                   │  │
│  │  (raw CDC sign pairs)        (latest per student)        (school/district cubes)         │  │
│  │  sign=-1 (undo)                                          district_aggregates_gold        │  │
│  │  sign=+1 (redo)              membership_school_mirror     AggregatingMT                  │  │
│  │                              membership_district_mirror                                  │  │
│  │                              student_attributes                                          │  │
│  └──────────────────────────────────────────────────────────┬──────────────────────────────┘  │
│                                                              │                                 │
│                                  Aggregate Refresh Lambda (nightly + on-demand)               │
│                                  Reads Gold layer → writes to Postgres report_cache            │
│                                                              │                                 │
│  ┌─────────────┐  ┌────────────────────┐  ┌────────────────▼───────────────────────────────┐ │
│  │ CloudFront  │  │  API Gateway       │  │  Lambda: API                                    │ │
│  │ + React SPA │─▶│  + Lambda Auth     │─▶│  Three-tier fallback:                           │ │
│  └─────────────┘  │  (RH SSO JWT)      │  │  [1] Postgres report_cache          (<5ms)       │ │
│                   └────────────────────┘  │  [2] ClickHouse Gold live query     (<200ms)     │ │
│                                           │  [3] Postgres MV / live query       (fallback)   │ │
│                                           └────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Responsibilities

| Component | Technology | Responsibility |
|-----------|-----------|----------------|
| Scoring System | External | Produces score JSON files → S3 |
| S3 Raw | AWS S3 | Landing zone and permanent archive for raw score files |
| SQS Score Queue | AWS SQS (+ DLQ) | Decouples Scoring System from Score Processor; buffers bursts |
| Lambda: Score Processor | AWS Lambda (.NET 8 / C#) | Downloads S3 files; validates; UPSERTs to Aurora only; **no ClickHouse write** |
| RTS | External | Authoritative source of roster / school / district membership |
| Lambda: RTS Membership Sync | AWS Lambda (.NET 8 / C#) | Syncs RTS data into Aurora; best-effort mirror to ClickHouse membership tables |
| Aurora PostgreSQL Serverless v2 | AWS Aurora | **Primary store for ALL data**: scores, components, membership, config, report cache, idempotency log |
| PeerDB Engine | Fargate (0.25 vCPU / 0.5 GB) | Reads Aurora WAL via logical replication slot. **Two CDC jobs:** (1) score tables — delivers Before/After images to Sign-Pair Transformer; (2) membership tables (`school_student`, `district_student`) — replicates directly to ClickHouse `membership_school_mirror` / `membership_district_mirror` bypassing the Transformer |
| C# Sign-Pair Transformer | Fargate (0.5 vCPU / 1 GB) | Converts CDC events to `-1/+1` Bronze row pairs; batch-inserts to ClickHouse Bronze via HTTP |
| ClickHouse Single Node | EC2 `r6i.2xlarge` (aggregation-only) | Medallion Bronze/Silver/Gold layers; serves school/district aggregate queries; rebuilt from Postgres on total loss |
| Lambda: Aggregate Refresh | AWS Lambda (.NET 8 / C#) | Scheduled; reads ClickHouse Gold layer; writes results to Postgres `report_cache`; falls back to Postgres MV refresh if ClickHouse unavailable |
| Lambda: API | AWS Lambda (.NET 8 / C#) | REST API; three-tier fallback: `report_cache` → ClickHouse Gold live → Postgres MV/live |
| Auth | Red Hat SSO (OIDC) | External IdP — sole source of user identity and roles. Issues signed JWTs (`id_token`, RS256) with `tenant_id` and `roles` (string array) claims. Lambda authorizer validates signature against JWKS endpoint. TRS has no local users table. Embargo access is determined by matching JWT role names against `trs.embargo_roles` per tenant |

---

## 3. Infrastructure & Hosting Decisions

### 3.1 Why Aurora PostgreSQL is the Primary Store

| Reason | Detail |
|--------|--------|
| **Single source of truth** | Eliminates reconciliation problems if ClickHouse diverges or must be rebuilt. Postgres always holds the complete, authoritative dataset. |
| **Resilience** | Aurora is an AWS-managed service with built-in HA, automated failover, and Multi-AZ replication. ClickHouse on EC2 requires manual HA setup. |
| **Transactional correctness** | Rescores handled with a single atomic `UPSERT ... WHERE EXCLUDED.date_scored > existing`. No eventual-consistency window at the source-of-truth layer. |
| **Fallback capability** | ClickHouse becomes optional. Postgres always holds the complete dataset for any query. |
| **Operational simplicity** | No need to maintain separate EC2 node sizing, ClickHouse Keeper quorum, or EC2 replication. |

### 3.2 Why ClickHouse is Retained (Aggregation-Only)

| Reason | Detail |
|--------|--------|
| **Aggregate speed** | For large scopes (50k–300k students), ClickHouse `GROUP BY` is 10–100× faster than Postgres. School queries: <30ms vs. 200–800ms in Postgres. |
| **Columnar scans** | Score data reads only queried columns; Postgres reads full rows. |
| **Optional by design** | System degrades gracefully, not catastrophically, when ClickHouse is unavailable. |
| **Gold layer pre-aggregation** | `AggregatingMergeTree` with `avgState`/`uniqState`/`sumState` provides sub-10ms dashboard queries without on-the-fly `GROUP BY` once the MV is populated. |

### 3.3 Single ClickHouse Instance

Because Postgres is the source of truth:
- A ClickHouse failure is a **reporting performance degradation**, not a data loss event.
- The API fallback (§9) handles it automatically and transparently.
- ClickHouse can be fully rebuilt from Postgres at any time (`PeerDB Initial Load` snapshot).
- No ClickHouse replica node is needed or justified.

**Instance choice:** `r6i.2xlarge` (64 GB RAM, 8 vCPU).
- Sufficient for aggregation-only queries over ~2 GB compressed score data (TX scale).
- Silver layer (per-student reads) is served by Postgres — ClickHouse does not need to support that workload.
- Use a 1-Year Reserved instance to reduce cost by ~42%.

### 3.4 Why PeerDB (Not Debezium + MSK)

| Factor | Debezium + MSK | PeerDB (Chosen) |
|--------|---------------|-----------------|
| Infrastructure | Aurora + MSK cluster + Fargate + ClickHouse | Aurora + Fargate + ClickHouse |
| Monthly cost (CDC layer, TX scale) | ~$586/month (MSK) | ~$27/month |
| Durability buffer | Kafka 7-day topic retention | Postgres WAL replication slot (holds WAL until PeerDB consumes) |
| Replayability | Full Kafka offset rewind | Re-snapshot via PeerDB Initial Load |
| Restart recovery | Consumer group offset check | PeerDB resumes from last confirmed LSN automatically |
| Complexity | High — Kafka consumer, schema registry, Connect cluster | Low — single Fargate task, no broker |

**Why the WAL replication slot is sufficient buffering:** PeerDB holds an open replication slot on Postgres. If ClickHouse is briefly unavailable, PeerDB simply pauses consumption and the WAL accumulates on the Postgres side — no data loss. Set `max_slot_wal_keep_size = 10 GB` as a safety cap.

### 3.5 Aurora PostgreSQL Sizing

| Client Scale | Students | Aurora Config | Notes |
|---|---|---|---|
| MVP / single client | < 500k | Serverless v2 (min 0.5 ACU, max 16 ACU) | Auto-scales; ~$50–100/mo |
| Small (VA, IN) | ~1.0–1.3M | Serverless v2 (min 0.5 ACU, max 16 ACU) | ~$80–150/mo |
| Medium (IL) | ~1.85M | Serverless v2 (min 2, max 32 ACU) + 1 read replica | Read replica for report queries |
| Large (TX) | ~5.5M | Serverless v2 (min 4, max 64 ACU) + 1 read replica | Read replica handles all report queries |

### 3.6 ClickHouse EC2 Sizing

| Client Scale | Students | Instance | Monthly On-Demand | Monthly Reserved (1-yr) |
|---|---|---|---|---|
| MVP / Small (up to ~1.3M) | < 1.3M | `r6i.xlarge` (32 GB RAM) | ~$181 | ~$105 |
| Medium (IL, ~1.85M) | ~1.85M | `r6i.xlarge` (32 GB RAM) | ~$181 | ~$105 |
| Large (TX, ~5.5M) | ~5.5M | `r6i.2xlarge` (64 GB RAM) | ~$362 | ~$210 |

**EBS:** 200 GB `gp3` data volume at ~$16/month (all scales; ClickHouse compresses score data heavily).

### 3.7 Backup & Recovery

**Aurora:** Continuous automated backups to S3 (point-in-time recovery to any second within the retention window). Multi-AZ standby for sub-minute failover. No additional backup Lambda needed.

**ClickHouse:** Daily S3 snapshot via Lambda cron (optional — ClickHouse data is fully rebuildable from Postgres via PeerDB Initial Load). Recovery path: provision new EC2 → apply Medallion schema → PeerDB Initial Load → switch to streaming mode.

---

## 4. CDC Pipeline — PeerDB + Sign-Pair Transformer

### 4.1 Why the Sign-Pair Transformer is Necessary

ClickHouse `VersionedCollapsingMergeTree` and `AggregatingMergeTree` are append-only engines. They cannot handle a simple UPDATE event by subtracting the old value and adding the new. A naive stream of only "new values" leads to silent double-counting in school and district aggregates on every rescore.

The Sign-Pair Transformer solves this by converting the Before/After images from the WAL into two rows:

| Row | Action |
|-----|--------|
| **Undo row** | Old score value, `sign = -1`, `date_scored = T_before` |
| **Redo row** | New score value, `sign = +1`, `date_scored = T_after` |

Both rows are inserted in a **single HTTP batch** to ClickHouse. `VersionedCollapsingMergeTree` collapses the pair during background merges. Gold layer `avgState(score_value * sign)` corrects the running aggregate atomically.

### 4.2 PeerDB Event Contract

PeerDB delivers structured CDC events via SQS (or a direct HTTP endpoint), each containing:

```json
{
  "EventType": "UPDATE",
  "TableName": "trs.student_opportunities",
  "Timestamp": "2026-03-03T14:22:05.123Z",
  "Lsn": "0/1F4A2B8",
  "BeforeImage": {
    "opp_key": "550e8400-...",
    "tenant_id": "tx",
    "school_year": 2026,
    "test_group_id": "cp1-g5-ela-2026",
    "student_id": 12345,
    "score_value": 2400.0,
    "overall_perf_level": 2,
    "date_scored": "2025-10-16T08:23:00.000Z",
    "is_aggregate_eligible": 1,
    "is_deleted": 0
  },
  "AfterImage": {
    "opp_key": "550e8400-...",
    "tenant_id": "tx",
    "school_year": 2026,
    "test_group_id": "cp1-g5-ela-2026",
    "student_id": 12345,
    "score_value": 2500.0,
    "overall_perf_level": 3,
    "date_scored": "2025-10-17T09:01:00.000Z",
    "is_aggregate_eligible": 1,
    "is_deleted": 0
  }
}
```

> **Critical prerequisite:** `REPLICA IDENTITY FULL` must be set on all replicated Aurora tables.
> Without it, DELETE and UPDATE Before images contain only primary key columns — all other fields
> are null, making the undo row useless for aggregate subtraction. See §16.

### 4.3 Sign-Pair Emission Logic

#### INSERT Event

```
Input:  BeforeImage = null, AfterImage = A

Output: One Bronze row:
  (opp_key=A.OppKey, sign=+1, is_deleted=0, date_scored=A.DateScored, ...A fields)
```

#### UPDATE Event (Rescore)

```
Input:  BeforeImage = B (old score), AfterImage = A (new score)
        Validate: A.DateScored > B.DateScored
        If out-of-order: log warning; emit pair anyway (VersionedCollapsing handles order)

Output: Two Bronze rows in a SINGLE HTTP batch INSERT:
  Row 1 (Undo): (opp_key=B.OppKey, sign=-1, is_deleted=0, date_scored=B.DateScored, ...B fields)
  Row 2 (Redo): (opp_key=A.OppKey, sign=+1, is_deleted=0, date_scored=A.DateScored, ...A fields)
```

**Critical:** Both rows must be submitted in a single atomic HTTP batch to ClickHouse. Splitting them creates a window where aggregates are temporarily incorrect.

#### DELETE Event

```
Input:  BeforeImage = B (the row being deleted), AfterImage = null

Output: One Bronze row (Undo only — no Redo counterpart):
  (opp_key=B.OppKey, sign=-1, is_deleted=1, date_scored=T_now, ...B fields)
```

`T_now` is `DateTime.UtcNow` with millisecond precision — a synthetic "last known event time" ensuring the delete wins in `argMaxState(is_deleted, date_scored)` logic in the Silver layer.

#### DELETE → Re-Insert Sequence

Idempotent by design:

```
T1: INSERT  → sign=+1, is_deleted=0, date_scored=T1
T2: DELETE  → sign=-1, is_deleted=1, date_scored=T2  (T2 > T1)
T3: Re-INSERT→ sign=+1, is_deleted=0, date_scored=T3  (T3 > T2)

Silver argMaxState(is_deleted, date_scored) → returns 0 at T3 (row re-visible)
Gold-layer aggregates gain the score back via the +1 sign at T3
```

### 4.4 CDC Event Type Reference

| CDC Event | Before | After | Signs Emitted | `is_deleted` | Bronze Result | Silver Result | Gold Result |
|---|:---:|:---:|:---:|:---:|---|---|---|
| **INSERT** | ❌ | ✅ | `+1` | `0` | Row added | `argMax` picks new row | Score added to avg / PL |
| **UPDATE (rescore)** | ✅ | ✅ | `-1` then `+1` | `0` | Pair collapses → net new score | `argMax` picks T_after row | Old subtracted, new added |
| **DELETE** | ✅ | ❌ | `-1` only | `1` | Collapses original `+1` | `is_deleted=1` wins | Score subtracted |
| **DELETE → re-INSERT** | ✅→✅ | ❌→✅ | `-1` then `+1` | `1` then `0` | Net zero then re-added | `is_deleted=0` wins at T3 | Score re-added |

### 4.5 Transformer Deployment

| Parameter | Value |
|-----------|-------|
| Container image | `trs/cdc-transformer:latest` (.NET 8 `BackgroundService`) |
| Fargate task size | 0.5 vCPU / 1 GB RAM |
| Scaling | Single task (stateless w.r.t. ClickHouse; PeerDB handles ordering and LSN checkpointing) |
| Input | SQS queue fed by PeerDB webhook events (or PeerDB direct HTTP push) |
| Output | ClickHouse HTTP interface — batch `INSERT INTO trs.student_scores_bronze FORMAT RowBinary` |
| Retry on ClickHouse error | Exponential back-off (1 s → 2 s → 4 s → max 30 s); do NOT advance LSN until INSERT succeeds |
| Fatal error (4xx) | Emit CloudWatch alarm `TRS/CDC/SchemaError`; do not retry; human review required |
| Bad events | Dead-letter to SQS DLQ; advance LSN past the malformed event |

---

## 5. Data Models — Inputs & Events

### 5.1 Test Identification

| Field | Description | Example |
|-------|-------------|---------|
| `TestKey` | Unique identifier for a specific test administration | `checkpoint1-ela-grade1-2025` |
| `TestGroupId` | Composite: `{TestFamily}-{Subject}-{Grade}-{Year}-{Attempt}` | `checkpoint1-ela-grade1-2025-attempt1` |
| `TestFamily` | Group of related tests | `CheckPoint1` |
| `Subject` | Subject area | `ELA`, `Math`, `Science` |
| `Grade` | Grade level | `Grade1`, `Grade2` |
| `Attempt` | Opportunity number | `Attempt1`, `Attempt2` |
| `SchoolYear` | Academic year | `2025`, `2026` |

**TestKey vs TestGroupId:** In 99%+ of cases these are identical. In rare multi-variant cases (online + paper), `TestKey` adds a variant suffix while `TestGroupId` groups all variants.

### 5.2 Incoming Score Event (JSON)

```json
{
  "OppKey": "550e8400-e29b-41d4-a716-446655440000",
  "StudentId": 12345,
  "TestKey": "checkpoint1-grade5-ela-2026",
  "TestEvent": "fall-2026",
  "SchoolYear": "2026",
  "TestedGrade": "Grade5",
  "OppStatus": "scored",
  "ConditionCode": null,
  "TestedDate": "2025-10-15",
  "DateScored": "2025-10-16T08:23:00.000Z",
  "OverallScaleScore": 2500,
  "OverallRawScore": 42,
  "OverallPerformanceLevel": 3,
  "StandardError": 2.1,
  "StandardScores": {
    "MA.912.GR.1.1": { "PerformanceLevel": 1, "StandardError": 1.5 }
  },
  "ReportingCategoryScores": {
    "geometry": { "ScaleScore": 1001, "PerformanceLevel": 5 }
  }
}
```

### 5.3 Rescore vs. Retest

| Event | `OppKey` | `DateScored` | Postgres behavior | ClickHouse behavior |
|-------|----------|--------------|------------------|---------------------|
| Original score | `uuid-A` | T1 | INSERT new row | CDC INSERT: Bronze `sign=+1` |
| **Rescore** — same sitting | `uuid-A` | T2 > T1 | ON CONFLICT → UPDATE (T2 wins) | CDC UPDATE: Bronze `-1@T1` + `+1@T2`; Gold corrected atomically |
| **Retest** — student retakes | `uuid-B` (new) | T3 | INSERT new row | CDC INSERT: Bronze `sign=+1` |
| Duplicate delivery | `uuid-A` | T1 | ON CONFLICT: WHERE fails; no-op → no CDC emitted | No Bronze insert (Postgres no-op → no WAL event) |
| Stale resend | `uuid-A` | T0 < T1 | ON CONFLICT: WHERE fails; no-op → no CDC emitted | No Bronze insert |
| Data conflict (same key + time, different data) | `uuid-A` | T1 | Log alert; no-op | No Bronze insert |

### 5.4 Aggregate Eligibility

Computed at ingest time by Score Processor Lambda. Stored in Postgres. Propagated to ClickHouse via CDC.

```
is_aggregate_eligible = TRUE  when:
  OppStatus IN ('scored', 'partially_scored')
  AND (ConditionCode IS NULL OR ConditionCode IN ('', 'none'))

is_aggregate_eligible = FALSE  when:
  OppStatus = 'notscored'
  OR ConditionCode IN ('Invalidated', 'Expired', 'Absent', ...)
```

On rescore, the Lambda recomputes the flag. If the flag changes, the Aurora UPDATE triggers a CDC event → Sign-Pair Transformer → Bronze insert with the corrected `is_aggregate_eligible` value. Gold-layer `WHERE is_aggregate_eligible = 1` propagates the update automatically.

### 5.5 Relationship Data (from RTS)

```
state → district → school → teacher → roster → student
```

TRS uses only **current** relationships (no effective-dated history). Membership data is refreshed from RTS; demographic data is stored in `student_attributes`.

---

## 6. Database Schema — Aurora PostgreSQL

Aurora is the **primary and authoritative store** for all TRS data.

### 6.1 `trs.student_opportunities` — Score Table

Two-level composite partitioning: Level 1 = `LIST(tenant_id)`, Level 2 = `LIST(school_year)`.

```sql
-- Application code ALWAYS queries the parent table only.
-- Postgres routes transparently to the correct leaf partition at plan time.
-- Never reference partition names (e.g. student_opportunities_tx_2026) in application code.

CREATE TABLE trs.student_opportunities (
    tenant_id              TEXT         NOT NULL,   -- L1 partition key
    school_year            SMALLINT     NOT NULL,   -- L2 partition key
    opp_key                UUID         NOT NULL,

    test_group_id          TEXT         NOT NULL,
    test_key               TEXT         NOT NULL,
    test_event             TEXT,
    student_id             INTEGER      NOT NULL,
    opp_status             TEXT         NOT NULL,
    condition_code         TEXT,
    tested_date            DATE,
    date_scored            TIMESTAMPTZ  NOT NULL,   -- version key; must be millisecond precision

    is_aggregate_eligible  BOOLEAN      NOT NULL DEFAULT FALSE,

    overall_scale_score    REAL,
    overall_raw_score      INTEGER,
    overall_lexile_score   INTEGER,
    overall_quantile_score INTEGER,
    overall_perf_level     SMALLINT,
    overall_standard_error REAL,

    ingested_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    raw_s3_key             TEXT,

    -- PK must include both partition keys for unique constraint on partitioned table
    PRIMARY KEY (tenant_id, school_year, opp_key)
)
PARTITION BY LIST (tenant_id);

-- Per-tenant partition (example: Texas)
CREATE TABLE trs.student_opportunities_tx
    PARTITION OF trs.student_opportunities
    FOR VALUES IN ('tx')
    PARTITION BY LIST (school_year);

CREATE TABLE trs.student_opportunities_tx_2026
    PARTITION OF trs.student_opportunities_tx
    FOR VALUES IN (2026);
```

**Indexes:**
```sql
-- Primary query pattern: tenant + year + test + student_id list (roster queries)
CREATE INDEX idx_so_roster
    ON trs.student_opportunities (tenant_id, school_year, test_group_id, student_id)
    WHERE is_aggregate_eligible = TRUE;

-- Full per-student score list (includes ineligible)
CREATE INDEX idx_so_student
    ON trs.student_opportunities (tenant_id, school_year, test_group_id, student_id);
```

**Rescore UPSERT:**
```sql
INSERT INTO trs.student_opportunities
    (tenant_id, school_year, opp_key, test_group_id, test_key, test_event,
     student_id, opp_status, condition_code, tested_date, date_scored,
     is_aggregate_eligible, overall_scale_score, overall_raw_score,
     overall_perf_level, overall_standard_error, raw_s3_key)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17)
ON CONFLICT (tenant_id, school_year, opp_key) DO UPDATE SET
    date_scored            = EXCLUDED.date_scored,
    opp_status             = EXCLUDED.opp_status,
    condition_code         = EXCLUDED.condition_code,
    tested_date            = EXCLUDED.tested_date,
    is_aggregate_eligible  = EXCLUDED.is_aggregate_eligible,
    overall_scale_score    = EXCLUDED.overall_scale_score,
    overall_raw_score      = EXCLUDED.overall_raw_score,
    overall_perf_level     = EXCLUDED.overall_perf_level,
    overall_standard_error = EXCLUDED.overall_standard_error,
    ingested_at            = NOW()
WHERE EXCLUDED.date_scored > trs.student_opportunities.date_scored;
-- WHERE clause: stale resends are silently ignored; zero PG rows affected → no WAL UPDATE event → no CDC
```

**PeerDB configuration note:** Set `REPLICA IDENTITY FULL` on this table (see §16) to ensure Before images contain all columns, not just the PK.

### 6.2 `trs.student_component_scores`

```sql
CREATE TABLE trs.student_component_scores (
    tenant_id             TEXT         NOT NULL,
    school_year           SMALLINT     NOT NULL,
    opp_key               UUID         NOT NULL,
    component_type        TEXT         NOT NULL,  -- STANDARD | RC | WRITING_DIM
    component_id          TEXT         NOT NULL,

    test_group_id         TEXT         NOT NULL,
    student_id            INTEGER      NOT NULL,
    perf_level            SMALLINT,
    scale_score           REAL,
    standard_error        REAL,
    condition_code        TEXT,
    is_aggregate_eligible BOOLEAN      NOT NULL DEFAULT FALSE,
    date_scored           TIMESTAMPTZ  NOT NULL,

    PRIMARY KEY (tenant_id, school_year, opp_key, component_type, component_id)
)
PARTITION BY LIST (tenant_id);

CREATE INDEX idx_scs_query
    ON trs.student_component_scores
       (tenant_id, school_year, test_group_id, student_id, component_type)
    WHERE is_aggregate_eligible = TRUE;
```

### 6.3 Membership & Config Tables

| Table | Purpose |
|-------|---------|
| `test_configs` | Test family, subject, grade, standards metadata; embargo configuration |
| `embargo_roles` | Per-tenant roles that grant embargo visibility (matched against JWT claims) |
| `roster_student` | RTS mirror — current roster ↔ student membership |
| `school_student` | RTS mirror — current school ↔ student membership |
| `district_student` | RTS mirror — current district ↔ student membership |
| `district_school` | RTS mirror — district ↔ school relationships |
| `teacher_roster` | RTS mirror — teacher ↔ roster assignments |

#### `trs.test_configs`

```sql
CREATE TABLE trs.test_configs (
    tenant_id       TEXT        NOT NULL,
    test_key        TEXT        NOT NULL,
    test_group_id   TEXT        NOT NULL,
    test_family     TEXT        NOT NULL,
    subject         TEXT        NOT NULL,
    grade           TEXT        NOT NULL,
    school_year     SMALLINT    NOT NULL,
    metadata        JSONB,                    -- standards map, reporting categories, etc.

    -- Embargo: NULL = not embargoed (visible to all users).
    -- Future timestamp = embargoed until that moment (auto-lifts when clock passes it).
    -- Setting to a past timestamp is equivalent to lifting the embargo immediately.
    embargo_until   TIMESTAMPTZ NULL DEFAULT NULL,

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (tenant_id, test_key)
);

-- Efficient lookup for the API embargo pre-check
CREATE INDEX idx_test_configs_lookup
    ON trs.test_configs (tenant_id, test_group_id);

-- Efficient scan to find all currently embargoed tests (used by ops tooling)
CREATE INDEX idx_test_configs_embargo
    ON trs.test_configs (tenant_id, embargo_until)
    WHERE embargo_until IS NOT NULL;
```

**Embargo semantics:**

| `embargo_until` value | Embargoed? | Lifts automatically? |
|-----------------------|------------|----------------------|
| `NULL` | No — visible to all roles | N/A |
| Future timestamp | Yes — hidden from regular users | Yes — at `embargo_until` |
| Past timestamp | No — embargo has passed | Already lifted |

> **Ingestion is unaffected by embargo status.** The Score Processor never reads `embargo_until`. Scores for embargoed tests flow through the full pipeline (Postgres → CDC → ClickHouse) identically to non-embargoed tests. Embargo is a **read-side visibility gate only**, enforced exclusively at the API layer.

#### User Identity & Roles — SSO Only (no `trs.users` table)

TRS does **not** maintain a local users table. Red Hat SSO is the sole identity provider and
authoritative source of user roles. On login, RH SSO issues a signed JWT (`id_token`, RS256)
containing:

| JWT Claim | Type | Description |
|-----------|------|-------------|
| `sub` | string | Unique user identifier (opaque SSO ID) |
| `tenant_id` | string | TRS tenant the user belongs to |
| `roles` | string array | All roles assigned to the user in SSO |

Role names are defined and managed entirely within Red Hat SSO. TRS does not enumerate or
hardcode role names, **except for embargo access**, which is governed by `trs.embargo_roles`
(see below).

The Lambda authorizer validates the JWT signature against the RH SSO JWKS endpoint and rejects
expired or tampered tokens before the request reaches the API Lambda. The API Lambda reads
`roles` directly from the validated token claims — **no per-request database lookup is needed
for role resolution** outside of embargo access.

#### `trs.embargo_roles` — Per-Tenant Embargo Access Configuration

Because different tenants may use different role names in their SSO configuration, the set of
roles that grant embargo visibility is stored per tenant in TRS rather than hardcoded.

```sql
CREATE TABLE trs.embargo_roles (
    tenant_id  TEXT  NOT NULL,
    role_name  TEXT  NOT NULL,  -- must exactly match a value in the JWT 'roles' array claim
    PRIMARY KEY (tenant_id, role_name)
);
```

**How it works:** On an embargoed test request, the API reads the caller's `roles` array from
the JWT and checks whether any of those role values appear in `trs.embargo_roles` for the
caller's `tenant_id`. If at least one matches, the caller can see the test. If none match,
the request returns `404 Not Found`.

**Example data:**

```sql
-- Tenant "tx" allows two roles to view embargoed tests
INSERT INTO trs.embargo_roles (tenant_id, role_name) VALUES
    ('tx', 'STATE_ASSESSMENT_DIRECTOR'),
    ('tx', 'TRS_INTERNAL_STAFF');

-- Tenant "va" uses a different role name for the same purpose
INSERT INTO trs.embargo_roles (tenant_id, role_name) VALUES
    ('va', 'EMBARGO_REVIEWER');
```

> **Ops note:** `embargo_roles` rows are managed via direct DB update or an admin API.
> The in-process cache in `EmbargoService` expires every 5 minutes, so role changes
> take effect within 5 minutes without a Lambda redeployment.

**New normalized RTS relationship tables (replace `school_roster_members`):**

```sql
-- Direct district ↔ student (replaces derived two-hop path)
CREATE TABLE trs.district_student (
    tenant_id   TEXT     NOT NULL,
    district_id TEXT     NOT NULL,
    student_id  INTEGER  NOT NULL,
    active      BOOLEAN  NOT NULL DEFAULT TRUE,
    synced_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, district_id, student_id)
);

-- Direct school ↔ student (replaces school_roster_members)
CREATE TABLE trs.school_student (
    tenant_id   TEXT     NOT NULL,
    school_id   TEXT     NOT NULL,
    student_id  INTEGER  NOT NULL,
    active      BOOLEAN  NOT NULL DEFAULT TRUE,
    synced_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, school_id, student_id)
);

-- Direct roster ↔ student (replaces roster_members for CDC path)
CREATE TABLE trs.roster_student (
    tenant_id   TEXT     NOT NULL,
    roster_id   TEXT     NOT NULL,
    student_id  INTEGER  NOT NULL,
    active      BOOLEAN  NOT NULL DEFAULT TRUE,
    synced_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, roster_id, student_id)
);

-- Explicit district ↔ school relationship
CREATE TABLE trs.district_school (
    tenant_id   TEXT     NOT NULL,
    district_id TEXT     NOT NULL,
    school_id   TEXT     NOT NULL,
    PRIMARY KEY (tenant_id, district_id, school_id)
);

-- Explicit teacher ↔ roster relationship
CREATE TABLE trs.teacher_roster (
    tenant_id   TEXT     NOT NULL,
    teacher_id  TEXT     NOT NULL,
    roster_id   TEXT     NOT NULL,
    PRIMARY KEY (tenant_id, teacher_id, roster_id)
);
```

### 6.4 `trs.score_ingest_log` — Idempotency Table

```sql
CREATE TABLE trs.score_ingest_log (
    s3_key       TEXT        NOT NULL,
    etag         TEXT        NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (s3_key, etag)
);
-- Cleanup: pg_cron deletes rows older than 14 days nightly
```

### 6.5 `trs.report_cache` — Pre-Computed Aggregate Store

Replaces DynamoDB from v1. Holds pre-computed aggregates at school, district, and state scope.
Written by the Aggregate Refresh Lambda (computed via ClickHouse Gold or Postgres MV).
Read first by the API Lambda (Tier 1 of the fallback chain).

```sql
CREATE TABLE trs.report_cache (
    -- Key format: "{scope}#{tenant_id}#{scope_id}#{school_year}#{test_group_id}"
    -- Examples:
    --   "school#tx#s-789#2026#cp1-g5-ela"
    --   "district#tx#d-456#2026#cp1-g5-ela"
    --   "state#tx#tx#2026#cp1-g5-ela"
    cache_key    TEXT        PRIMARY KEY,
    payload      JSONB       NOT NULL,  -- full aggregate result
    computed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ NOT NULL,
    computed_by  TEXT        NOT NULL   -- 'clickhouse_gold' | 'postgres_mv' | 'postgres_live' | 'nightly_job'
);

CREATE INDEX idx_report_cache_expiry ON trs.report_cache (expires_at);
-- pg_cron cleanup: DELETE FROM trs.report_cache WHERE expires_at < now();
```

**Cache TTLs:**

| Scope | TTL | Rationale |
|-------|-----|-----------|
| Roster | No cache — always live | ~30 students; <5 ms in both Postgres and ClickHouse |
| School | 15 minutes | Fast ClickHouse Gold query; short TTL keeps it fresh during test windows |
| District | 5 minutes | Balances freshness vs. cost for large districts |
| State | Until next nightly run | State aggregates change slowly; nightly is sufficient |

### 6.6 Postgres Materialized Views (Fallback Pre-Aggregation)

These MVs are the Tier 3 fallback for school and district scope when both `report_cache` is stale and ClickHouse is unavailable. `REFRESH MATERIALIZED VIEW CONCURRENTLY` re-reads source data entirely — correct for rescores (no double-counting).

```sql
-- School-level overall aggregate MV
CREATE MATERIALIZED VIEW trs.mv_school_overall AS
SELECT
    ss.tenant_id,
    ss.school_year,
    ss.test_group_id,
    ss_m.school_id,
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)     AS pl1,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)     AS pl2,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)     AS pl3,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)     AS pl4,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)     AS pl5,
    AVG(ss.overall_scale_score)                           AS avg_scale_score,
    AVG(ss.overall_standard_error)                        AS avg_standard_error,
    NOW()                                                 AS refreshed_at
FROM trs.student_opportunities ss
JOIN trs.school_student ss_m
    ON ss.tenant_id = ss_m.tenant_id AND ss.student_id = ss_m.student_id
WHERE ss.is_aggregate_eligible = TRUE AND ss_m.active = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.test_group_id, ss_m.school_id;

CREATE UNIQUE INDEX uq_mv_school_overall
    ON trs.mv_school_overall (tenant_id, school_year, test_group_id, school_id);

-- District-level overall aggregate MV
CREATE MATERIALIZED VIEW trs.mv_district_overall AS
SELECT
    ss.tenant_id,
    ss.school_year,
    ss.test_group_id,
    ds.district_id,
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)     AS pl1,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)     AS pl2,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)     AS pl3,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)     AS pl4,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)     AS pl5,
    AVG(ss.overall_scale_score)                           AS avg_scale_score,
    NOW()                                                 AS refreshed_at
FROM trs.student_opportunities ss
JOIN trs.district_student ds
    ON ss.tenant_id = ds.tenant_id AND ss.student_id = ds.student_id
WHERE ss.is_aggregate_eligible = TRUE AND ds.active = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.test_group_id, ds.district_id;

CREATE UNIQUE INDEX uq_mv_district_overall
    ON trs.mv_district_overall (tenant_id, school_year, test_group_id, district_id);
```

**Refresh schedule (Lambda cron):**

| MV | Frequency |
|----|-----------|
| `mv_school_overall` | Every 15 min during test window; nightly off-window |
| `mv_district_overall` | Every 30 min during test window; nightly off-window |

---

## 7. Database Schema — ClickHouse Medallion Architecture

ClickHouse mirrors score data from Postgres via the CDC pipeline and serves fast aggregation queries. It is not the source of truth. If ClickHouse is unavailable, Postgres serves all queries.

```
[Bronze Layer]                      [Silver Layer]                    [Gold Layer]
VersionedCollapsingMergeTree   →    AggregatingMergeTree         →   AggregatingMergeTree
Raw CDC sign-pair stream             Deduplicated per-student          School / District cubes
(-1/+1 rescore events)               opportunity state                 Pre-aggregated for UI
```

### 7.1 Layer 1: Bronze (Raw CDC Stream)

The landing zone for all CDC events from the Sign-Pair Transformer. Stores raw sign-paired rows.

```sql
CREATE TABLE trs.student_scores_bronze (
    tenant_id              String,
    school_year            UInt16,
    test_group_id          String,
    student_id             Int32,
    opp_key                UUID,

    sign                   Int8,            -- -1 (undo) or +1 (redo)
    is_deleted             UInt8,           -- 1 on DELETE events; 0 otherwise

    test_key               String,
    opp_status             LowCardinality(String),
    condition_code         Nullable(LowCardinality(String)),
    tested_date            Date,
    date_scored            DateTime64(3),   -- millisecond precision; version key

    is_aggregate_eligible  UInt8 DEFAULT 0,

    overall_scale_score    Nullable(Float32),
    overall_raw_score      Nullable(Int32),
    overall_perf_level     Nullable(UInt8),
    overall_standard_error Nullable(Float32),

    ingested_at            DateTime64(3) DEFAULT now64(3)
)
ENGINE = VersionedCollapsingMergeTree(sign, date_scored)
PARTITION BY (tenant_id, school_year)
ORDER BY (tenant_id, school_year, test_group_id, student_id, opp_key, date_scored);
```

**Key properties:**
- `VersionedCollapsingMergeTree(sign, date_scored)` collapses `-1/+1` pairs during background merges when both `sign` values exist for the same `(opp_key, date_scored)`.
- A lone `-1` (DELETE undo) collapses against the original `+1` INSERT, physically removing the row.
- **Never queried directly by the API** — feeds Silver and Gold via Materialized Views.

### 7.2 Layer 2: Silver (Deduplicated Per-Student State)

The Silver layer collapses the Bronze sign-pair stream into a single authoritative version per student opportunity. Used for per-student roster queries in ClickHouse (optional; Postgres is the primary path for roster reads).

```sql
CREATE TABLE trs.student_scores_silver (
    tenant_id         String,
    school_year       UInt16,
    test_group_id     String,
    student_id        Int32,
    opp_key           UUID,
    -- Intermediate states — resolved at read time with -Merge combinators
    latest_score      AggregateFunction(argMax, Float32,      DateTime64(3)),
    latest_perf_level AggregateFunction(argMax, UInt8,        DateTime64(3)),
    is_eligible       AggregateFunction(argMax, UInt8,        DateTime64(3)),
    last_updated      AggregateFunction(max,    DateTime64(3)),
    -- Soft-delete sentinel: argMax on date_scored → latest event always wins
    -- DELETE sets is_deleted=1 with newest timestamp → wins until re-insert with newer T
    is_deleted        AggregateFunction(argMax, UInt8,        DateTime64(3))
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, test_group_id, student_id, opp_key);

CREATE MATERIALIZED VIEW trs.student_scores_silver_mv
TO trs.student_scores_silver AS
SELECT
    tenant_id, school_year, test_group_id, student_id, opp_key,
    argMaxState(overall_scale_score,           date_scored) AS latest_score,
    argMaxState(overall_perf_level,            date_scored) AS latest_perf_level,
    argMaxState(is_aggregate_eligible,         date_scored) AS is_eligible,
    maxState(date_scored)                                   AS last_updated,
    argMaxState(is_deleted,                    date_scored) AS is_deleted
FROM trs.student_scores_bronze
GROUP BY tenant_id, school_year, test_group_id, student_id, opp_key;
```

**Silver read query (roster scope — optional ClickHouse path):**
```sql
SELECT
    student_id,
    opp_key,
    argMaxMerge(latest_score)      AS score,
    argMaxMerge(latest_perf_level) AS perf_level,
    argMaxMerge(is_eligible)       AS eligible,
    maxMerge(last_updated)         AS scored_at
FROM trs.student_scores_silver
WHERE tenant_id = ? AND school_year = ? AND test_group_id = ?
  AND student_id IN (?)
GROUP BY student_id, opp_key
HAVING argMaxMerge(is_deleted) = 0;
```

> **Note on Silver role:** Because Postgres is the primary store for per-student data, the roster read path goes directly to Aurora in most cases. Silver is retained in ClickHouse for future features requiring ClickHouse-side deduplication (e.g., demographic slice reports at roster scope).

### 7.3 Layer 3: Gold — School Aggregates

```sql
CREATE TABLE trs.school_aggregates_gold (
    tenant_id       String,
    school_year     UInt16,
    school_id       String,
    test_group_id   String,
    avg_score       AggregateFunction(avg,            Float32),
    student_count   AggregateFunction(uniq,           Int32),
    pl_distribution AggregateFunction(groupUniqArray, UInt8)
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, school_id, test_group_id);

CREATE MATERIALIZED VIEW trs.bronze_to_school_gold_mv
TO trs.school_aggregates_gold AS
SELECT
    s.tenant_id, s.school_year, m.school_id, s.test_group_id,
    -- sign multiplication: -1 rows subtract, +1 rows add
    avgState(s.overall_scale_score * s.sign)    AS avg_score,
    uniqState(s.student_id)                     AS student_count,
    groupUniqArrayState(s.overall_perf_level)   AS pl_distribution
FROM trs.student_scores_bronze AS s
INNER JOIN (
    SELECT DISTINCT tenant_id, school_id, student_id
    FROM trs.membership_school_mirror FINAL
    WHERE is_deleted = 0
) AS m ON s.tenant_id = m.tenant_id AND s.student_id = m.student_id
WHERE s.is_aggregate_eligible = 1
GROUP BY s.tenant_id, s.school_year, m.school_id, s.test_group_id;
```

**Gold School read query:**
```sql
SELECT
    school_id,
    test_group_id,
    avgMerge(avg_score)                      AS average_score,
    uniqMerge(student_count)                 AS students_tested,
    groupUniqArrayMerge(pl_distribution)     AS perf_level_bands
FROM trs.school_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND school_id IN (?)
GROUP BY school_id, test_group_id
ORDER BY school_id;
```

### 7.4 Layer 3: Gold — District Aggregates

A **separate physical table** from the School Gold table — not a roll-up of the school table — to prevent double-counting students who appear in multiple school rosters (transfers, shared programs).

```sql
CREATE TABLE trs.district_aggregates_gold (
    tenant_id       String,
    school_year     UInt16,
    district_id     String,
    test_group_id   String,
    -- sumState over sign-adjusted indicator columns is cheaper than groupUniqArray at district scale
    pl_1_count      AggregateFunction(sum,  Int64),
    pl_2_count      AggregateFunction(sum,  Int64),
    pl_3_count      AggregateFunction(sum,  Int64),
    pl_4_count      AggregateFunction(sum,  Int64),
    avg_score       AggregateFunction(avg,  Float32),
    student_count   AggregateFunction(uniq, Int32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, district_id, test_group_id);

CREATE MATERIALIZED VIEW trs.bronze_to_district_gold_mv
TO trs.district_aggregates_gold AS
SELECT
    s.tenant_id,
    s.school_year,
    m.district_id,
    s.test_group_id,
    sumState(if(s.overall_perf_level = 1, s.sign, 0))  AS pl_1_count,
    sumState(if(s.overall_perf_level = 2, s.sign, 0))  AS pl_2_count,
    sumState(if(s.overall_perf_level = 3, s.sign, 0))  AS pl_3_count,
    sumState(if(s.overall_perf_level = 4, s.sign, 0))  AS pl_4_count,
    avgState(s.overall_scale_score * s.sign)            AS avg_score,
    uniqState(s.student_id)                             AS student_count
FROM trs.student_scores_bronze AS s
INNER JOIN (
    SELECT DISTINCT tenant_id, district_id, student_id
    FROM trs.membership_district_mirror FINAL
    WHERE is_deleted = 0
) AS m ON s.tenant_id = m.tenant_id AND s.student_id = m.student_id
WHERE s.is_aggregate_eligible = 1
GROUP BY s.tenant_id, s.school_year, m.district_id, s.test_group_id;
```

**Gold District read query:**
```sql
SELECT
    district_id,
    test_group_id,
    avgMerge(avg_score)       AS average_score,
    uniqMerge(student_count)  AS students_tested,
    sumMerge(pl_1_count)      AS pl_1,
    sumMerge(pl_2_count)      AS pl_2,
    sumMerge(pl_3_count)      AS pl_3,
    sumMerge(pl_4_count)      AS pl_4,
    round(sumMerge(pl_1_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_1_pct,
    round(sumMerge(pl_2_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_2_pct,
    round(sumMerge(pl_3_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_3_pct,
    round(sumMerge(pl_4_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_4_pct
FROM trs.district_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND district_id IN (?)
GROUP BY district_id, test_group_id
ORDER BY district_id;
```

### 7.5 Membership Mirror Tables

PeerDB replicates `school_student` → `membership_school_mirror` and `district_student` → `membership_district_mirror` via dedicated CDC replication jobs (separate from the score replication job). `version` = `_peerdb_version` (Postgres LSN); higher LSN wins. `is_deleted = 1` handles deletes.

```sql
CREATE TABLE trs.membership_school_mirror (
    tenant_id   String,
    school_id   String,
    student_id  Int32,
    is_deleted  UInt8   DEFAULT 0,
    version     Int64   DEFAULT 0   -- _peerdb_version (LSN)
)
ENGINE = ReplacingMergeTree(version, is_deleted)
PARTITION BY tenant_id
ORDER BY (tenant_id, school_id, student_id);

CREATE TABLE trs.membership_district_mirror (
    tenant_id   String,
    district_id String,
    student_id  Int32,
    is_deleted  UInt8   DEFAULT 0,
    version     Int64   DEFAULT 0
)
ENGINE = ReplacingMergeTree(version, is_deleted)
PARTITION BY tenant_id
ORDER BY (tenant_id, district_id, student_id);
```

#### Diagnostic Tables

> `student_scope_today` is **not used in any MV**; retained for ad-hoc investigations only.

```sql
-- Diagnostic only — not referenced by any Materialized View
CREATE TABLE trs.student_scope_today (
    tenant_id   String,
    student_id  Int32,
    school_id   String,
    district_id String,
    synced_at   DateTime64(3)
)
ENGINE = ReplacingMergeTree(synced_at)
PARTITION BY tenant_id
ORDER BY (tenant_id, student_id, school_id);
```

```sql
-- Student demographics
CREATE TABLE trs.student_attributes (
    tenant_id   String,
    student_id  Int32,
    gender      Nullable(LowCardinality(String)),
    ethnicity   Nullable(LowCardinality(String)),
    ell         UInt8 DEFAULT 0,
    lep         UInt8 DEFAULT 0,
    section_504 UInt8 DEFAULT 0,
    synced_at   DateTime64(3)
)
ENGINE = ReplacingMergeTree(synced_at)
PARTITION BY tenant_id
ORDER BY (tenant_id, student_id);
```

### 7.6 Medallion Layer Summary

| Feature | Bronze | Silver | Gold — School | Gold — District |
|---|---|---|---|---|
| **Engine** | `VersionedCollapsingMergeTree` | `AggregatingMergeTree` | `AggregatingMergeTree` | `AggregatingMergeTree` |
| **Granularity** | 1 row per CDC event | 1 row per student opportunity | 1 row per school / test | 1 row per district / test |
| **Primary Question** | "What changed and when?" | "Latest score per student?" | "School average?" | "District PL breakdown?" |
| **API Query Type** | Not queried directly | Roster-level reads (future) | School dashboard | District dashboard |
| **Rescore Handling** | Signs cancel on background merge | `argMax` keeps highest `date_scored` | `sign × score_value` cancels old value | `sign`-adjusted PL indicator columns |
| **Dictionary Lookup** | None | None | `membership_school_mirror FINAL (JOIN)` | `membership_district_mirror FINAL (JOIN)` |

---

## 8. Data Flow

### 8.1 Score Ingestion Path

```
Scoring System
    │
    ▼ PUT JSON file
  S3 Raw (s3://trs-raw/{tenant_id}/scores/...)
    │
    ▼ S3 Event Notification
  SQS Score Queue  ──[DLQ]──▶ Dead Letter Queue
    │
    ▼ Batch trigger (BatchSize=10, MaxBatchingWindow=30s)
  Lambda: Score Processor
    │
    ├─ 1. Download S3 score file
    ├─ 2. Validate required fields
    ├─ 3. Idempotency check (score_ingest_log)
    ├─ 4. Resolve test_group_id from test_key_config cache
    ├─ 5. Compute is_aggregate_eligible
    ├─ 6. Conflict guard (same opp_key + date_scored, different data → DLQ + alert)
    ├─ 7. UPSERT → Aurora student_opportunities (source of truth)
    │       ON CONFLICT ... WHERE EXCLUDED.date_scored > existing
    ├─ 8. UPSERT → Aurora student_component_scores
    │       ON CONFLICT ... WHERE EXCLUDED.date_scored > existing
    ├─ 9. INSERT → score_ingest_log (idempotency record)
    └─10. Acknowledge SQS message only after Aurora commit
    │
    │  *** NO direct ClickHouse write ***
    │  *** All ClickHouse writes originate from CDC pipeline ***
    │
    ▼ Aurora WAL (logical replication slot)
  PeerDB Engine (Fargate)
    │ Before + After row images per changed row
    ▼
  C# Sign-Pair Transformer (Fargate)
    │
    ├─ INSERT:  emit Bronze row (sign=+1)
    ├─ UPDATE:  emit Bronze pair (sign=-1 @T_before, sign=+1 @T_after) — single batch
    └─ DELETE:  emit Bronze row (sign=-1, is_deleted=1)
    │
    ▼ HTTP batch INSERT
  ClickHouse Bronze (VersionedCollapsingMergeTree)
    │
    ├─▶ Silver MV (AggregatingMergeTree) — fires on Bronze INSERT
    ├─▶ Gold School MV (AggregatingMergeTree) — fires on Bronze INSERT
    └─▶ Gold District MV (AggregatingMergeTree) — fires on Bronze INSERT
```

### 8.2 Rescore Handling

```
Score Processor receives rescore (same OppKey, higher DateScored):

  Postgres:
    ON CONFLICT (tenant_id, school_year, opp_key) DO UPDATE ...
    WHERE EXCLUDED.date_scored > student_opportunities.date_scored
    → Atomic UPDATE. Correct immediately. Emits WAL UPDATE event.

  PeerDB:
    Captures the UPDATE (Before = old score, After = new score)
    Delivers to Sign-Pair Transformer

  Sign-Pair Transformer:
    Emits Bronze batch:
      row 1: (opp_key, old_score, sign=-1, date_scored=T1)
      row 2: (opp_key, new_score, sign=+1, date_scored=T2)
    Single HTTP INSERT → atomic in ClickHouse

  ClickHouse Gold:
    avgState(score * sign): T1 contribution subtracted (-1×T1_score), T2 added (+1×T2_score)
    Net: Gold aggregate reflects new score correctly

  Report Cache (Postgres):
    TTL on affected report_cache rows expires (within 5–15 min depending on scope)
    OR: Score Processor notifies SNS → Aggregate Refresh Lambda invalidates specific rows
    Next cache read triggers Aggregate Refresh Lambda → re-queries ClickHouse Gold → writes updated cache
```

### 8.3 Membership Sync Path (RTS → TRS)

```
RTS external event or nightly full sync
    │
    ▼
  Lambda: RTS Membership Sync
    │
    ├─▶ Aurora: UPSERT district_student, school_student, roster_student,
    │           district_school, teacher_roster, student_attributes
    │
    └─▶ ClickHouse: INSERT INTO trs.student_attributes  (best-effort; ReplacingMergeTree)
    │
    ▼
  Note: PeerDB mirrors school_student → membership_school_mirror and
        district_student → membership_district_mirror via CDC replication jobs.
        No manual ClickHouse refresh needed for membership mirrors.
```

### 8.4 Aggregate Refresh Path

```
EventBridge Cron: every 15 min (school), 30 min (district), nightly (state)
  OR
SNS rescore notification → targeted cache invalidation
    │
    ▼
  Lambda: Aggregate Refresh
    │
    ├─ Probe ClickHouse health (200ms timeout on /ping)
    │
    ├─ [ClickHouse AVAILABLE]:
    │    Query Gold layer:
    │      School:    SELECT avgMerge(avg_score), uniqMerge(student_count)... FROM school_aggregates_gold
    │      District:  SELECT sumMerge(pl_1_count)... FROM district_aggregates_gold
    │    Upsert results:
    │      INSERT INTO trs.report_cache (cache_key, payload, computed_at, expires_at, computed_by='clickhouse_gold')
    │      ON CONFLICT (cache_key) DO UPDATE SET payload=EXCLUDED.payload, ...
    │
    ├─ [ClickHouse UNAVAILABLE]:
    │    Execute Postgres MV refresh:
    │      REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_school_overall;
    │      REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_district_overall;
    │    Query refreshed Postgres MVs:
    │      SELECT * FROM trs.mv_school_overall WHERE ...
    │      SELECT * FROM trs.mv_district_overall WHERE ...
    │    Upsert results:
    │      INSERT INTO trs.report_cache (..., computed_by='postgres_mv')
    │      ON CONFLICT (cache_key) DO UPDATE SET ...
    │
    └─ ALSO ALWAYS (regardless of ClickHouse status):
         REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_school_overall
         REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_district_overall
         (Keeps Postgres MVs current as a standing Tier 3 fallback)
```

### 8.5 Score File Rejection Path

When a score file cannot be processed (no test config, invalid JSON, or data conflict), the Score
Processor rejects it permanently rather than retrying — retrying will not fix a missing config.

**Rejection reasons:**

| Code | Trigger |
|------|---------|
| `UNKNOWN_TEST_CONFIG` | `test_key` in the file has no matching row in `trs.test_configs` |
| `VALIDATION_FAILED` | Required fields missing or file is unparseable |
| `DATA_CONFLICT` | Same `opp_key` + `date_scored` but different score data |

**Rejection flow (all three reasons follow the same path):**

```
Score Processor detects unprocessable file
    │
    ├─ 1. Copy file to s3://trs-rejected/{tenant_id}/{reason}/{test_key}/{date}/{filename}
    │       (preserve original in trs-raw — do NOT delete)
    ├─ 2. INSERT INTO trs.score_ingest_rejections (audit log + resolution tracking)
    ├─ 3. Emit CloudWatch metric TRS/Ingest/ScoreFileRejected
    │       CloudWatch alarms → SNS → ops (alarm fires once per threshold breach,
    │       not once per file — avoids alert storms on bulk missing-config rejections)
    └─ 4. ACK the SQS message  ← no retry; file is in rejection bucket
```

**Replay after config is added:**
Query `score_ingest_rejections WHERE resolved = FALSE` to find affected files, re-send their
original `source_s3_key` values back to the SQS Score Queue, then mark rows resolved.
The existing `score_ingest_log` idempotency check prevents duplicate rows on replay.

See [implementation/Score_Reject_Handling.md](implementation/Score_Reject_Handling.md) for full schema, C# implementation, and ops runbook.

---

## 9. API Fallback Logic

The API Lambda implements a **three-tier fallback chain** for every aggregate request. The chain is transparent to the caller. A `servedFrom` metadata field in the response enables observability.

### 9.1 Three-Tier Fallback (All Scopes)

An **embargo pre-check** runs before the fallback chain on every request. It is the first gate after JWT validation.

```
Incoming API Request
        │
        ▼
[Embargo Check] Does test_group_id have embargo_until > NOW()?
        │ YES (embargoed) — does JWT roles contain 'EMBARGO_VIEWER'?
        │   NO  ──────────────────────────────────────────────────▶  Return 404 Not Found
        │   YES ─────────────────────────────────────────────────▶  Continue to Tier 1
        │ NO (not embargoed)
        ▼
[Tier 1] Postgres report_cache ──── HIT (not expired) ──────────▶  Return result (~3ms)
        │ MISS or expired
        ▼
[Tier 2] ClickHouse healthy? ──── YES ──▶  Query Gold layer
        │                                   (school/district aggregates: <200ms)
        │                                   Write to report_cache
        │                                   Return result; servedFrom='clickhouse_gold'
        │ NO (timeout/5xx/unreachable)
        ▼
[Tier 3a] Postgres MV          ──── Fresh ──▶  Query mv_school_overall / mv_district_overall
        │                                       Write result to report_cache; computed_by='postgres_mv'
        │                                       Return result; servedFrom='postgres_mv'
        │ MV too stale (refreshed_at > threshold)
        ▼
[Tier 3b] Postgres Live Query  ──────────────▶  Run raw GROUP BY on student_opportunities
                                                 Write to report_cache; computed_by='postgres_live'
                                                 Return result; servedFrom='postgres_live'
                                                 Log CloudWatch metric: TRS/API/ClickHouseFallback
```

ClickHouse health is checked via a 200 ms timeout probe on `GET /ping`. A failure routes immediately to Tier 3 — no partial ClickHouse query is attempted.

**Why 404 and not 403 for embargoed tests?** Returning 403 would confirm to a regular user that the test exists but is restricted. 404 treats the test as non-existent from the caller's perspective, preventing information leakage about unreleased assessments.

### 9.2 Scope-Specific Routing

| Scope | Tier 1 (Cache) | Tier 2 (ClickHouse) | Tier 3 (Postgres) | Cache TTL |
|-------|---------------|---------------------|-------------------|-----------|
| **Roster** | No cache | Silver + `argMaxMerge` OR direct Bronze | Aurora live query | N/A — always live |
| **School** | `report_cache` | Gold School `avgMerge` + `uniqMerge` | `mv_school_overall` → raw live | 15 min |
| **District** | `report_cache` | Gold District `sumMerge` + `avgMerge` | `mv_district_overall` → raw live | 5 min |
| **State** | `report_cache` | Gold District → GROUP BY all | `mv_district_overall` GROUP BY → very slow | Until next nightly run |

### 9.3 C# Fallback Implementation (School scope)

```csharp
public async Task<AggregateResult> GetSchoolAggregateAsync(
    string tenantId, string schoolId, string testGroupId, int schoolYear,
    IReadOnlyList<string> callerRoles)
{
    // Embargo pre-check (before any cache or DB query)
    await _embargoService.AssertVisibleAsync(tenantId, testGroupId, callerRoles);
    // Throws EmbargoException (→ 404) if embargoed and caller lacks EMBARGO_VIEWER role.
    // embargo_until is cached in-process for 60 s to avoid a DB round-trip per request.

    // Tier 1: Postgres report_cache
    var cached = await _pgRepo.GetReportCacheAsync(
        $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}");
    if (cached != null && cached.ExpiresAt > DateTime.UtcNow)
        return cached.Payload;

    // Tier 2: ClickHouse Gold layer
    if (await _clickHouseHealth.IsAvailableAsync())
    {
        try
        {
            var chResult = await _chRepo.GetSchoolGoldAggregateAsync(
                tenantId, schoolId, testGroupId, schoolYear);
            // Async background cache write — do not block the response
            _ = _pgRepo.UpsertReportCacheAsync(
                $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}",
                chResult, computedBy: "clickhouse_gold", ttlMinutes: 15);
            return chResult;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "ClickHouse unavailable; falling back to Postgres");
            _metrics.Increment("TRS/API/ClickHouseFallback");
        }
    }

    // Tier 3a: Postgres MV
    var mvResult = await _pgRepo.GetSchoolFromMvAsync(tenantId, schoolId, testGroupId, schoolYear);
    if (mvResult != null)
    {
        _ = _pgRepo.UpsertReportCacheAsync(
            $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}",
            mvResult, computedBy: "postgres_mv", ttlMinutes: 15);
        return mvResult;
    }

    // Tier 3b: Postgres live query
    var pgResult = await _pgRepo.GetSchoolAggregateLiveAsync(
        tenantId, schoolId, testGroupId, schoolYear);
    _ = _pgRepo.UpsertReportCacheAsync(
        $"school#{tenantId}#{schoolId}#{schoolYear}#{testGroupId}",
        pgResult, computedBy: "postgres_live", ttlMinutes: 15);
    return pgResult;
}
```

### 9.4 ClickHouse Health Check (in-process circuit breaker)

> See §9.5 for the `EmbargoService` implementation.

```csharp
public class ClickHouseHealthCheck
{
    private volatile bool _available = true;
    private DateTime _lastCheck = DateTime.MinValue;
    private const int CheckIntervalSeconds = 10;

    public async Task<bool> IsAvailableAsync()
    {
        if ((DateTime.UtcNow - _lastCheck).TotalSeconds < CheckIntervalSeconds)
            return _available;
        try
        {
            await _connection.ExecuteScalarAsync("SELECT 1");
            _available = true;
        }
        catch
        {
            _available = false;
        }
        _lastCheck = DateTime.UtcNow;
        return _available;
    }
}
```

### 9.5 Embargo Service (in-process cache)

`EmbargoService` is called by the API Lambda immediately after JWT validation, before any cache
or database query. It enforces the embargo visibility gate with two in-process cached lookups:

| Lookup | Source | Cache TTL |
|--------|--------|-----------|
| `embargo_until` for the `test_group_id` | `trs.test_configs` | 60 s per `(tenant_id, test_group_id)` |
| Permitted embargo role names | `trs.embargo_roles` | 5 min per `tenant_id` |

If the test is currently embargoed and none of the caller's JWT `roles` claims match any row in
`trs.embargo_roles` for the tenant, the service throws `EmbargoException` → **HTTP 404**.

The **test discovery endpoint** (`GET /v1/{tenantId}/tests?schoolYear={Y}`) is the only place
callers learn which `test_group_id`s exist. It applies the same `embargo_roles` check and omits
embargoed entries for non-permitted callers, so regular users never acquire a `test_group_id`
to probe via the aggregate endpoints.

See [implementation/EmbargoService.md](implementation/EmbargoService.md) for the full C#
class, repository SQL queries, test discovery endpoint SQL, and cache invalidation notes.

---

## 10. Query Patterns

### 10.1 Roster Queries (Tier 2 — ClickHouse Silver or direct Bronze)

**Overall aggregate (ClickHouse Silver path):**
```sql
SELECT
    student_id,
    argMaxMerge(latest_score)      AS score,
    argMaxMerge(latest_perf_level) AS perf_level,
    argMaxMerge(is_eligible)       AS eligible,
    maxMerge(last_updated)         AS scored_at
FROM trs.student_scores_silver
WHERE tenant_id = ? AND school_year = ? AND test_group_id = ?
  AND student_id IN (?)           -- ~30 IDs resolved from Aurora roster_members
GROUP BY student_id, opp_key
HAVING argMaxMerge(is_deleted) = 0;
```

**Overall aggregate (Postgres fallback):**
```sql
SELECT
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE overall_perf_level = 1)        AS pl1,
    COUNT(*) FILTER (WHERE overall_perf_level = 2)        AS pl2,
    COUNT(*) FILTER (WHERE overall_perf_level = 3)        AS pl3,
    COUNT(*) FILTER (WHERE overall_perf_level = 4)        AS pl4,
    AVG(overall_scale_score)                              AS avg_scale_score
FROM trs.student_opportunities
WHERE tenant_id            = $1
  AND school_year          = $2
  AND test_group_id        = $3
  AND student_id           = ANY($4)
  AND is_aggregate_eligible = TRUE;
```

**Per-student score list (always Postgres — primary data):**
```sql
SELECT
    student_id,
    opp_key,
    overall_scale_score,
    overall_perf_level,
    overall_raw_score,
    overall_standard_error,
    opp_status,
    condition_code,
    tested_date
FROM trs.student_opportunities
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND student_id    = ANY($4)
ORDER BY student_id;
```

**Standard-level class aggregate (Postgres):**
```sql
SELECT
    component_id                                          AS standard_id,
    COUNT(*)                                              AS students_tested,
    COUNT(*) FILTER (WHERE perf_level = 1)               AS pl1,
    COUNT(*) FILTER (WHERE perf_level = 2)               AS pl2,
    COUNT(*) FILTER (WHERE perf_level = 3)               AS pl3,
    AVG(scale_score)                                     AS avg_scale_score
FROM trs.student_component_scores
WHERE tenant_id            = $1
  AND school_year          = $2
  AND test_group_id        = $3
  AND student_id           = ANY($4)
  AND component_type       = 'STANDARD'
  AND is_aggregate_eligible = TRUE
GROUP BY component_id
ORDER BY component_id;
```

### 10.2 School / District Queries (Tier 2 — ClickHouse Gold)

**School aggregate — ClickHouse Gold (Tier 2):**
```sql
SELECT
    school_id,
    test_group_id,
    avgMerge(avg_score)                  AS average_score,
    uniqMerge(student_count)             AS students_tested,
    groupUniqArrayMerge(pl_distribution) AS perf_level_bands
FROM trs.school_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND school_id IN (?)
GROUP BY school_id, test_group_id;
```

**School aggregate — Postgres MV (Tier 3a):**
```sql
SELECT students_tested, pl1, pl2, pl3, pl4, pl5, avg_scale_score
FROM trs.mv_school_overall
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND school_id     = $4;
```

**District aggregate — ClickHouse Gold (Tier 2):**
```sql
SELECT
    district_id,
    test_group_id,
    avgMerge(avg_score)                                       AS average_score,
    uniqMerge(student_count)                                  AS students_tested,
    sumMerge(pl_1_count)                                      AS pl_1,
    sumMerge(pl_2_count)                                      AS pl_2,
    sumMerge(pl_3_count)                                      AS pl_3,
    sumMerge(pl_4_count)                                      AS pl_4,
    round(sumMerge(pl_1_count)*100.0/nullIf(uniqMerge(student_count),0),1) AS pl_1_pct
FROM trs.district_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND district_id IN (?)
GROUP BY district_id, test_group_id;
```

**District aggregate — Postgres MV (Tier 3a):**
```sql
SELECT students_tested, pl1, pl2, pl3, pl4, pl5, avg_scale_score
FROM trs.mv_district_overall
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND district_id   = $4;
```

---

## 11. Aggregate Cache & Materialized Views

### 11.1 Serving Strategy by Scope

| Scope | Rows scanned | ClickHouse Gold latency | Postgres MV latency | Postgres raw latency | Cache TTL |
|-------|-------------|------------------------|--------------------|---------------------|-----------|
| Roster (~30 students) | ~30 | <5 ms | N/A | <10 ms (index seek) | No cache |
| School (~6,800 students) | ~6,800 | <30 ms | <5 ms (MV lookup) | 200–800 ms | 15 min |
| District ≤300k | ≤300k | 50–200 ms | <5 ms (MV lookup) | 3–15 s | 5 min |
| District 300k–2M | ~500k–2M | 150–500 ms | <5 ms (MV lookup) | 15–120 s | 5 min |
| State (TX 5.5M) | 5.5M | 400 ms–3 s | <5 ms (MV lookup) | Too slow — cache only | Nightly |

> **Rule for Postgres raw path:** A direct `GROUP BY` on `student_opportunities` for large
> districts or state is too slow for an interactive API response. At district/state scope,
> always prefer the Postgres MV or `report_cache`. Only fall back to a live Postgres raw query
> for school scope and below.

### 11.2 Cache Invalidation

| Trigger | Action |
|---------|--------|
| TTL expires (`expires_at < NOW()`) | API serves stale cache + triggers async Aggregate Refresh Lambda |
| Rescore event arrives at Score Processor | Publish to SNS `trs-rescore-events`; Aggregate Refresh Lambda subscribes, invalidates affected `report_cache` rows, re-queries ClickHouse Gold |
| RTS Membership Sync completes | Aggregate Refresh Lambda re-runs Postgres MV refreshes |
| Manual admin action | `DELETE FROM trs.report_cache WHERE cache_key LIKE 'district#tx#d-456%'` |
| **Embargo applied to a previously visible test** | Ops must manually purge affected `report_cache` entries: `DELETE FROM trs.report_cache WHERE cache_key LIKE '%#{test_group_id}%'`. Until purged, existing cache entries remain accessible to anyone who holds the exact key — in practice negligible since regular users only discover tests via the discovery endpoint (which already filters by embargo). Also invalide the in-process `EmbargoService` cache by waiting up to 60 s or redeploying. |
| **Embargo lifted** (`embargo_until` set to past or NULL) | No action required. The `report_cache` entries that were populated by `EMBARGO_VIEWER` users remain valid and will be served immediately once regular users can access the test. The in-process cache expires within 60 s automatically. |

### 11.3 Nightly Pre-Warm Job (EventBridge Scheduler, 02:00 AM per tenant)

Pre-computes state and school aggregates so the first user of the day always hits Tier 1. Runs as a separate Lambda.

```
For each (tenant_id, school_year, test_group_id) combination:
  1. Query ClickHouse Gold → build state aggregate
  2. INSERT INTO report_cache (cache_key='state#...',  expires_at='tomorrow 02:00', computed_by='nightly_job')
  3. For each school in tenant:
     Query ClickHouse Gold → build school aggregate
     INSERT INTO report_cache (cache_key='school#...', expires_at=now()+15min, computed_by='nightly_job')
  4. REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_school_overall
  5. REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_district_overall
```

---

## 12. Resilience & Failure Modes

### 12.1 ClickHouse Instance Reboot

```
ClickHouse goes offline
        │
        ▼
API health check fails → all queries redirect to Tier 1 (report_cache)
                          or Tier 3 (Postgres MV / live)
                          Zero user-visible impact
        │
        ▼
PeerDB pauses delivery (destination unreachable)
Aurora Postgres continues writing ALL changes to WAL
WAL accumulates in PeerDB replication slot (capped by max_slot_wal_keep_size = 10 GB)
        │
        ▼
ClickHouse comes back online
        │
        ▼
PeerDB checks last saved LSN checkpoint → replays all missed WAL changes automatically
        │
        ▼
ClickHouse Bronze/Silver/Gold reaches real-time status
Sign-Pair idempotency ensures duplicate CDC messages (from restart) collapse correctly
API resumes ClickHouse Gold queries
```

### 12.2 PeerDB Fargate Task Restart

```
Fargate task restarts (deployment, OOM, task replacement)
        │
        ▼
Postgres replication slot holds WAL for this specific slot
WAL does NOT advance past the slot's confirmed_flush_lsn while PeerDB is offline
        │
        ▼
Fargate task comes back → PeerDB reconnects to same replication slot
        │
        ▼
PeerDB requests all changes since last confirmed LSN → processes backlog into ClickHouse
Buffer window: WAL capped at 10 GB (set max_slot_wal_keep_size = 10 GB on Aurora)
        │
        ▼
VersionedCollapsingMergeTree + Sign-Pair logic is idempotent
Duplicate messages from messy restart are collapsed automatically
```

### 12.3 Total EC2 Loss (Disaster Recovery)

```
ClickHouse EC2 terminated / EBS volume corrupted
        │
        ▼
API immediately falls back to Postgres (Tier 1 / Tier 3) — users unaffected
        │
        ▼
Provision new EC2 instance via CDK / Terraform
Apply Medallion schema:
  Bronze (VersionedCollapsingMergeTree)
  Silver MV → Silver table
  membership_school_mirror (ReplacingMergeTree)
  membership_district_mirror (ReplacingMergeTree)
  Gold School MV → school_aggregates_gold
  Gold District MV → district_aggregates_gold
  student_attributes
  student_scope_today (diagnostic only)
        │
        ▼
Trigger PeerDB "Initial Load"
PeerDB performs high-speed parallel snapshot of all score data from Aurora → new ClickHouse
  TX scale (5.5M students, ~55M rows): estimated 30–90 minutes
        │
        ▼
Snapshot completes → PeerDB automatically switches to Streaming Mode
Pulls in all WAL changes that occurred WHILE the snapshot was running
        │
        ▼
ClickHouse is current → API resumes ClickHouse Gold queries
```

> Throughout the entire multi-hour rebuild, users continue seeing reports via the Postgres fallback.
> The rebuild is fully transparent at the API layer.

### 12.4 Resilience Summary

| Failed Component | User Impact | Data Loss Risk | Recovery Mechanism | Est. Recovery Time |
|---|---|---|---|---|
| **ClickHouse instance** | None — API falls back to Postgres | None — WAL buffered in replication slot | PeerDB LSN replay on restart | Minutes |
| **PeerDB Fargate task** | None — API falls back to Postgres | None — replication slot holds WAL | Fargate auto-restart; PeerDB resumes from last LSN | Minutes |
| **Sign-Pair Transformer task** | None — API falls back | None — replication slot holds WAL | Fargate auto-restart; retry from last unconsumed SQS message | Minutes |
| **Total EC2 + EBS loss** | None — API falls back | Temporary aggregate lag only | PeerDB Initial Load snapshot → Streaming handover | 30–90 min (TX scale) |
| **Aurora Postgres** | Full outage (primary store down) | Depends on Aurora HA tier | Aurora Multi-AZ automatic failover | ~30 seconds (RDS failover) |

> **The architectural guarantee:** ClickHouse holds no data that cannot be fully reconstructed from Aurora.
> Its loss is bounded to query performance degradation with automatic API-level mitigation.
> Aurora Multi-AZ is the only tier where failure causes true user impact.

---

## 13. Scale & Capacity

### 13.1 State-Level Student Volumes

| State | Total Students | Largest District | Largest School |
|-------|----------------|-----------------|----------------|
| Texas | 5,543,751 | 189,934 (Houston ISD) | 6,798 (Allen HS) |
| Virginia | 1,259,958 | 179,858 (Fairfax County) | 5,100 (Alexandria City) |
| Illinois | 1,850,074 | 321,666 (Chicago PS) | 4,300 (Lane Tech) |
| Indiana | 1,009,888 | 31,000 (Indianapolis PS) | 5,400 (Carmel HS) |

### 13.2 Annual Row Counts (Texas Scale)

| Table | Rows / Year | Postgres Storage (uncompressed) | ClickHouse Bronze (compressed) |
|-------|------------|--------------------------------|-------------------------------|
| `student_opportunities` | ~55M | ~22 GB | ~2 GB |
| `student_component_scores` | ~1.65B | ~660 GB | ~9–13 GB |
| `student_scores_bronze` | ~55M (+ rescore pairs) | N/A | ~2.5 GB |
| `school_aggregates_gold` | ~30k (per test × school) | N/A | < 100 MB |
| `district_aggregates_gold` | ~1k (per test × district) | N/A | < 10 MB |

### 13.3 Postgres Query Performance at Scale

| Query | Students | Expected Postgres Latency |
|-------|---------|--------------------------|
| Roster aggregate (Q1, live) | 30 | < 20 ms (index seek on student_id array) |
| School aggregate (MV lookup) | N/A | < 5 ms (index lookup) |
| School aggregate (raw live) | 6,800 | 200–600 ms (partial index + bitmap scan) |
| District aggregate (MV lookup) | N/A | < 5 ms (index lookup) |
| District aggregate (raw live, Houston ISD) | 189,934 | 15–60 s (too slow; use MV/cache) |
| State aggregate (raw live, TX) | 5.5M | > 60 s (never use raw; nightly pre-computed only) |

### 13.4 ClickHouse Gold Query Latency (r6i.2xlarge)

| Query | Scope | Expected ClickHouse Latency |
|-------|-------|--------------------------|
| Gold School | 6,800 students | < 30 ms |
| Gold District | 50k students | < 50 ms |
| Gold District | 200k students (Houston ISD) | 100–200 ms |
| Gold State | 5.5M students | 400 ms – 3 s |

### 13.5 SQS Ingestion Throughput

- Up to 5,500 students / minute at TX peak.
- BatchSize=10, MaxBatchingWindow=30s → ~50 concurrent Lambda invocations.
- Each invocation: 1 Postgres batch UPSERT (source of truth) — no direct ClickHouse write.
- Postgres batch UPSERT of 10 rows: ~20–50 ms. Well within Lambda timeout.
- CDC pipeline lag (PeerDB → Transformer → Bronze): ~5–30 s during active ingestion.

---

## 14. Cost Analysis

### 14.1 Base Infrastructure (Texas Scale, Monthly)

| Service | Config | Monthly On-Demand |
|---------|--------|------------------|
| Aurora Serverless v2 (PRIMARY — all scores + config + membership) | avg 0.5–1 ACU; ~50 GB storage | ~$53–96 |
| EC2: ClickHouse single node (aggregation only) | `r6i.2xlarge` (64 GB RAM, 8 vCPU) | $362 |
| EBS: ClickHouse data volume | 200 GB `gp3` | $16 |
| Lambda (Score Processor + API + RTS Sync + Aggregate Refresh) | ~50M invocations | $10–20 |
| S3 (raw score files + ClickHouse backup) | ~5 TB/year | $15–30 |
| SQS (ingest queue + DLQ + PeerDB events) | ~50M messages | $3–5 |
| CloudFront + API Gateway | 1,000 concurrent users | $10–20 |
| CloudWatch (metrics + logs + alarms) | | $10–20 |
| **Base Subtotal** | | **~$479–567/month** |

### 14.2 CDC Layer (Option B — PeerDB, Recommended)

| Service | Config | Monthly Cost |
|---------|--------|-------------|
| PeerDB Engine (Fargate) | 0.25 vCPU / 0.5 GB | ~$9 |
| C# Sign-Pair Transformer (Fargate) | 0.5 vCPU / 1 GB | ~$18 |
| **CDC Subtotal** | | **~$27/month** |

### 14.3 Total Cost by Scale

| Scale | ClickHouse Node | On-Demand / Month | 1-Year Reserved / Month |
|-------|-----------------|------------------|-----------------------|
| **MVP / single client** | `r6i.xlarge` (32 GB) | ~$270–310 | ~$195–235 |
| **Virginia (1.26M students)** | `r6i.xlarge` (32 GB) | ~$310–360 | ~$238–288 |
| **Texas (5.5M students)** | `r6i.2xlarge` (64 GB) | ~$506–594 | ~$284–372 |

> **1-Year Reserved EC2** applies ~42% discount to the ClickHouse node
> (`r6i.2xlarge`: $362/mo on-demand → ~$210/mo reserved), saving ~$152/month.

### 14.4 Cost Savings vs. Original Architecture

| Architecture | Monthly Cost (TX scale) |
|---|---|
| Original (ClickHouse primary + MSK + replica) | ~$1,345–$1,385 |
| **This architecture** (Postgres primary + PeerDB + single ClickHouse) | ~$506–$594 |
| **Savings** | **~$830–$870/month (~$10,000/year)** |

---

## 15. Key Engineering Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **Aurora PostgreSQL as primary store** | Single source of truth; AWS-managed HA/failover; transactional UPSERT handles rescores atomically; enables full query fallback without ClickHouse |
| 2 | **Score Processor writes to Postgres only; ClickHouse receives data via CDC** | Eliminates dual-write race conditions; ClickHouse state is always derived from Postgres; rescore correctness guaranteed by WAL before/after images |
| 3 | **PeerDB (not Debezium + MSK) for CDC** | ~$27/month vs. ~$586/month for MSK; Postgres WAL replication slot provides durable buffering without a separate broker; simpler operational model |
| 4 | **Sign-Pair Transformer (-1/+1) for ClickHouse INSERT correctness** | ClickHouse engines are append-only; a naive UPDATE stream permanently double-counts rescored values in Gold aggregates; the sign-pair pattern solves this atomically |
| 5 | **Medallion Architecture (Bronze → Silver → Gold)** | Bronze absorbs raw CDC events; Silver de-duplicates per student; Gold pre-aggregates at school/district scope with `AggregatingMergeTree`; each layer has a clear responsibility |
| 6 | **No ClickHouse streaming MVs for school/district pre-aggregation at query-time** | Streaming MVs (even chained on top of `AggregatingMergeTree`) accumulate incremental INSERT batches — not the final merged state. A rescore fires two MV writes (T1 batch + T2 batch) and the old T1 contribution cannot be subtracted. Gold MV aggregation via `sign * score_value` with `VersionedCollapsingMergeTree` collapse is the correct pattern. |
| 7 | **Postgres `report_cache` replaces DynamoDB** | Eliminates a separate cache service; co-located with score data; Aurora Serverless v2 scales read capacity automatically; `computed_by` metadata enables observability |
| 8 | **Postgres `MATERIALIZED VIEW CONCURRENTLY` for school/district pre-aggregation (Tier 3 fallback)** | `REFRESH` re-reads all source rows → fully correct for rescores (no double-counting). Non-blocking refresh preserves availability. |
| 9 | **Three-tier API fallback: report_cache → ClickHouse Gold → Postgres MV / live** | Sub-millisecond cache responses when warm; fast ClickHouse Gold as primary real-time path; Postgres MV as always-available performant fallback for district scope; raw Postgres for school scope |
| 10 | **Single ClickHouse instance (no replica)** | ClickHouse is disposable — it can be fully rebuilt from Postgres via PeerDB Initial Load. A replica is only justified when ClickHouse is the primary store. Saves ~$150–$362/month. |
| 11 | **Membership mirror JOIN FINAL replaces dictGet lookups** | `HASHED()` dictionary layout stores exactly one value per primary key, silently dropping the second school for multi-enrolled students. `ReplacingMergeTree` mirrors natively support N rows per student. `FINAL` + `DISTINCT` subquery in the Gold MV provides correct multi-enrollment attribution at MV write time with no race window. |
| 12 | **`REPLICA IDENTITY FULL` on all replicated Aurora tables** | Without it, PeerDB Before images contain only PK columns on UPDATE/DELETE — all non-key fields are null — making the Sign-Pair Transformer's undo row useless for aggregate subtraction |
| 13 | **`date_scored` as `DateTime64(3)` (millisecond precision) everywhere** | `argMaxState` in Silver and Gold MVs resolves ties by the version timestamp; second-precision timestamps create 1-second tie windows where two events (e.g., rapid rescore) may be arbitrarily ordered |
| 14 | **Postgres table partitioning by `tenant_id` (LIST) + `school_year` (LIST)** | Partition pruning on year-bound queries; enables efficient per-tenant maintenance; application always queries parent table — never leaf partition names |
| 15 | **`computed_by` metadata in `report_cache`** | Operators can detect if ClickHouse has been unavailable (cache populated from Postgres) and investigate the root cause; enables SLA monitoring per tier |

---

## 16. Operational Prerequisites

These two requirements **must** be enforced before enabling PeerDB replication. Skipping either causes silent data corruption in ClickHouse aggregates.

### 16.1 `REPLICA IDENTITY FULL` on Replicated Tables

```sql
-- Run once per replicated table on the Aurora Postgres instance
ALTER TABLE trs.student_opportunities        REPLICA IDENTITY FULL;
ALTER TABLE trs.student_component_scores     REPLICA IDENTITY FULL;
ALTER TABLE trs.school_student               REPLICA IDENTITY FULL;
ALTER TABLE trs.district_student             REPLICA IDENTITY FULL;
```

> **WAL size impact:** `REPLICA IDENTITY FULL` increases WAL volume because every UPDATE and DELETE
> writes the full row image. At TRS scale this is acceptable — score mutations are low-frequency
> relative to read volume.

### 16.2 Millisecond Precision on `date_scored`

```sql
-- Correct — DateTime64(3) everywhere a version key appears
date_scored   TIMESTAMPTZ  -- Postgres (stores microsecond but expose as millisecond to Transformer)
date_scored   DateTime64(3)  -- ClickHouse Bronze, Silver, Gold schemas

-- Wrong — second precision creates 1-second tie windows in argMaxState resolution
date_scored   DateTime     -- ClickHouse DateType without precision
```

Enforce `DateTime64(3)` in the C# Transformer's Bronze insert payload, the Silver schema, and the upstream Postgres column definition.

### 16.3 WAL Cap Configuration

```sql
-- On Aurora Postgres: cap WAL growth during extended PeerDB downtime
-- If cap is hit, the replication slot is invalidated → PeerDB Initial Load required
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
```

### 16.4 Aurora Tables Required by PeerDB Membership Replication

The following 5 normalized RTS relationship tables must exist in Aurora before PeerDB membership
replication jobs are configured. No Aurora views are required for ClickHouse schema deployment.

```sql
-- Required before configuring PeerDB CDC jobs for membership mirrors:
ALTER TABLE trs.school_student    REPLICA IDENTITY FULL;  -- source for membership_school_mirror
ALTER TABLE trs.district_student  REPLICA IDENTITY FULL;  -- source for membership_district_mirror
-- (school_student, district_student, roster_student, district_school, teacher_roster
--  must be created; see §6.3 for full DDL)
```

### 16.5 Deployment Order for ClickHouse Schema

Deploy ClickHouse tables and MVs in this strict order:

1. `trs.student_scores_bronze` (base table — must exist before any MVs that source from it)
2. `trs.student_scores_silver` (target table for Silver MV)
3. `trs.student_scores_silver_mv` (Materialized View — Bronze → Silver)
4. `trs.membership_school_mirror` (membership CDC mirror — must exist before School Gold MV)
5. `trs.membership_district_mirror` (membership CDC mirror — must exist before District Gold MV)
6. `trs.school_aggregates_gold` (target table for School Gold MV)
7. `trs.bronze_to_school_gold_mv` (Materialized View — Bronze + membership_school_mirror → Gold School)
8. `trs.district_aggregates_gold` (target table for District Gold MV)
9. `trs.bronze_to_district_gold_mv` (Materialized View — Bronze + membership_district_mirror → Gold District)
10. `trs.student_attributes` (demographics mirror — standalone)
11. `trs.student_scope_today` (diagnostic only; deploy last, no dependents)

---

## 17. MVP Scope & Future Enhancements

### 17.1 MVP Scope

- ✅ Roster-level aggregates (overall, per-student, standard-level)
- ✅ School and district aggregates (overall) — via `report_cache` + Postgres MVs + ClickHouse Gold
- ✅ Rescore handling (Postgres UPSERT `WHERE date_scored > existing` + Sign-Pair CDC to ClickHouse)
- ✅ Retest storage (separate rows per unique `opp_key`)
- ✅ RTS membership sync (Aurora primary; ClickHouse mirror best-effort)
- ✅ Test configuration management
- ✅ Full API fallback: ClickHouse unavailable → Postgres serves all queries transparently
- ✅ Postgres `report_cache` and materialized views for school/district scope
- ✅ PeerDB CDC pipeline feeding ClickHouse Bronze Medallion layer
- ✅ Sign-Pair Transformer for correct rescore propagation to Gold aggregates
- ✅ ClickHouse Medallion Architecture (Bronze / Silver / Gold) with membership mirror JOIN Gold MVs
- ✅ Red Hat SSO authentication; multi-tenant isolation via `tenant_id` partition

### 17.2 Deferred Enhancements

| # | Enhancement | Notes |
|---|-------------|-------|
| 1 | State-level Postgres MV (`mv_state_overall`) | Nightly refresh only; too expensive more frequently at 5.5M student scale |
| 2 | Multi-opportunity selection rule | Best / latest rule per TestFamily when a student has multiple `OppKey`s for the same `TestKey`, grade, and year |
| 3 | Standard-level aggregates for school / district | Extend `report_cache`, MVs, and ClickHouse Gold for `component_type = 'STANDARD'` |
| 4 | Demographic slice reports | `student_attributes` JOIN already designed; add `report_cache` rows keyed on demographic dimension |
| 5 | Reporting Category (RC) and Writing Dimension scores | Stored at ingest (`component_type = 'RC'/'WRITING_DIM'`); display views deferred |
| 6 | ClickHouse rehydration automation | Lambda to automatically detect ClickHouse lag vs. Aurora (compare max `date_scored`) and trigger PeerDB Initial Load |
| 7 | Postgres slow-query monitoring | Alerts on fallback Postgres raw queries; detect if district-scope live queries are being triggered (indicates MV staleness) |
| 8 | Cross-year trend comparisons | Out of scope |
| 9 | Demographics Gold layer | Add `gender`, `ethnicity`, `ell` as additional `ORDER BY` dimensions in Gold tables once base aggregates are validated |

---

## 18. Tech Stack Reference

| Layer | Technology | Notes |
|-------|-----------|-------|
| Language | C# (.NET 8) | Lambda functions, API layer, Fargate transformer |
| Primary DB | Aurora PostgreSQL Serverless v2 | **Source of truth for all data**: scores, membership, config, report cache, idempotency |
| Analytical DB | ClickHouse 24.x — single `r6i.2xlarge` | Self-hosted on EC2; aggregation-only; Medallion architecture; rebuilt from Postgres on total loss |
| CDC | PeerDB on Fargate | WAL logical replication; Before/After images; LSN checkpointing; SQS event handoff |
| CDC Transformer | C# `BackgroundService` on Fargate | Sign-Pair (-1/+1) emission; Bronze HTTP batch INSERT |
| Messaging | AWS SQS (+ DLQ) | Score ingestion queue; PeerDB → Transformer event queue; DLQs for both |
| Object storage | AWS S3 | Raw score files; ClickHouse daily backup |
| Compute | AWS Lambda + Fargate | Lambda: Score Processor, RTS Sync, API handlers, Aggregate Refresh; Fargate: PeerDB, Transformer |
| API | AWS API Gateway (HTTP API) | REST endpoints |
| Auth | Red Hat SSO (OIDC) | External IdP — sole source of user identity and roles. OIDC `id_token` (RS256); Lambda authorizer validates against RH SSO JWKS endpoint. JWT carries `tenant_id` + `roles` (string array). TRS has no local users table. Embargo access determined by matching JWT roles against `trs.embargo_roles` per tenant |
| Front-end | React SPA | Hosted on CloudFront |
| CDN | AWS CloudFront | SPA delivery; static asset caching |
| IaC | AWS CDK (C#) | Infrastructure as code |
| Postgres client | `Npgsql` NuGet v8.x | C# PostgreSQL driver; batch UPSERT; `UNNEST`-based set operations |
| ClickHouse client | `ClickHouse.Client` NuGet v7.x | C# ClickHouse ADO.NET driver; HTTP batch INSERT; `RowBinary` format |
| Scheduling | Amazon EventBridge Scheduler | Aggregate Refresh Lambda cron (15 min school / 30 min district / nightly state) |
| Observability | AWS CloudWatch | Metrics: `TRS/API/ClickHouseFallback`, `TRS/CDC/SchemaError`, `TRS/Scores/StaleResendDiscarded`, `TRS/RTS/ClickHouseWriteFailure`; Alarms on DLQ depth |

### 18.1 Key NuGet Packages

```xml
<PackageReference Include="Npgsql"                       Version="8.*" />
<PackageReference Include="ClickHouse.Client"            Version="7.*" />
<PackageReference Include="Amazon.Lambda.Core"           Version="2.*" />
<PackageReference Include="Amazon.Lambda.SQSEvents"      Version="3.*" />
<PackageReference Include="AWSSDK.S3"                    Version="3.*" />
<PackageReference Include="AWSSDK.SQS"                   Version="3.*" />
<PackageReference Include="AWSSDK.SecretsManager"        Version="3.*" />
<PackageReference Include="Amazon.CDK.Lib"               Version="2.*" />
<PackageReference Include="Microsoft.Extensions.Hosting" Version="8.*" />
```

### 18.2 Postgres Batch UPSERT Pattern (C#)

```csharp
// Set-based batch UPSERT using UNNEST — avoids N individual round-trips
var upsertSql = @"
INSERT INTO trs.student_opportunities
    (tenant_id, school_year, opp_key, test_group_id, student_id,
     date_scored, is_aggregate_eligible, overall_scale_score, overall_perf_level, raw_s3_key)
SELECT * FROM UNNEST(
    $1::text[], $2::smallint[], $3::uuid[], $4::text[], $5::int[],
    $6::timestamptz[], $7::boolean[], $8::real[], $9::smallint[], $10::text[])
ON CONFLICT (tenant_id, school_year, opp_key) DO UPDATE SET
    date_scored           = EXCLUDED.date_scored,
    is_aggregate_eligible = EXCLUDED.is_aggregate_eligible,
    overall_scale_score   = EXCLUDED.overall_scale_score,
    overall_perf_level    = EXCLUDED.overall_perf_level
WHERE EXCLUDED.date_scored > trs.student_opportunities.date_scored";
```

### 18.3 ClickHouse Bronze Batch INSERT (C# — Sign-Pair Transformer)

```csharp
// Single HTTP batch INSERT for a -1/+1 pair (or single row for INSERT/DELETE events)
using var client = new ClickHouseClient(clickhouseEndpoint);

var rows = new List<BronzeRow>();

// For an UPDATE: emit both undo (-1) and redo (+1) in the SAME batch
rows.Add(new BronzeRow { OppKey = before.OppKey, Sign = -1, IsDeleted = 0, DateScored = before.DateScored, /* ...before fields */ });
rows.Add(new BronzeRow { OppKey = after.OppKey,  Sign = +1, IsDeleted = 0, DateScored = after.DateScored,  /* ...after fields  */ });

var bulkCopy = new ClickHouseBulkCopy(client)
{
    DestinationTableName = "trs.student_scores_bronze",
    BatchSize = rows.Count  // flush immediately to ensure atomicity of -1/+1 pair
};
await bulkCopy.InitAsync();
await bulkCopy.WriteToServerAsync(rows);
// Do NOT advance PeerDB LSN checkpoint until this INSERT succeeds
```

---

## Implementation Files

Detailed implementation specifications are maintained separately in the `implementation/` folder:

| File | Component |
|------|-----------|
| [implementation/Score_Processor_Lambda.md](implementation/Score_Processor_Lambda.md) | Score Processor Lambda (Postgres-only write; idempotency; rescore path) |
| [implementation/Score_Reject_Handling.md](implementation/Score_Reject_Handling.md) | Score file rejection: unknown config, validation failures, data conflicts; S3 rejection bucket; replay runbook |
| [implementation/CDC_SignPair_Transformer.md](implementation/CDC_SignPair_Transformer.md) | C# Sign-Pair Transformer Fargate service (Bronze INSERT logic; error handling) |
| [implementation/RTS_Sync_Lambda.md](implementation/RTS_Sync_Lambda.md) | RTS Membership Sync Lambda (Aurora primary; ClickHouse mirror) |
| [implementation/API_Lambda.md](implementation/API_Lambda.md) | API Lambda (three-tier fallback; scope routing; report_cache; query logic) |

---

*Document maintained by: Principal Software Architect*
*Last updated: 2026-03-03*
*Supersedes: TRS_Design_Document_v3.md*
