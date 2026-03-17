# Teacher Reporting System (TRS) — Engineering Design Document

**Author:** Principal Software Engineer
**Date:** 2026-03-17
**Version:** v6
**Audience:** Engineering team;

> **Document purpose:** This is the authoritative design document for the TRS analytical
> reporting pipeline. Aurora PostgreSQL is the primary source of truth for all data. ClickHouse
> serves aggregation-only queries via a CDC-driven Medallion Architecture (Bronze → Silver).
> Aggregation queries JOIN Silver against current membership mirrors at query time — there is no
> pre-aggregated Gold layer. Postgres provides full query fallback at all times — ClickHouse is
> optional by design.

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
10. [Resilience & Failure Modes](#10-resilience--failure-modes)
11. [Scale & Capacity](#11-scale--capacity)
12. [Cost Analysis](#12-cost-analysis)
13. [Key Engineering Decisions](#13-key-engineering-decisions)
14. [Operational Prerequisites](#14-operational-prerequisites)
15. [MVP Scope & Future Enhancements](#15-mvp-scope--future-enhancements)
16. [Tech Stack Reference](#16-tech-stack-reference)

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
| ClickHouse schema | Medallion: Bronze → Silver (`VersionedCollapsingMergeTree` → `AggregatingMergeTree`) |
| Rescore in ClickHouse | Sign-Pair (-1/+1) atomic correction in Bronze; `argMaxMerge(latest_score, date_scored)` in Silver always resolves to the latest score |
| Score CDC transport | PeerDB on Fargate — WAL replication slot; before+after images → Sign-Pair Transformer |
| Membership CDC transport | PeerDB on Fargate — direct CDC from `school_student` / `district_student` → ClickHouse membership mirrors |
| ClickHouse sizing | Single `r6i.2xlarge` (aggregation-only, no replica) |
| Aggregate caching | Postgres `report_cache` table (keyed by scope/tenant/year/test) |
| Aggregate computation | ClickHouse Silver + membership mirror JOIN at query time → `report_cache` |
| Fallback when ClickHouse is down | Tier 1: Postgres `report_cache` → Tier 3: Postgres MV / live |
| Multi-enrollment attribution | Membership mirror `JOIN FINAL` at query time (`ReplacingMergeTree` supports N rows per student) |

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
│                                  TRS — Architecture v6                                        │
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
│                                               │  • roster_student / school_student        │  │
│                                               │  • district_student / district_school     │  │
│                                               │  • teacher_roster / students              │  │
│                                               │  • tenants / test_aliases / test_keys     │  │
│                                               │  • test_alias_groups / embargo_roles     │  │
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
│  │  [Bronze]                    [Silver]                                                    │  │
│  │  student_scores_bronze  ──▶  student_scores_silver  (AggregatingMT)                     │  │
│  │  VersionedCollapsingMT       (latest per student,    student_component_scores_silver     │  │
│  │  (raw CDC sign pairs)         use_for_aggregation)   (AggregatingMT)                     │  │
│  │  sign=-1 (undo)                                                                          │  │
│  │  sign=+1 (redo)              membership_school_mirror  (ReplacingMT)                     │  │
│  │                              membership_district_mirror (ReplacingMT)                    │  │
│  │                              student_attributes (ReplacingMT)                            │  │
│  └──────────────────────────────────────────────────────────┬──────────────────────────────┘  │
│                                                              │                                 │
│                                  Aggregate Refresh Lambda (nightly + on-demand)               │
│                                  Queries Silver + membership JOIN → writes to Postgres report_cache            │
│                                                              │                                 │
│  ┌─────────────┐  ┌────────────────────┐  ┌────────────────▼───────────────────────────────┐ │
│  │ CloudFront  │  │  API Gateway       │  │  Lambda: API                                    │ │
│  │ + React SPA │─▶│  + Lambda Auth     │─▶│  Three-tier fallback:                           │ │
│  └─────────────┘  │  (RH SSO JWT)      │  │  [1] Postgres report_cache          (<5ms)       │ │
│                   └────────────────────┘  │  [2] ClickHouse Silver + JOIN    (<250ms school; <2s district)  │ │
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
| ClickHouse Single Node | EC2 `r6i.2xlarge` (aggregation-only) | Medallion Bronze/Silver layers; serves school/district aggregate queries via Silver + membership JOIN at query time; rebuilt from Postgres on total loss |
| Lambda: Aggregate Refresh | AWS Lambda (.NET 8 / C#) | Scheduled; queries ClickHouse Silver + membership mirror JOIN; writes results to Postgres `report_cache`; falls back to Postgres MV refresh if ClickHouse unavailable |
| Lambda: API | AWS Lambda (.NET 8 / C#) | REST API; three-tier fallback: `report_cache` → ClickHouse Silver + membership JOIN → Postgres MV/live |
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
| **Aggregate speed** | For large scopes (50k–300k students), ClickHouse `GROUP BY` is 10–100× faster than Postgres. School queries: <250ms vs. 200–800ms in Postgres. |
| **Columnar scans** | Score data reads only queried columns; Postgres reads full rows. |
| **Optional by design** | System degrades gracefully, not catastrophically, when ClickHouse is unavailable. |
| **Silver + query-time JOIN** | `AggregatingMergeTree` Silver layer with `argMaxMerge` provides correct per-student deduplication; membership JOIN at query time guarantees accurate multi-enrollment attribution regardless of when scores arrived or transfers occurred. |

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

Both rows are inserted in a **single HTTP batch** to ClickHouse. `VersionedCollapsingMergeTree` collapses the pair during background merges. The Silver layer `argMaxMerge(latest_score, date_scored)` always resolves to the latest score, producing correct aggregates when queried with the membership JOIN.

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
    "testalias_id": "cp1-g5-ela-2026",
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
    "testalias_id": "cp1-g5-ela-2026",
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

#### UPDATE Event (`use_for_aggregation` Flag Toggle)

When Score Processor sets `use_for_aggregation = FALSE` on a superseded retest opportunity, Postgres emits a WAL UPDATE event. The Transformer handles this identically to a rescore UPDATE:

```
Input:  BeforeImage = B (old flag state), AfterImage = A (new flag state)
        Key difference: score fields are UNCHANGED; only use_for_aggregation
        and use_for_aggregation_set_at have changed.

Output: Two Bronze rows in a SINGLE HTTP batch INSERT:
  Row 1 (Undo): (opp_key=B.OppKey, sign=-1, use_for_aggregation=B.UseForAgg,
                  use_for_aggregation_set_at=B.SetAt, date_scored=B.DateScored, ...B fields)
  Row 2 (Redo): (opp_key=A.OppKey, sign=+1, use_for_aggregation=A.UseForAgg,
                  use_for_aggregation_set_at=A.SetAt, date_scored=A.DateScored, ...A fields)
```

In Silver: `argMaxState(use_for_aggregation, use_for_aggregation_set_at)` picks the latest flag state. Using `use_for_aggregation_set_at` (not `date_scored`) as the version key is critical — the score's `date_scored` does **not** change on a flag toggle, so using it as the version key would create an `argMaxState` tie.

**Important:** Component scores do **not** receive their own `use_for_aggregation` flag updates via CDC. The component Silver table inherits selection by JOINing against the overall score Silver at query time. This avoids 10–30 CDC UPDATE events per opp per retest flag change.

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
Silver aggregates include the score back on the next query after T3
```

### 4.4 CDC Event Type Reference

| CDC Event | Before | After | Signs Emitted | `is_deleted` | Bronze Result | Silver Result | Aggregate Query Result |
|---|:---:|:---:|:---:|:---:|---|---|---|
| **INSERT** | ❌ | ✅ | `+1` | `0` | Row added | `argMax` picks new row | Score included in count / avg |
| **UPDATE (rescore)** | ✅ | ✅ | `-1` then `+1` | `0` | Pair collapses → net new score | `argMax` picks T_after row | Latest score only; no double-count |
| **UPDATE (`use_for_aggregation` toggle)** | ✅ | ✅ | `-1` then `+1` | `0` | Pair collapses → net new flag state | `argMax(use_for_aggregation, use_for_aggregation_set_at)` picks latest flag | Opp included or excluded per updated flag |
| **DELETE** | ✅ | ❌ | `-1` only | `1` | Collapses original `+1` | `is_deleted=1` wins | Score excluded from aggregates |
| **DELETE → re-INSERT** | ✅→✅ | ❌→✅ | `-1` then `+1` | `1` then `0` | Net zero then re-added | `is_deleted=0` wins at T3 | Score re-included after T3 |

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
| `TestAliasId` | Composite: `{TestFamily}-{Subject}-{Grade}-{Year}-{Attempt}` | `checkpoint1-ela-grade1-2025-attempt1` |
| `TestGroupName` | `CheckPoint1` |
| `TestName` | `Grade 5 ELA CheckPoint1` |
| `Subject` | Subject area | `ELA`, `Math`, `Science` |
| `Grade` | Grade level | `Grade1`, `Grade2` |
| `SchoolYear` | Academic year | `2025`, `2026` |

**TestKey vs TestAliasId:** In 99%+ of cases these are identical. In rare multi-variant cases (online + paper), `TestKey` adds a variant suffix while `TestAliasId` groups all variants.

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
| Original score | `uuid-A` | T1 | INSERT new row | CDC INSERT: Bronze `sign=+1`; Silver accumulates row |
| **Rescore** — same sitting | `uuid-A` | T2 > T1 | ON CONFLICT → UPDATE (T2 wins) | CDC UPDATE: Bronze `-1@T1` + `+1@T2`; Silver `argMax` picks T2 score; no double-count |
| **Retest** — student retakes | `uuid-B` (new) | T3 | INSERT new row; Score Processor sets `use_for_aggregation=TRUE` on new opp, `FALSE` on prior opp(s) in same transaction | CDC INSERT for new opp + CDC UPDATE(s) for prior opp(s); Silver reflects `use_for_aggregation` flag per opp; only the selected opp contributes to aggregates |
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

On rescore, the Lambda recomputes the flag. If the flag changes, the Aurora UPDATE triggers a CDC event → Sign-Pair Transformer → Bronze insert with the corrected `is_aggregate_eligible` value. Silver `HAVING is_eligible = 1` propagates the update automatically on the next query.

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

    testalias_id          TEXT         NOT NULL,
    test_key               TEXT         NOT NULL,
    test_event             TEXT,
    student_id             INTEGER      NOT NULL,
    opp_status             TEXT         NOT NULL,
    condition_code         TEXT,
    tested_date            DATE,
    date_scored            TIMESTAMPTZ  NOT NULL,   -- version key; must be millisecond precision

    is_aggregate_eligible  BOOLEAN      NOT NULL DEFAULT FALSE,

    use_for_aggregation        BOOLEAN      NOT NULL DEFAULT TRUE,
    use_for_aggregation_set_at TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    -- use_for_aggregation = TRUE  → this opp is included in aggregates
    -- use_for_aggregation = FALSE → superseded by a later retest; excluded from aggregates
    -- use_for_aggregation_set_at  → version key; updated to NOW() whenever flag changes

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
    ON trs.student_opportunities (tenant_id, school_year, testalias_id, student_id)
    WHERE is_aggregate_eligible = TRUE;

-- Aggregate-eligible + selected opp (used by Silver query HAVING and Lambda batch)
CREATE INDEX idx_so_aggregation
    ON trs.student_opportunities (tenant_id, school_year, testalias_id, student_id)
    WHERE is_aggregate_eligible = TRUE AND use_for_aggregation = TRUE;

-- Full per-student score list (includes ineligible)
CREATE INDEX idx_so_student
    ON trs.student_opportunities (tenant_id, school_year, testalias_id, student_id);
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

    testalias_id         TEXT         NOT NULL,
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
       (tenant_id, school_year, testalias_id, student_id, component_type)
    WHERE is_aggregate_eligible = TRUE;
```

### 6.3 Membership & Config Tables

| Table | Purpose |
|-------|---------|
| `tenants` | Tenant / state client registry |
| `test_aliases` | One row per alias — test family, subject, grade, school year, embargo |
| `test_keys` | One row per delivery-variant key → maps to parent `testalias_id` |
| `test_alias_groups` | Sequences aliases within a multi-opportunity group (opp1=1, opp2=2) |
| `test_alias_standards` | Standards aligned to a test alias; `standard_id` matches `component_id` in component scores |
| `standard_performance_levels` | Per-level title and description for each standard (levels 1–4) |
| `test_alias_measures` | Which score measures to display per alias (scaleScore, rawScore, lexileScore, quantileScore, performanceLevel) with labels and descriptions |
| `test_alias_perf_levels` | Overall test-level performance level bands with score ranges; `level` matches `overall_perf_level` in student scores |
| `embargo_roles` | Per-tenant roles that grant embargo visibility (matched against JWT claims) |
| `roster_student` | RTS mirror — current roster ↔ student membership |
| `school_student` | RTS mirror — current school ↔ student membership |
| `district_student` | RTS mirror — current district ↔ student membership |
| `district_school` | RTS mirror — district ↔ school relationships |
| `teacher_roster` | RTS mirror — teacher ↔ roster assignments |

> **Note:** There are no standalone entity tables for `tenants`, `schools`, `districts`, `teachers`, or `rosters`. `tenant_id` is a plain string column validated via the SSO JWT. Roster/school/district/teacher entities are tracked only through their RTS membership mirrors above. Standards and their performance levels are stored in `trs.test_alias_standards` and `trs.standard_performance_levels`. `student_attributes` is a ClickHouse table (see §7).

#### `trs.test_aliases`

One row per `testalias_id`. Holds all config-level attributes shared across delivery variants.

```sql
CREATE TABLE trs.test_aliases (
    tenant_id           TEXT        NOT NULL,
    testalias_id        TEXT        NOT NULL,   -- e.g. checkpoint1-grade3-ela-2026-opp1

    test_family         TEXT        NOT NULL,   -- e.g. checkpoint1
    subject             TEXT        NOT NULL,   -- e.g. ELA
    grade               TEXT        NOT NULL,   -- e.g. grade3
    school_year         SMALLINT    NOT NULL,   -- e.g. 2026
    testalias_name      TEXT        NOT NULL,   -- display name

    -- Embargo / release date: these are the same concept.
    -- NULL             = never embargoed; test results are visible to all roles.
    -- Future timestamp = embargoed (unreleased); auto-lifts when clock passes it.
    -- Past timestamp   = embargo has lifted; results are released.
    -- Derivations: is_released ≡ (embargo_until IS NULL OR embargo_until <= NOW())
    --              release_date ≡ embargo_until::DATE
    embargo_until       TIMESTAMPTZ NULL DEFAULT NULL,

    retest_selection_rule TEXT NULL
        CHECK (retest_selection_rule IN ('latest_date', 'highest_score')),
    -- NULL = retests not permitted for this test alias
    -- 'latest_date'   = use the most recently dated opportunity
    -- 'highest_score' = use the opportunity with the highest overall_scale_score

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (tenant_id, testalias_id)
);

-- Efficient lookup for the API embargo pre-check
CREATE INDEX idx_test_aliases_embargo_lookup
    ON trs.test_aliases (tenant_id, testalias_id)
    WHERE embargo_until IS NOT NULL;

-- Efficient scan to find all currently embargoed tests (used by ops tooling)
CREATE INDEX idx_test_aliases_embargo
    ON trs.test_aliases (tenant_id, embargo_until)
    WHERE embargo_until IS NOT NULL;
```

#### `trs.test_keys`

One row per delivery-variant `test_key` (online, paper, braille, etc.). Points to its parent alias.

```sql
CREATE TABLE trs.test_keys (
    tenant_id       TEXT        NOT NULL,
    test_key        TEXT        NOT NULL,   -- e.g. checkpoint1-grade3-ela-2026-online-opp1
    testalias_id    TEXT        NOT NULL,   -- FK → test_aliases

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (tenant_id, test_key),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);

CREATE INDEX idx_test_keys_alias
    ON trs.test_keys (tenant_id, testalias_id);
```

#### `trs.test_alias_groups`

Sequences multiple aliases for multi-opportunity tests (opp1 → seq 1, opp2 → seq 2).

```sql
CREATE TABLE trs.test_alias_groups (
    tenant_id               TEXT        NOT NULL,
    testalias_group_name    TEXT        NOT NULL,   -- e.g. checkpoint1-grade3-ela-2026
    testalias_id            TEXT        NOT NULL,   -- FK → test_aliases
    sequence                SMALLINT    NOT NULL,   -- 1, 2, …
    sequence_label          TEXT        NOT NULL,   -- e.g. opp1, opp2

    PRIMARY KEY (tenant_id, testalias_group_name, sequence),
    UNIQUE      (tenant_id, testalias_id),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);

CREATE INDEX idx_test_alias_groups_alias
    ON trs.test_alias_groups (tenant_id, testalias_group_name, sequence);
```

#### `trs.test_alias_standards`

One row per standard aligned to a `testalias_id`. The `standard_id` matches the
`component_id` stored in `trs.student_component_scores` when `component_type = 'STANDARD'`.

```sql
CREATE TABLE trs.test_alias_standards (
    tenant_id       TEXT    NOT NULL,
    testalias_id    TEXT    NOT NULL,   -- FK → test_aliases
    standard_id     TEXT    NOT NULL,   -- e.g. CCSS.ELA-LITERACY.RL.3.1
    description     TEXT    NOT NULL,

    PRIMARY KEY (tenant_id, testalias_id, standard_id),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);
```

#### `trs.standard_performance_levels`

One row per performance level per standard. Descriptions are standard-specific even when
level titles (Below Basic / Basic / Proficient / Advanced) repeat across standards.

```sql
CREATE TABLE trs.standard_performance_levels (
    tenant_id       TEXT        NOT NULL,
    testalias_id    TEXT        NOT NULL,
    standard_id     TEXT        NOT NULL,
    level           SMALLINT    NOT NULL,   -- 1, 2, 3, 4  (matches component_perf_level in scores)
    title           TEXT        NOT NULL,   -- e.g. Below Basic, Basic, Proficient, Advanced
    description     TEXT        NOT NULL,
    min_score       REAL        NOT NULL,   -- inclusive lower bound; REAL to match overall_scale_score
    max_score       REAL        NOT NULL,   -- inclusive upper bound of the scale-score range for this level

    PRIMARY KEY (tenant_id, testalias_id, standard_id, level),
    FOREIGN KEY (tenant_id, testalias_id, standard_id)
        REFERENCES trs.test_alias_standards (tenant_id, testalias_id, standard_id)
);
```

#### `trs.test_alias_measures`

One row per measure type per alias. Controls which score measures are displayed in reports
and provides labels and descriptions shown to users.

`min_score` / `max_score` are NULL for `performanceLevel` — its score ranges are defined
per-level in `trs.test_alias_perf_levels`.

```sql
CREATE TABLE trs.test_alias_measures (
    tenant_id       TEXT        NOT NULL,
    testalias_id    TEXT        NOT NULL,   -- FK → test_aliases
    measure_type    TEXT        NOT NULL,   -- scaleScore | rawScore | lexileScore | quantileScore
                                            -- | percentCorrect | percentile | performanceLevel
    show            BOOLEAN     NOT NULL DEFAULT TRUE,
    label           TEXT        NOT NULL,
    description     TEXT        NOT NULL,
    min_score       REAL        NULL,       -- NULL for performanceLevel; REAL to match overall_scale_score
    max_score       REAL        NULL,       -- NULL for performanceLevel

    PRIMARY KEY (tenant_id, testalias_id, measure_type),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);
```

#### `trs.test_alias_perf_levels`

Overall test-level performance levels. Parallel to `trs.standard_performance_levels` but
keyed to the alias only — these describe the test's overall score bands, not per-standard.
`level` matches `overall_perf_level` in `trs.student_opportunities`.

```sql
CREATE TABLE trs.test_alias_perf_levels (
    tenant_id       TEXT        NOT NULL,
    testalias_id    TEXT        NOT NULL,   -- FK → test_aliases
    level           SMALLINT    NOT NULL,   -- 1, 2, 3, 4  (matches overall_perf_level in student_opportunities)
    title           TEXT        NOT NULL,   -- e.g. Below Basic, Basic, Proficient, Advanced
    description     TEXT        NOT NULL,
    min_score       REAL        NOT NULL,   -- inclusive lower bound; REAL to match overall_scale_score
    max_score       REAL        NOT NULL,   -- inclusive upper bound

    PRIMARY KEY (tenant_id, testalias_id, level),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);
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
)
PARTITION BY LIST (tenant_id);

-- Direct school ↔ student (replaces school_roster_members)
CREATE TABLE trs.school_student (
    tenant_id   TEXT     NOT NULL,
    school_id   TEXT     NOT NULL,
    student_id  INTEGER  NOT NULL,
    active      BOOLEAN  NOT NULL DEFAULT TRUE,
    synced_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, school_id, student_id)
)
PARTITION BY LIST (tenant_id);

-- Direct roster ↔ student (replaces roster_members for CDC path)
CREATE TABLE trs.roster_student (
    tenant_id   TEXT     NOT NULL,
    roster_id   TEXT     NOT NULL,
    student_id  INTEGER  NOT NULL,
    active      BOOLEAN  NOT NULL DEFAULT TRUE,
    synced_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (tenant_id, roster_id, student_id)
)
PARTITION BY LIST (tenant_id);

-- Explicit district ↔ school relationship
CREATE TABLE trs.district_school (
    tenant_id   TEXT     NOT NULL,
    district_id TEXT     NOT NULL,
    school_id   TEXT     NOT NULL,
    PRIMARY KEY (tenant_id, district_id, school_id)
)
PARTITION BY LIST (tenant_id);

-- Explicit teacher ↔ roster relationship
CREATE TABLE trs.teacher_roster (
    tenant_id   TEXT     NOT NULL,
    teacher_id  TEXT     NOT NULL,
    roster_id   TEXT     NOT NULL,
    PRIMARY KEY (tenant_id, teacher_id, roster_id)
)
PARTITION BY LIST (tenant_id);
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
Written by the Aggregate Refresh Lambda (computed via ClickHouse Silver + membership JOIN or Postgres MV).
Read first by the API Lambda (Tier 1 of the fallback chain).

```sql
CREATE TABLE trs.report_cache (
    -- Key format: "{scope}#{tenant_id}#{scope_id}#{school_year}#{testalias_id}"
    -- Examples:
    --   "school#tx#s-789#2026#cp1-g5-ela"
    --   "district#tx#d-456#2026#cp1-g5-ela"
    --   "state#tx#tx#2026#cp1-g5-ela"
    cache_key    TEXT        PRIMARY KEY,
    payload      JSONB       NOT NULL,  -- full aggregate result
    computed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ NOT NULL,
    computed_by  TEXT        NOT NULL   -- 'clickhouse_silver' | 'postgres_mv' | 'postgres_live' | 'nightly_job'
);

CREATE INDEX idx_report_cache_expiry ON trs.report_cache (expires_at);
-- pg_cron cleanup: DELETE FROM trs.report_cache WHERE expires_at < now();
```

**Cache TTLs:**

| Scope | TTL | Rationale |
|-------|-----|-----------|
| Roster | No cache — always live | ~30 students; <5 ms in both Postgres and ClickHouse |
| School | 15 minutes | Fast ClickHouse Silver + JOIN query; short TTL keeps it fresh during test windows |
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
    ss.testalias_id,
    ss_m.school_id,
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)      AS pl_1,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)      AS pl_2,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)      AS pl_3,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)      AS pl_4,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)      AS pl_5,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 6)      AS pl_6,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 7)      AS pl_7,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 8)      AS pl_8,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 9)      AS pl_9,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 10)     AS pl_10,
    AVG(ss.overall_scale_score)                            AS avg_scale_score,
    AVG(ss.overall_raw_score)                              AS avg_raw_score,
    AVG(ss.overall_standard_error)                         AS avg_standard_error,
    NOW()                                                  AS refreshed_at
FROM trs.student_opportunities ss
JOIN trs.school_student ss_m
    ON ss.tenant_id = ss_m.tenant_id AND ss.student_id = ss_m.student_id
WHERE ss.is_aggregate_eligible = TRUE AND ss.use_for_aggregation = TRUE AND ss_m.active = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.testalias_id, ss_m.school_id;

CREATE UNIQUE INDEX uq_mv_school_overall
    ON trs.mv_school_overall (tenant_id, school_year, testalias_id, school_id);

-- District-level overall aggregate MV
CREATE MATERIALIZED VIEW trs.mv_district_overall AS
SELECT
    ss.tenant_id,
    ss.school_year,
    ss.testalias_id,
    ds.district_id,
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)      AS pl_1,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)      AS pl_2,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)      AS pl_3,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)      AS pl_4,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)      AS pl_5,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 6)      AS pl_6,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 7)      AS pl_7,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 8)      AS pl_8,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 9)      AS pl_9,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 10)     AS pl_10,
    AVG(ss.overall_scale_score)                            AS avg_scale_score,
    AVG(ss.overall_raw_score)                              AS avg_raw_score,
    AVG(ss.overall_standard_error)                         AS avg_standard_error,
    NOW()                                                  AS refreshed_at
FROM trs.student_opportunities ss
JOIN trs.district_student ds
    ON ss.tenant_id = ds.tenant_id AND ss.student_id = ds.student_id
WHERE ss.is_aggregate_eligible = TRUE AND ss.use_for_aggregation = TRUE AND ds.active = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.testalias_id, ds.district_id;

CREATE UNIQUE INDEX uq_mv_district_overall
    ON trs.mv_district_overall (tenant_id, school_year, testalias_id, district_id);
```

**Refresh schedule (Lambda cron):**

| MV | Frequency |
|----|-----------|
| `mv_school_overall` | Every 15 min during test window; nightly off-window |
| `mv_district_overall` | Every 30 min during test window; nightly off-window |

### 6.7 Postgres Materialized Views — Component Scores (Current-Year Fallback Only)

At 1.65 billion rows/year (TX scale), a single Postgres MV spanning all years and all component types is **not feasible**. The following pragmatic constraints apply:

- **Current year only.** The MV covers only the active school year. Historical component aggregates at school/district scope are served from `report_cache` (populated by ClickHouse Silver + JOIN queries); live Postgres computation of historical component aggregates is not supported.
- **Explicit partition pruning.** The MV must join `student_component_scores` to `school_student` with `tenant_id` and `school_year` predicates so the planner can prune to the current-year leaf partition.
- **Scoped by type.** If the 15-minute refresh SLA cannot be met with a single MV, create separate MVs per `component_type` (`STANDARD`, `RC`).

```sql
-- Current-year school-level STANDARD aggregate (Tier 3 fallback for component queries)
CREATE MATERIALIZED VIEW trs.mv_school_standards_current AS
SELECT
    scs.tenant_id,
    scs.school_year,
    scs.testalias_id,
    ss_m.school_id,
    scs.component_id,
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE scs.perf_level = 1)             AS pl_1,
    COUNT(*) FILTER (WHERE scs.perf_level = 2)             AS pl_2,
    COUNT(*) FILTER (WHERE scs.perf_level = 3)             AS pl_3,
    COUNT(*) FILTER (WHERE scs.perf_level = 4)             AS pl_4,
    COUNT(*) FILTER (WHERE scs.perf_level = 5)             AS pl_5,
    COUNT(*) FILTER (WHERE scs.perf_level = 6)             AS pl_6,
    COUNT(*) FILTER (WHERE scs.perf_level = 7)             AS pl_7,
    COUNT(*) FILTER (WHERE scs.perf_level = 8)             AS pl_8,
    COUNT(*) FILTER (WHERE scs.perf_level = 9)             AS pl_9,
    COUNT(*) FILTER (WHERE scs.perf_level = 10)            AS pl_10,
    NOW()                                                  AS refreshed_at
FROM trs.student_component_scores scs
JOIN trs.school_student ss_m
    ON scs.tenant_id = ss_m.tenant_id AND scs.student_id = ss_m.student_id
JOIN trs.student_opportunities so
    ON scs.tenant_id = so.tenant_id AND scs.opp_key = so.opp_key
WHERE 1=1
    --and scs.school_year          = 2026          -- Partition pruning — update each school year
  AND scs.component_type        = 'STANDARD'    -- Separate MV for RC if needed
  AND scs.is_aggregate_eligible  = TRUE
  AND so.use_for_aggregation     = TRUE
  AND ss_m.active                = TRUE
GROUP BY scs.tenant_id, scs.school_year, scs.testalias_id, ss_m.school_id, scs.component_id;

CREATE UNIQUE INDEX uq_mv_school_standards_current
    ON trs.mv_school_standards_current (tenant_id, school_year, testalias_id, school_id, component_id);
```

> **Guardrail — no district-wide live component scan on Postgres.** Because raw `GROUP BY` over component scores at district scope risks exhausting Aurora connection/IO budget, the Tier 3b live query path for component scores is **restricted to school scope only**. District-scope component requests that cannot be served from `report_cache` or ClickHouse Silver + JOIN are **rejected with HTTP 503** (or served from a long-TTL stale cache entry) rather than falling through to a live Postgres scan.

**Refresh schedule:**

| MV | Frequency |
|----|-----------|
| `mv_school_standards_current` | Every 15 min during test window; nightly off-window |

---

### 6.8 `trs.score_ingest_rejections` — Rejection Audit & Replay Tracking

When a score file cannot be processed, Score Processor inserts a row here for audit and replay.

```sql
CREATE TABLE trs.score_ingest_rejections (
    id                  BIGSERIAL    PRIMARY KEY,
    tenant_id           TEXT         NOT NULL,
    rejection_reason    TEXT         NOT NULL
        CHECK (rejection_reason IN ('UNKNOWN_TEST_KEY', 'VALIDATION_FAILED', 'DATA_CONFLICT')),
    source_s3_key       TEXT         NOT NULL,   -- original path in trs-raw bucket
    rejected_s3_key     TEXT         NOT NULL,   -- copy path in trs-rejected bucket
    test_key            TEXT,                     -- NULL if file unparseable
    testalias_id        TEXT,                     -- NULL if test_key unresolvable
    error_detail        TEXT         NOT NULL,    -- human-readable rejection reason
    rejected_at         TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    resolved            BOOLEAN      NOT NULL DEFAULT FALSE,
    resolved_at         TIMESTAMPTZ,
    resolved_by         TEXT,                     -- operator ID or 'replay_automation'
    replay_queued_at    TIMESTAMPTZ               -- set when file is re-queued to SQS
);

CREATE INDEX idx_sir_unresolved ON trs.score_ingest_rejections (tenant_id, rejected_at)
    WHERE resolved = FALSE;
CREATE INDEX idx_sir_test_key   ON trs.score_ingest_rejections (tenant_id, test_key)
    WHERE resolved = FALSE;
```

**Replay procedure:** Query `WHERE resolved = FALSE` to find affected files → re-send `source_s3_key` to SQS Score Queue → set `replay_queued_at` → idempotency check in Score Processor prevents duplicate inserts.

See [implementation/Score_Reject_Handling.md](implementation/Score_Reject_Handling.md) for full C# implementation and ops runbook.

---

## 7. Database Schema — ClickHouse Medallion Architecture

ClickHouse mirrors score data from Postgres via the CDC pipeline and serves fast aggregation queries. It is not the source of truth. If ClickHouse is unavailable, Postgres serves all queries.

```
[Bronze Layer]                      [Silver Layer]
VersionedCollapsingMergeTree   →    AggregatingMergeTree
Raw CDC sign-pair stream             Deduplicated per-student
(-1/+1 rescore events)               opportunity state + use_for_aggregation flag
                                     Queried at runtime + membership JOIN → aggregates
```

### 7.1 Layer 1: Bronze (Raw CDC Stream)

The landing zone for all CDC events from the Sign-Pair Transformer. Stores raw sign-paired rows.

```sql
CREATE TABLE trs.student_scores_bronze (
    tenant_id              String,
    school_year            UInt16,
    testalias_id          String,
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

    use_for_aggregation        UInt8         DEFAULT 1,   -- 0 when superseded by a later retest
    use_for_aggregation_set_at DateTime64(3) DEFAULT now64(3), -- version key for argMax tie-breaking

    overall_scale_score    Nullable(Float32),
    overall_raw_score      Nullable(Int32),
    overall_perf_level     Nullable(UInt8),
    overall_standard_error Nullable(Float32),

    ingested_at            DateTime64(3) DEFAULT now64(3)
)
ENGINE = VersionedCollapsingMergeTree(sign, date_scored)
PARTITION BY (tenant_id, school_year)
ORDER BY (tenant_id, school_year, testalias_id, student_id, opp_key, date_scored);
```

**Key properties:**
- `VersionedCollapsingMergeTree(sign, date_scored)` collapses `-1/+1` pairs during background merges when both `sign` values exist for the same `(opp_key, date_scored)`.
- A lone `-1` (DELETE undo) collapses against the original `+1` INSERT, physically removing the row.
- **Never queried directly by the API** — feeds Silver via Materialized View.

### 7.2 Layer 2: Silver (Deduplicated Per-Student State)

The Silver layer collapses the Bronze sign-pair stream into a single authoritative version per student opportunity. Used for per-student roster queries in ClickHouse (optional; Postgres is the primary path for roster reads).

```sql

CREATE TABLE trs.student_scores_silver (
    tenant_id         String,
    school_year       UInt16,
    testalias_id     String,
    student_id        Int32,
    opp_key           UUID,
    -- Intermediate states — resolved at read time with -Merge combinators
    latest_score        AggregateFunction(argMax, Float32,      DateTime64(3)),
    latest_perf_level   AggregateFunction(argMax, UInt8,        DateTime64(3)),
    is_eligible         AggregateFunction(argMax, UInt8,        DateTime64(3)),
    last_updated        AggregateFunction(max,    DateTime64(3)),
    -- Retest selection flag: argMax keyed on use_for_aggregation_set_at (not date_scored)
    -- because the flag changes via UPDATE without changing date_scored
    use_for_aggregation AggregateFunction(argMax, UInt8,        DateTime64(3)),
    -- Soft-delete sentinel: argMax on date_scored → latest event always wins
    -- DELETE sets is_deleted=1 with newest timestamp → wins until re-insert with newer T
    is_deleted          AggregateFunction(argMax, UInt8,        DateTime64(3))
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, testalias_id, student_id, opp_key);

CREATE MATERIALIZED VIEW trs.student_scores_silver_mv
TO trs.student_scores_silver AS
SELECT
    tenant_id, school_year, testalias_id, student_id, opp_key,
    argMaxState(overall_scale_score,           date_scored)              AS latest_score,
    argMaxState(overall_perf_level,            date_scored)              AS latest_perf_level,
    argMaxState(is_aggregate_eligible,         date_scored)              AS is_eligible,
    maxState(date_scored)                                                 AS last_updated,
    argMaxState(use_for_aggregation,           use_for_aggregation_set_at) AS use_for_aggregation,
    argMaxState(is_deleted,                    date_scored)              AS is_deleted
FROM trs.student_scores_bronze
GROUP BY tenant_id, school_year, testalias_id, student_id, opp_key;
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
WHERE tenant_id = ? AND school_year = ? AND testalias_id = ?
  AND student_id IN (?)
GROUP BY student_id, opp_key
HAVING argMaxMerge(is_deleted) = 0;
```

> **Note on Silver role:** Because Postgres is the primary store for per-student data, the roster read path goes directly to Aurora in most cases. Silver is retained in ClickHouse for future features requiring ClickHouse-side deduplication (e.g., demographic slice reports at roster scope).

### 7.3 Silver — Component Scores

Component scores are high-volume (up to **1.65 billion rows/year at Texas scale**). They follow the same Sign-Pair Bronze CDC pattern as overall scores, replicated through a dedicated Bronze table and accumulated into a dedicated Silver table.

**Why no `use_for_aggregation` flag on component rows?** Component scores inherit retest selection from the parent opportunity via a JOIN against `student_scores_silver` at query time. This avoids 10–30 CDC UPDATE events per opp per retest flag change.

#### Bronze: `student_component_scores_bronze`

PeerDB replicates `student_component_scores` CDC events through the Sign-Pair Transformer into this table. `REPLICA IDENTITY FULL` must be set (see §14).

```sql
CREATE TABLE trs.student_component_scores_bronze (
    tenant_id             String,
    school_year           UInt16,
    testalias_id          String,
    student_id            Int32,
    opp_key               UUID,
    component_type        LowCardinality(String),  -- STANDARD | RC | WRITING_DIM
    component_id          String,

    sign                  Int8,             -- -1 (undo) or +1 (redo)
    is_deleted            UInt8,            -- 1 on DELETE events; 0 otherwise

    perf_level            Nullable(UInt8),
    scale_score           Nullable(Float32),
    is_aggregate_eligible UInt8 DEFAULT 0,
    date_scored           DateTime64(3),

    ingested_at           DateTime64(3) DEFAULT now64(3)
)
ENGINE = VersionedCollapsingMergeTree(sign, date_scored)
PARTITION BY (tenant_id, school_year)
ORDER BY (tenant_id, school_year, testalias_id, student_id, opp_key, component_type, component_id, date_scored);
```

#### Silver: `student_component_scores_silver`

```sql
CREATE TABLE trs.student_component_scores_silver (
    tenant_id          String,
    school_year        UInt16,
    testalias_id       String,
    student_id         Int32,
    opp_key            UUID,
    component_type     LowCardinality(String),
    component_id       String,

    latest_scale_score AggregateFunction(argMax, Float32, DateTime64(3)),
    latest_perf_level  AggregateFunction(argMax, UInt8,   DateTime64(3)),
    is_eligible        AggregateFunction(argMax, UInt8,   DateTime64(3)),
    is_deleted         AggregateFunction(argMax, UInt8,   DateTime64(3))
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, testalias_id, student_id, opp_key, component_type, component_id);

CREATE MATERIALIZED VIEW trs.student_component_scores_silver_mv
TO trs.student_component_scores_silver AS
SELECT
    tenant_id, school_year, testalias_id, student_id, opp_key, component_type, component_id,
    argMaxState(scale_score,           date_scored) AS latest_scale_score,
    argMaxState(perf_level,            date_scored) AS latest_perf_level,
    argMaxState(is_aggregate_eligible, date_scored) AS is_eligible,
    argMaxState(is_deleted,            date_scored) AS is_deleted
FROM trs.student_component_scores_bronze
GROUP BY tenant_id, school_year, testalias_id, student_id, opp_key, component_type, component_id;
```

**Component Score Read Query — School Scope:**

```sql
WITH school_students AS (
    SELECT DISTINCT student_id
    FROM trs.membership_school_mirror FINAL
    WHERE tenant_id = ? AND school_id = ? AND is_deleted = 0
),
selected_opps AS (
    -- inherit use_for_aggregation from overall score Silver
    SELECT tenant_id, student_id, opp_key
    FROM trs.student_scores_silver
    WHERE tenant_id = ? AND school_year = ? AND testalias_id = ?
      AND student_id IN (SELECT student_id FROM school_students)
    GROUP BY tenant_id, student_id, opp_key
    HAVING argMaxMerge(is_deleted)          = 0
       AND argMaxMerge(is_eligible)         = 1
       AND argMaxMerge(use_for_aggregation) = 1
),
component_resolved AS (
    SELECT
        csil.student_id, csil.component_type, csil.component_id,
        argMaxMerge(csil.latest_scale_score) AS scale_score,
        argMaxMerge(csil.latest_perf_level)  AS perf_level,
        argMaxMerge(csil.is_deleted)         AS is_deleted,
        argMaxMerge(csil.is_eligible)        AS is_eligible
    FROM trs.student_component_scores_silver csil
    INNER JOIN selected_opps sel
        ON  csil.tenant_id  = sel.tenant_id
        AND csil.student_id = sel.student_id
        AND csil.opp_key    = sel.opp_key
    WHERE csil.tenant_id = ? AND csil.school_year = ? AND csil.testalias_id = ?
      AND csil.student_id IN (SELECT student_id FROM school_students)
    GROUP BY csil.student_id, csil.component_type, csil.component_id
    HAVING is_deleted = 0 AND is_eligible = 1
)
SELECT component_type, component_id,
       avg(scale_score)         AS avg_scale_score,
       count(*)                 AS students_tested,
       countIf(perf_level = 1)  AS pl_1,
       countIf(perf_level = 2)  AS pl_2,
       countIf(perf_level = 3)  AS pl_3,
       countIf(perf_level = 4)  AS pl_4
FROM component_resolved
GROUP BY component_type, component_id
ORDER BY component_type, component_id;
```

**Performance (school scope, TX scale):** ~6,800 students × 15–20 components = ~100–136k rows touched. Tier 2 live query: 80–250ms. District component queries: 5s timeout gate; HTTP 503 if exceeded. State component aggregates: nightly Lambda only.

---

### 7.4 Silver — School/District Aggregation Query (Overall Scores)

There is no pre-aggregated Gold layer. School and district aggregates are computed by querying Silver + membership JOIN at query time. This query is used by the Aggregate Refresh Lambda (batch, all schools) and by API Tier 2 (single school on cache miss).

```sql
-- Used by Aggregate Refresh Lambda (all schools batch) and API Tier 2 (single school)
SELECT
    m.school_id,
    count(*)                      AS students_tested,
    avg(s.latest_score)           AS avg_score,
    countIf(s.perf_level = 1)     AS pl_1,
    countIf(s.perf_level = 2)     AS pl_2,
    countIf(s.perf_level = 3)     AS pl_3,
    countIf(s.perf_level = 4)     AS pl_4,
    countIf(s.perf_level = 5)     AS pl_5,
    countIf(s.perf_level = 6)     AS pl_6,
    countIf(s.perf_level = 7)     AS pl_7,
    countIf(s.perf_level = 8)     AS pl_8,
    countIf(s.perf_level = 9)     AS pl_9,
    countIf(s.perf_level = 10)    AS pl_10
FROM (
    SELECT
        tenant_id, student_id,
        argMaxMerge(latest_score)         AS latest_score,
        argMaxMerge(latest_perf_level)    AS perf_level,
        argMaxMerge(is_deleted)           AS is_deleted,
        argMaxMerge(is_eligible)          AS is_eligible,
        argMaxMerge(use_for_aggregation)  AS use_for_agg
    FROM trs.student_scores_silver
    WHERE tenant_id = ? AND school_year = ? AND testalias_id = ?
    GROUP BY tenant_id, student_id, opp_key
    HAVING is_deleted = 0 AND is_eligible = 1 AND use_for_agg = 1
) AS s
INNER JOIN (
    SELECT DISTINCT tenant_id, school_id, student_id
    FROM trs.membership_school_mirror FINAL
    WHERE tenant_id = ? AND is_deleted = 0
) AS m ON s.tenant_id = m.tenant_id AND s.student_id = m.student_id
-- Omit WHERE m.school_id = ? in Lambda batch mode (returns all schools in one pass)
GROUP BY m.school_id;
```

**Why query-time JOIN is correct:**
- **Multi-enrollment**: JOIN produces N rows per student; each school aggregate counts the student once.
- **Score before membership**: score sits in Silver; next Lambda run JOINs current membership state.
- **Rescore**: `argMaxMerge` picks the latest score; no denominator inflation.
- **Retest**: `HAVING use_for_agg = 1` selects one opp per student.
- **Transfer**: next Lambda run JOINs current membership — previous school's aggregate drops the student.

---

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
-- Student demographics (Future)
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

| Feature | Bronze | Silver (Overall) | Silver (Component) |
|---|---|---|---|
| **Engine** | `VersionedCollapsingMergeTree` | `AggregatingMergeTree` | `AggregatingMergeTree` |
| **Granularity** | 1 row per CDC event | 1 row per student opportunity | 1 row per student × opp × component |
| **Primary Question** | "What changed and when?" | "Latest score per student?" | "Latest component score?" |
| **API Query Type** | Not queried directly | School/District/State aggregates (JOIN membership at query time) | School/District component aggregates (JOIN overall Silver + membership at query time) |
| **Rescore Handling** | Signs cancel on background merge | `argMax` keeps highest `date_scored` | `argMax` keeps highest `date_scored` |
| **Retest Handling** | Both opps land in Bronze | `HAVING use_for_aggregation = 1` selects one opp per student | Inherited via opp_key JOIN to overall Silver |
| **Membership Attribution** | N/A | `membership_school_mirror FINAL` JOIN at query time | Via overall Silver JOIN |

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
    ├─ 4. Resolve testalias_id from test_key_config cache
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
    └─▶ Silver MV (AggregatingMergeTree) — fires on Bronze INSERT
         (use_for_aggregation aggregated via argMaxState keyed on use_for_aggregation_set_at)
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

  ClickHouse Silver:
    argMaxMerge(latest_score, date_scored) picks T2 score on next query
    No avgState denominator inflation — denominator grows only with student count
    No double-count — student appears once in Silver per opp_key

  Report Cache (Postgres):
    TTL on affected report_cache rows expires (within 5–15 min depending on scope)
    OR: Score Processor notifies SNS → Aggregate Refresh Lambda invalidates specific rows
    Next cache read triggers Aggregate Refresh Lambda → re-queries Silver + membership JOIN → writes updated cache
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
    │    Query Silver + membership JOIN (see §7.4 and §7.3 queries):
    │      School:    overall Silver + membership_school_mirror JOIN → all schools in one pass
    │      District:  overall Silver + membership_district_mirror JOIN → all districts in one pass
    │      Component: component Silver + overall Silver opp_key JOIN + membership JOIN
    │    Upsert results:
    │      INSERT INTO trs.report_cache (cache_key, payload, computed_at, expires_at, computed_by='clickhouse_silver')
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
| `UNKNOWN_TEST_KEY` | `test_key` in the file has no matching row in `trs.test_keys` |
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
[Embargo Check] Does testalias_id have embargo_until > NOW()?
        │ YES (embargoed) — does JWT roles contain 'EMBARGO_VIEWER'?
        │   NO  ──────────────────────────────────────────────────▶  Return 404 Not Found
        │   YES ─────────────────────────────────────────────────▶  Continue to Tier 1
        │ NO (not embargoed)
        ▼
[Tier 1] Postgres report_cache ──── HIT (not expired) ──────────▶  Return result (~3ms)
        │ MISS or expired
        ▼
[Tier 2] ClickHouse healthy? ──── YES ──▶  Query Silver + membership JOIN
        │                                   (school: <250ms; district: <2s)
        │                                   Write to report_cache
        │                                   Return result; servedFrom='clickhouse_silver'
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

ClickHouse health is checked via the in-process circuit breaker (see below) rather than a fresh probe on every request. A OPEN circuit routes immediately to Tier 3.

**Circuit Breaker Pattern:**

The API Lambda implements an in-process circuit breaker with three states (cached per Lambda container):

| State | Behavior | Transition |
|-------|----------|------------|
| **CLOSED** | Normal; Tier 2 queries attempted | → OPEN on probe failure |
| **OPEN** | Skip Tier 2 entirely; go directly to Tier 3 | → HALF_OPEN after 30s TTL |
| **HALF_OPEN** | Allow one probe attempt | → CLOSED on success; → OPEN on failure |

- OPEN TTL: 30 seconds (configurable via `CH_CIRCUIT_OPEN_TTL_SEC` environment variable).
- On first probe failure: immediately transition to OPEN; no retry.
- State is per-container (not shared across containers) — acceptable; each container discovers health independently within 30s.
- CloudWatch metric `TRS/API/CircuitBreakerOpen` emitted on transition to OPEN.
- Alarm: `TRS/API/CircuitBreakerOpen` sustained > 5 minutes → SNS → ops.

**Why 404 and not 403 for embargoed tests?** Returning 403 would confirm to a regular user that the test exists but is restricted. 404 treats the test as non-existent from the caller's perspective, preventing information leakage about unreleased assessments.

### 9.2 Scope-Specific Routing

| Scope | Tier 1 (Cache) | Tier 2 (ClickHouse) | Tier 3 (Postgres) | Cache TTL |
|-------|---------------|---------------------|-------------------|-----------|
| **Roster** | No cache | Silver `argMaxMerge` per student | Aurora live query | N/A — always live |
| **School** | `report_cache` | Silver + `membership_school_mirror` JOIN (§7.4) | `mv_school_overall` → raw live | 15 min |
| **District** | `report_cache` | Silver + `membership_district_mirror` JOIN (§7.4) | `mv_district_overall` → raw live | 5 min |
| **State** | `report_cache` | Silver + membership JOIN (GROUP BY district, all) | `mv_district_overall` GROUP BY → very slow | Until next nightly run |
| **Roster — components** | No cache | *(not applicable — small N)* | Aurora live query on `student_component_scores` | N/A — always live |
| **School — components** | `report_cache` | Component Silver + opp_key JOIN + membership JOIN (§7.3) | `mv_school_standards_current` (current year); long-TTL cache entry (historical) | 15 min |
| **District — components** | `report_cache` | Component Silver + opp_key JOIN + membership JOIN (§7.3) | **No live Postgres fallback** — return HTTP 503 if cache and ClickHouse both unavailable | 15 min |


### 9.5 Embargo Service (in-process cache)

`EmbargoService` is called by the API Lambda immediately after JWT validation, before any cache
or database query. It enforces the embargo visibility gate with two in-process cached lookups:

| Lookup | Source | Cache TTL |
|--------|--------|-----------|
| `embargo_until` for the `testalias_id` | `trs.test_aliases` | 60 s per `(tenant_id, testalias_id)` |
| Permitted embargo role names | `trs.embargo_roles` | 5 min per `tenant_id` |

If the test is currently embargoed and none of the caller's JWT `roles` claims match any row in
`trs.embargo_roles` for the tenant, the service throws `EmbargoException` → **HTTP 404**.

The **test discovery endpoint** (`GET /v1/{tenantId}/tests?schoolYear={Y}`) is the only place
callers learn which `testalias_id`s exist. It applies the same `embargo_roles` check and omits
embargoed entries for non-permitted callers, so regular users never acquire a `testalias_id`
to probe via the aggregate endpoints.

See [implementation/EmbargoService.md](implementation/EmbargoService.md) for the full C#
class, repository SQL queries, test discovery endpoint SQL, and cache invalidation notes.

---




## 10. Resilience & Failure Modes

### 10.1 ClickHouse Instance Reboot

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
ClickHouse Bronze/Silver reaches real-time status
Sign-Pair idempotency ensures duplicate CDC messages (from restart) collapse correctly
API resumes ClickHouse Silver + membership JOIN queries
```

### 10.2 PeerDB Fargate Task Restart

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

### 10.3 Total EC2 Loss (Disaster Recovery)

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
  Silver MV → Silver table (overall scores)
  membership_school_mirror (ReplacingMergeTree)
  membership_district_mirror (ReplacingMergeTree)
  component Bronze (VersionedCollapsingMergeTree)
  component Silver MV → component Silver table
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
ClickHouse is current → API resumes ClickHouse Silver + membership JOIN queries
```

> Throughout the entire multi-hour rebuild, users continue seeing reports via the Postgres fallback.
> The rebuild is fully transparent at the API layer.

### 10.4 Resilience Summary

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

## 11. Scale & Capacity

### 11.1 State-Level Student Volumes

| State | Total Students | Largest District | Largest School |
|-------|----------------|-----------------|----------------|
| Texas | 5,543,751 | 189,934 (Houston ISD) | 6,798 (Allen HS) |
| Virginia | 1,259,958 | 179,858 (Fairfax County) | 5,100 (Alexandria City) |
| Illinois | 1,850,074 | 321,666 (Chicago PS) | 4,300 (Lane Tech) |
| Indiana | 1,009,888 | 31,000 (Indianapolis PS) | 5,400 (Carmel HS) |

### 11.2 Annual Row Counts (Texas Scale)

| Table | Rows / Year | Postgres Storage (uncompressed) | ClickHouse Bronze (compressed) |
|-------|------------|--------------------------------|-------------------------------|
| `student_opportunities` | ~55M | ~22 GB | ~2 GB |
| `student_component_scores` | ~1.65B | ~660 GB | ~9–13 GB |
| `student_scores_bronze` | ~55M (+ rescore pairs) | N/A | ~2.5 GB |
| `student_scores_silver` | ~55M rows (1 per student × opp_key) | N/A | ~1.5 GB |
| `student_component_scores_silver` | ~1.65B rows | N/A | ~8–12 GB |

### 11.3 Postgres Query Performance at Scale

| Query | Students | Expected Postgres Latency |
|-------|---------|--------------------------|
| Roster aggregate (Q1, live) | 30 | < 20 ms (index seek on student_id array) |
| School aggregate (MV lookup) | N/A | < 5 ms (index lookup) |
| School aggregate (raw live) | 6,800 | 200–600 ms (partial index + bitmap scan) |
| District aggregate (MV lookup) | N/A | < 5 ms (index lookup) |
| District aggregate (raw live, Houston ISD) | 189,934 | 15–60 s (too slow; use MV/cache) |
| State aggregate (raw live, TX) | 5.5M | > 60 s (never use raw; nightly pre-computed only) |

### 13.4 ClickHouse Silver + Membership JOIN Query Latency (r6i.2xlarge)

| Query | Scope | Expected ClickHouse Latency |
|-------|-------|--------------------------|
| Silver School (JOIN membership) | 6,800 students | < 250 ms |
| Silver District (JOIN membership) | 50k students | < 1 s |
| Silver District (JOIN membership, Houston ISD) | 200k students | 1–2 s |
| Silver State | 5.5M students | 30–60 s (nightly Lambda only; never real-time) |

### 11.5 SQS Ingestion Throughput

- Up to 5,500 students / minute at TX peak.
- BatchSize=10, MaxBatchingWindow=30s → ~50 concurrent Lambda invocations.
- Each invocation: 1 Postgres batch UPSERT (source of truth) — no direct ClickHouse write.
- Postgres batch UPSERT of 10 rows: ~20–50 ms. Well within Lambda timeout.
- CDC pipeline lag (PeerDB → Transformer → Bronze): ~5–30 s during active ingestion.

---

## 12. Cost Analysis

### 12.1 Base Infrastructure (Texas Scale, Monthly)

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

### 12.2 CDC Layer (Option B — PeerDB, Recommended)

| Service | Config | Monthly Cost |
|---------|--------|-------------|
| PeerDB Engine (Fargate) | 0.25 vCPU / 0.5 GB | ~$9 |
| C# Sign-Pair Transformer (Fargate) | 0.5 vCPU / 1 GB | ~$18 |
| **CDC Subtotal** | | **~$27/month** |

### 12.3 Total Cost by Scale

| Scale | ClickHouse Node | On-Demand / Month | 1-Year Reserved / Month |
|-------|-----------------|------------------|-----------------------|
| **MVP / single client** | `r6i.xlarge` (32 GB) | ~$270–310 | ~$195–235 |
| **Virginia (1.26M students)** | `r6i.xlarge` (32 GB) | ~$310–360 | ~$238–288 |
| **Texas (5.5M students)** | `r6i.2xlarge` (64 GB) | ~$506–594 | ~$284–372 |

> **1-Year Reserved EC2** applies ~42% discount to the ClickHouse node
> (`r6i.2xlarge`: $362/mo on-demand → ~$210/mo reserved), saving ~$152/month.

### 12.4 Cost Savings vs. Original Architecture

| Architecture | Monthly Cost (TX scale) |
|---|---|
| Original (ClickHouse primary + MSK + replica) | ~$1,345–$1,385 |
| **This architecture** (Postgres primary + PeerDB + single ClickHouse) | ~$506–$594 |
| **Savings** | **~$830–$870/month (~$10,000/year)** |

---

## 13. Key Engineering Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **Aurora PostgreSQL as primary store** | Single source of truth; AWS-managed HA/failover; transactional UPSERT handles rescores atomically; enables full query fallback without ClickHouse |
| 2 | **Score Processor writes to Postgres only; ClickHouse receives data via CDC** | Eliminates dual-write race conditions; ClickHouse state is always derived from Postgres; rescore correctness guaranteed by WAL before/after images |
| 3 | **PeerDB (not Debezium + MSK) for CDC** | ~$27/month vs. ~$586/month for MSK; Postgres WAL replication slot provides durable buffering without a separate broker; simpler operational model |
| 4 | **Sign-Pair Transformer (-1/+1) for ClickHouse INSERT correctness** | ClickHouse engines are append-only; a naive UPDATE stream permanently double-counts rescored values in aggregates; the sign-pair pattern solves this atomically |
| 5 | **Medallion Architecture (Bronze → Silver)** | Bronze absorbs raw CDC events; Silver de-duplicates per student via `AggregatingMergeTree`; membership JOIN at query time provides correct, transfer-safe aggregation; each layer has a clear responsibility |
| 6 | **No Gold streaming MVs; Silver + query-time JOIN is the aggregation source** | Gold streaming MVs have three correctness defects: (1) membership JOIN baked at INSERT time — student transfers never reflected; (2) `uniqState` double-counts on rescore; (3) `avgState` denominator inflated on rescore pair. Silver + query-time JOIN is correct for all cases: transfers, multi-enrollment, rescores, and retests. |
| 7 | **Postgres `report_cache` replaces DynamoDB** | Eliminates a separate cache service; co-located with score data; Aurora Serverless v2 scales read capacity automatically; `computed_by` metadata enables observability |
| 8 | **Postgres `MATERIALIZED VIEW CONCURRENTLY` for school/district pre-aggregation (Tier 3 fallback)** | `REFRESH` re-reads all source rows → fully correct for rescores (no double-counting). Non-blocking refresh preserves availability. |
| 9 | **Three-tier API fallback: report_cache → ClickHouse Silver + membership JOIN → Postgres MV / live** | Sub-millisecond cache responses when warm; ClickHouse Silver + JOIN as primary real-time path; Postgres MV as always-available performant fallback for district scope; raw Postgres for school scope |
| 10 | **Single ClickHouse instance (no replica)** | ClickHouse is disposable — it can be fully rebuilt from Postgres via PeerDB Initial Load. A replica is only justified when ClickHouse is the primary store. Saves ~$150–$362/month. |
| 11 | **Membership mirror JOIN FINAL at query time replaces baked MV JOIN** | Baking the membership JOIN in a streaming MV at INSERT time permanently misattributes scores for students who later transfer. Query-time JOIN against `ReplacingMergeTree FINAL` always uses current membership state. `ReplacingMergeTree` mirrors natively support N rows per student for multi-enrollment. |
| 12 | **`REPLICA IDENTITY FULL` on all replicated Aurora tables** | Without it, PeerDB Before images contain only PK columns on UPDATE/DELETE — all non-key fields are null — making the Sign-Pair Transformer's undo row useless for aggregate subtraction |
| 13 | **`date_scored` as `DateTime64(3)` (millisecond precision) everywhere** | `argMaxState` in Silver MVs resolves ties by the version timestamp; second-precision timestamps create 1-second tie windows where two events (e.g., rapid rescore) may be arbitrarily ordered |
| 14 | **Postgres table partitioning by `tenant_id` (LIST) + `school_year` (LIST)** | Partition pruning on year-bound queries; enables efficient per-tenant maintenance; application always queries parent table — never leaf partition names |
| 15 | **`computed_by` metadata in `report_cache`** | Operators can detect if ClickHouse has been unavailable (cache populated from Postgres) and investigate the root cause; enables SLA monitoring per tier |
| 16 | **`use_for_aggregation` flag + `use_for_aggregation_set_at` version key for retest selection** | Score Processor sets flag in a single Postgres transaction (INSERT new opp TRUE + UPDATE prior opps FALSE) ensuring no window where two opps are simultaneously selected. `use_for_aggregation_set_at` is the `argMaxState` version key — not `date_scored` — because `date_scored` does not change when the flag is toggled. |
| 17 | **Circuit breaker on ClickHouse health probe** | A single 200ms per-request probe during sustained ClickHouse outage wastes 200ms on every API call before fallback. In-process circuit breaker caches health state; OPEN state skips Tier 2 entirely for 30s TTL. |
| 18 | **WAL replication slot size monitoring alarm** | If `max_slot_wal_keep_size` is reached, the slot is invalidated and PeerDB requires a full 30–90 minute Initial Load re-snapshot. CloudWatch alarms at 80% and 90% of cap provide early warning. |

---

## 14. Operational Prerequisites

These requirements **must** be enforced before enabling PeerDB replication. Skipping any causes silent data corruption or operational failure.

### 14.1 `REPLICA IDENTITY FULL` on Replicated Tables

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

### 14.2 Millisecond Precision on `date_scored`

```sql
-- Correct — DateTime64(3) everywhere a version key appears
date_scored   TIMESTAMPTZ  -- Postgres (stores microsecond but expose as millisecond to Transformer)
date_scored   DateTime64(3)  -- ClickHouse Bronze, Silver, Gold schemas

-- Wrong — second precision creates 1-second tie windows in argMaxState resolution
date_scored   DateTime     -- ClickHouse DateType without precision
```

Enforce `DateTime64(3)` in the C# Transformer's Bronze insert payload, the Silver schema, and the upstream Postgres column definition.

### 14.3 WAL Cap Configuration

```sql
-- On Aurora Postgres: cap WAL growth during extended PeerDB downtime
-- If cap is hit, the replication slot is invalidated → PeerDB Initial Load required
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
```

### 14.4 Aurora Tables Required by PeerDB Membership Replication

The following 5 normalized RTS relationship tables must exist in Aurora before PeerDB membership
replication jobs are configured. No Aurora views are required for ClickHouse schema deployment.

```sql
-- Required before configuring PeerDB CDC jobs for membership mirrors:
ALTER TABLE trs.school_student    REPLICA IDENTITY FULL;  -- source for membership_school_mirror
ALTER TABLE trs.district_student  REPLICA IDENTITY FULL;  -- source for membership_district_mirror
-- (school_student, district_student, roster_student, district_school, teacher_roster
--  must be created; see §6.3 for full DDL)
```

### 14.5 Deployment Order for ClickHouse Schema

Deploy ClickHouse tables and MVs in this strict order:

1. `trs.student_scores_bronze` (base table — must exist before any MVs that source from it)
2. `trs.student_scores_silver` (target table for Silver MV)
3. `trs.student_scores_silver_mv` (Materialized View — Bronze → Silver)
4. `trs.student_component_scores_bronze` (component Bronze base table)
5. `trs.student_component_scores_silver` (component Silver target table)
6. `trs.student_component_scores_silver_mv` (Materialized View — component Bronze → component Silver)
7. `trs.membership_school_mirror` (membership CDC mirror; required before any Silver queries that JOIN it)
8. `trs.membership_district_mirror` (membership CDC mirror; required before district aggregation queries)
9. `trs.student_attributes` (demographics mirror — standalone)
10. `trs.student_scope_today` (diagnostic only; deploy last, no dependents)

**Removed from deployment order (Gold layer eliminated):** `school_aggregates_gold`, `bronze_to_school_gold_mv`, `district_aggregates_gold`, `bronze_to_district_gold_mv`, `component_aggregates_gold`, `bronze_to_component_gold_mv`.

---

### 14.6 WAL Replication Slot Monitoring

If `max_slot_wal_keep_size` is reached, the replication slot is **invalidated** and PeerDB requires a full Initial Load snapshot to recover (30–90 minutes at TX scale). A lightweight monitoring Lambda runs every 5 minutes via EventBridge Scheduler:

```sql
-- Queries Aurora via read replica connection
SELECT
    slot_name,
    active,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) / 1024 / 1024 AS slot_lag_mb
FROM pg_replication_slots
WHERE slot_name = 'peerdb_trs_slot';
```

Emits custom CloudWatch metric: `TRS/CDC/WALSlotSizeMB`.

| Threshold | Action |
|-----------|--------|
| > 8,192 MB (80% of 10 GB cap) | CloudWatch alarm → SNS → ops (`WARN`: investigate PeerDB lag) |
| > 9,216 MB (90% of cap) | CloudWatch alarm → SNS → ops (`CRITICAL`: imminent slot invalidation) |

---

## 15. MVP Scope & Future Enhancements

### 15.1 MVP Scope

- ✅ Roster-level aggregates (overall, per-student, standard-level)
- ✅ School and district aggregates (overall) — via `report_cache` + Postgres MVs + ClickHouse Silver + membership JOIN
- ✅ Rescore handling (Postgres UPSERT `WHERE date_scored > existing` + Sign-Pair CDC to ClickHouse)
- ✅ Retest handling — `use_for_aggregation` flag selects which opp counts (latest or highest per `retest_selection_rule`)
- ✅ School and district component aggregates (standards, RC) via ClickHouse component Silver + opp_key JOIN
- ✅ RTS membership sync (Aurora primary; ClickHouse mirror best-effort)
- ✅ Test configuration management
- ✅ Full API fallback: ClickHouse unavailable → Postgres serves all queries transparently
- ✅ Postgres `report_cache` and materialized views for school/district scope
- ✅ PeerDB CDC pipeline feeding ClickHouse Bronze Medallion layer
- ✅ Sign-Pair Transformer for correct rescore propagation
- ✅ ClickHouse Medallion Architecture (Bronze / Silver) with query-time membership JOIN
- ✅ Red Hat SSO authentication; multi-tenant isolation via `tenant_id` partition
- ✅ API circuit breaker for ClickHouse health
- ✅ WAL replication slot size monitoring alert

### 15.2 Deferred Enhancements

| # | Enhancement | Notes |
|---|-------------|-------|
| 1 | State-level Postgres MV (`mv_state_overall`) | Nightly refresh only; too expensive more frequently at 5.5M student scale |
| 2 | Demographic slice reports | `student_attributes` JOIN already designed; add `report_cache` rows keyed on demographic dimension |
| 3 | Reporting Category (RC) and Writing Dimension score display views | Stored at ingest (`component_type = 'RC'/'WRITING_DIM'`); display views deferred |
| 6 | ClickHouse rehydration automation | Lambda to automatically detect ClickHouse lag vs. Aurora (compare max `date_scored`) and trigger PeerDB Initial Load |
| 7 | Postgres slow-query monitoring | Alerts on fallback Postgres raw queries; detect if district-scope live queries are being triggered (indicates MV staleness) |
| 8 | Cross-year trend comparisons | Out of scope |
| 9 | Demographics slice aggregates | Add `gender`, `ethnicity`, `ell` as additional `GROUP BY` dimensions in the Silver + membership JOIN aggregation query once base aggregates are validated |

---

## 16. Tech Stack Reference

| Layer | Technology | Notes |
|-------|-----------|-------|
| Language | C# (.NET 8) | Lambda functions, API layer, Fargate transformer |
| Primary DB | Aurora PostgreSQL Serverless v2 | **Source of truth for all data**: scores, membership, config, report cache, idempotency |
| Analytical DB | ClickHouse 24.x — single `r6i.2xlarge` | Self-hosted on EC2; aggregation-only; Bronze/Silver Medallion; rebuilt from Postgres on total loss |
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
| Scheduling | Amazon EventBridge Scheduler | Aggregate Refresh Lambda cron (15 min school / 30 min district / nightly state); WAL Slot Monitoring Lambda every 5 min |
| Observability | AWS CloudWatch | Metrics: `TRS/API/ClickHouseFallback`, `TRS/API/CircuitBreakerOpen`, `TRS/CDC/SchemaError`, `TRS/CDC/WALSlotSizeMB`, `TRS/Scores/StaleResendDiscarded`, `TRS/RTS/ClickHouseWriteFailure`, `TRS/Ingest/ScoreFileRejected`; Alarms on DLQ depth, circuit breaker open >5min, WAL slot at 80% and 90% cap |

---

## Implementation Files

Detailed implementation specifications are maintained separately in the `implementation/` folder:

| File | Component |
|------|-----------|
| [implementation/Score_Processor_Lambda.md](implementation/Score_Processor_Lambda.md) | Score Processor Lambda (Postgres-only write; idempotency; rescore path) |
| [implementation/Score_Reject_Handling.md](implementation/Score_Reject_Handling.md) | Score file rejection: unknown test key, validation failures, data conflicts; S3 rejection bucket; `score_ingest_rejections` DDL; replay runbook |
| [implementation/CDC_SignPair_Transformer.md](implementation/CDC_SignPair_Transformer.md) | C# Sign-Pair Transformer Fargate service (Bronze INSERT logic; error handling) |
| [implementation/RTS_Sync_Lambda.md](implementation/RTS_Sync_Lambda.md) | RTS Membership Sync Lambda (Aurora primary; ClickHouse mirror) |
| [implementation/API_Lambda.md](implementation/API_Lambda.md) | API Lambda (three-tier fallback; scope routing; report_cache; query logic) |

---

*Document maintained by: Principal Software Architect*
*Last updated: 2026-03-17*
*Version: v6*
*Supersedes: TRS_Design_Document_v5.md*
