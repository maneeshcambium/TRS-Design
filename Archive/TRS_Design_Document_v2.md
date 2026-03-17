# Teacher Reporting System (TRS) — Engineering Design Document v2

**Author:** Principal Software Architect  
**Date:** 2026-03-02  
**Status:** Draft — Supersedes v1 for review  
**Audience:** Engineering team; AI implementation agents  

> **Document purpose:** This is the v2 design document reflecting an architectural shift from the original design.
> **Key change from v1:** Aurora PostgreSQL is now the **primary datastore for all data including scores**.
> ClickHouse is retained as an **optional analytical acceleration layer** — the API falls back to
> PostgreSQL if ClickHouse is unavailable. Pre-computed aggregates are written back to Postgres so
> they are available during ClickHouse downtime.

---

## Table of Contents

1. [System Overview](#1-system-overview)
2. [Architecture](#2-architecture)
3. [Infrastructure & Hosting Decisions](#3-infrastructure--hosting-decisions)
4. [Data Models — Inputs & Events](#4-data-models--inputs--events)
5. [Database Schema — Aurora PostgreSQL](#5-database-schema--aurora-postgresql)
6. [Database Schema — ClickHouse (Acceleration Layer)](#6-database-schema--clickhouse-acceleration-layer)
7. [Data Flow](#7-data-flow)
8. [API Fallback Logic](#8-api-fallback-logic)
9. [Query Patterns](#9-query-patterns)
10. [Aggregate Cache & Materialized Views](#10-aggregate-cache--materialized-views)
11. [Scale & Capacity](#11-scale--capacity)
12. [Key Engineering Decisions](#12-key-engineering-decisions)
13. [MVP Scope & Future Enhancements](#13-mvp-scope--future-enhancements)
14. [Tech Stack Reference](#14-tech-stack-reference)

---

## 1. System Overview

### 1.1 Purpose

The **Teacher Reporting System (TRS)** receives student test scores from an upstream Scoring System and presents aggregated analytical reports to teachers and administrators. It is a **read-heavy analytical system** with infrequent writes and heavy aggregations over large student populations.

### 1.2 Architectural Philosophy (v2)

| Concern | v1 Approach | v2 Approach |
|---------|------------|-------------|
| Score storage | ClickHouse primary | **Aurora PostgreSQL primary** |
| Aggregation engine | ClickHouse only | **ClickHouse preferred; Postgres fallback** |
| Aggregate caching | DynamoDB | **Postgres `aggregate_cache` table** |
| Pre-computed aggregates | DynamoDB nightly | **Postgres MVs + ClickHouse-computed cache written to Postgres** |
| ClickHouse down | System partially unavailable | **Transparent failover to Postgres** |

### 1.3 Core Inputs

| Input | Description | Source |
|-------|-------------|--------|
| Student test scores | JSON events per student sitting | Upstream Scoring System |
| Roster/school/district relationships | Hierarchical membership data | External Relationship System (RTS) |
| Test configuration | Test family, subject, grade, standards metadata | Config systems |

### 1.4 Core Outputs

| Report | Aggregation Level | Latency Target |
|--------|-------------------|----------------|
| Class (roster) aggregates | Roster → all students in class | ~1 hour |
| School aggregates | School → all students in school | Up to 1 day |
| District aggregates | District → all students in district | Up to 1 day |
| Per-student score list | Individual student scores within a class | ~1 hour |
| Standard-level class aggregates | Per standard, within a class | ~1 hour |

### 1.5 Users & Concurrent Load

- **Teachers** — class-level (roster) reports.
- **School/District Administrators** — school and district aggregate reports.
- **MVP target concurrency:** 100 concurrent users per client; 1,000 per client per hour.

---

## 2. Architecture

### 2.1 High-Level Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              TRS — Architecture v2                                    │
│                                                                                      │
│  ┌─────────────┐   ┌──────┐   ┌──────────────────┐   ┌────────────────────────────┐ │
│  │  Scoring    │──▶│  S3  │──▶│  SQS Score Queue │──▶│  Lambda: Score Processor   │ │
│  │  System     │   │  Raw │   │  (+ DLQ)         │   │  (dual-write)              │ │
│  └─────────────┘   └──────┘   └──────────────────┘   └────────────┬───────────────┘ │
│                                                                     │                │
│  ┌─────────────┐   ┌──────────────────────┐                        │                │
│  │  RTS        │──▶│  Lambda: RTS         │                        │                │
│  │  (external) │   │  Membership Sync     │                        │                │
│  └─────────────┘   └──────────┬───────────┘                        │                │
│                                │                          ┌─────────▼──────────┐    │
│                                │                          │                    │    │
│                                │          ┌───────────────▶ Aurora PostgreSQL  │    │
│                                └──────────▶ Serverless v2 │ (PRIMARY STORE)    │    │
│                                           │               │ Scores + Membership│    │
│                                           │               │ Config + Auth      │    │
│                                           │               │ Aggregate Cache    │    │
│                                           │               └───────────┬────────┘    │
│                                           │                           │             │
│                                           │         dual-write (best-effort)        │
│                                           │                           │             │
│                                           │               ┌───────────▼────────┐    │
│                                           │               │ ClickHouse Cluster │    │
│                                           └───────────────▶ (EC2) — optional   │    │
│                                                           │ acceleration layer │    │
│                                                           └───────────┬────────┘    │
│                                                                       │             │
│                                               Aggregate Refresh Lambda│             │
│                                          (ClickHouse → Postgres cache)│             │
│                                                                       │             │
│  ┌───────────┐   ┌──────────────────┐   ┌────────────────────────────▼──────────┐  │
│  │CloudFront │   │  API Gateway     │   │   Lambda API                          │  │
│  │+ React    │──▶│  + Lambda Auth   │──▶│   Fallback chain:                     │  │
│  │  (SPA)    │   │  (RH SSO JWT)    │   │   1. Postgres aggregate_cache (fast)  │  │
│  └───────────┘   └──────────────────┘   │   2. ClickHouse live query            │  │
│                                         │   3. Postgres live query (fallback)   │  │
│                                         └───────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 Component Responsibilities

| Component | Technology | Responsibility |
|-----------|-----------|----------------|
| Scoring System | External | Produces score events → S3 |
| S3 Raw | AWS S3 | Landing zone for raw score files; permanent archive |
| SQS Score Queue | AWS SQS (+ DLQ) | Decouples Scoring System from Score Processor; buffers bursts |
| Lambda: Score Processor | AWS Lambda (.NET 8 / C#) | Reads S3 files from SQS batch; validates; **UPSERTs to Postgres (primary)**; dual-writes to ClickHouse (best-effort) |
| RTS | External | Authoritative source of roster/school/district membership |
| Lambda: RTS Membership Sync | AWS Lambda (.NET 8 / C#) | Syncs RTS data into Aurora; also mirrors to ClickHouse membership tables |
| Aurora PostgreSQL Serverless v2 | AWS Aurora | **Primary store for ALL data**: scores, components, membership, config, aggregate cache |
| ClickHouse Cluster | EC2 (r6i series) | **Optional** analytical acceleration; mirrors score data; serves fast aggregation queries |
| Lambda: Aggregate Refresh | AWS Lambda (.NET 8 / C#) | Periodically computes school/district aggregates (in ClickHouse or Postgres) and writes results to `aggregate_cache` in Postgres |
| Lambda API | AWS Lambda (.NET 8 / C#) | REST API handlers; implements fallback chain: aggregate_cache → ClickHouse live → Postgres live |
| Auth | Red Hat SSO (OIDC) | OIDC JWTs with `tenant_id`/`role` claims |
| CloudFront + React SPA | AWS CloudFront | Front-end delivery |

---

## 3. Infrastructure & Hosting Decisions

### 3.1 Why Aurora PostgreSQL is Now the Primary Store

| Reason | Detail |
|--------|--------|
| **Resilience** | Aurora is an AWS-managed service with built-in HA, automated failover, and Multi-AZ replication. ClickHouse on EC2 requires manual HA setup. |
| **Single source of truth** | Having Postgres as the authoritative store eliminates reconciliation problems if ClickHouse diverges or loses data. |
| **Operational simplicity** | No need to maintain separate EC2 node sizing, ClickHouse Keeper quorum, or EC2 replication. |
| **Transactional correctness** | Rescores are handled with a single atomic UPSERT in Postgres (`ON CONFLICT DO UPDATE WHERE date_scored > existing`). No eventual-consistency window. |
| **Fallback capability** | ClickHouse becomes optional; Postgres always holds the complete, authoritative dataset to serve any query. |

### 3.2 Why ClickHouse is Retained

| Reason | Detail |
|--------|--------|
| **Aggregate speed** | For large scopes (50k–300k students), ClickHouse GROUP BY is 10–100× faster than Postgres. School queries: <30ms vs 200–800ms in Postgres. |
| **Columnar scans** | Score data reads only queried columns; Postgres reads full rows. |
| **Optional by design** | System degrades gracefully, not catastrophically, when ClickHouse is unavailable. |

### 3.3 ClickHouse Limitation: No Pre-Aggregated MVs for Rescores

> **Critical:** ClickHouse streaming MVs (triggered by INSERT) are **not safe** for rescored data,
> even if rescores are implemented as a **delete + insert** operation.
>
> **Why delete + insert does not help:**
>
> ClickHouse MVs are wired **only to the INSERT pipeline**. Neither `ALTER TABLE ... DELETE`
> (heavyweight mutation) nor `DELETE FROM` (lightweight delete, ClickHouse 23+) causes the MV to
> subtract the previously accumulated values. The sequence under delete + insert is:
>
> | Step | MV reaction |
> |------|-------------|
> | Original INSERT (score T1) | MV fires → T1 values added to aggregate |
> | DELETE old row (heavyweight or lightweight) | **MV does nothing** — deletes are invisible to MVs |
> | INSERT new row (score T2) | MV fires → T2 values added to aggregate |
> | Net result in MV | **T1 + T2** — old values were never removed. Silent corruption. |
>
> The base table becomes correct after delete + insert, but the MV accumulator is permanently
> corrupted for that key.
>
> **Additional cost of ClickHouse deletes:** `ALTER TABLE ... DELETE` is an asynchronous mutation
> that rewrites data parts — under high rescore volume this creates mutation backlogs.
> Lightweight deletes are faster but still carry an asynchronous visibility window. Neither
> variant fixes the MV problem.
>
> **Safe alternatives:**
> - Live ClickHouse queries with `SELECT ... FINAL` (ReplacingMergeTree deduplicates correctly at read time — no deletes needed)
> - Scheduled Lambda that runs `SELECT ... FINAL GROUP BY` (aggregate over already-deduplicated rows) and writes results to the Postgres `aggregate_cache` table
> - Postgres `MATERIALIZED VIEW` with `REFRESH MATERIALIZED VIEW CONCURRENTLY` — re-reads all source rows on every refresh; rescores are reflected correctly because the refresh is a full recomputation, not an incremental append

### 3.4 Why Postgres Aggregate Cache (not DynamoDB)

- DynamoDB is eliminated in v2. Postgres `aggregate_cache` serves the same purpose.
- Pre-computed aggregates are co-located with the source data in one managed service.
- Aurora Serverless v2 scales read capacity automatically; no separate cache tier needed for this data volume.
- Postgres `MATERIALIZED VIEW` with `REFRESH MATERIALIZED VIEW CONCURRENTLY` provides native, lock-free pre-aggregation directly from score data — correct for rescores (full re-read, no append-only double-counting).

### 3.5 Aurora PostgreSQL Sizing

| Client Scale | Students | Aurora Config | Notes |
|---|---|---|---|
| Small (<1.3M students) | VA, IN | Serverless v2 (min 0.5 ACU, max 16 ACU) | Auto-scales; ~$50–150/mo |
| Medium (~1.85M students) | IL | Serverless v2 (min 2, max 32 ACU) | Read replica for report queries |
| Large (~5.5M students) | TX | Serverless v2 (min 4, max 64 ACU) + 1 read replica | Read replica handles all report queries |

**Storage:** Aurora auto-scales storage. Texas: ~130 GB uncompressed score data/year — Aurora charges ~$0.10/GB/month → ~$13/month for raw scores.

### 3.6 ClickHouse EC2 Sizing (unchanged from v1 — now optional)

| Client Scale | Instance | Nodes | Monthly EC2 Cost |
|---|---|---|---|
| Small | `r6i.2xlarge` (64GB RAM) | 1 (single node acceptable) | ~$362 |
| Medium | `r6i.2xlarge` | 2 (replica set) | ~$724 |
| Large (TX) | `r6i.4xlarge` (128GB RAM) | 2 (replica set) | ~$1,450 |

ClickHouse can be brought down for maintenance without service interruption — the API falls back to Postgres automatically.

### 3.7 Backup & Recovery

**Aurora:** Automated continuous backups to S3 (point-in-time recovery to any second within retention window). Multi-AZ standby for sub-minute failover. No additional backup Lambda needed.

**ClickHouse:** Daily S3 backup via Lambda cron (optional — ClickHouse data is rebuildable from Postgres).

```sql
-- ClickHouse can be fully rebuilt from Postgres if needed:
-- Score Processor re-reads from S3 raw archive → re-inserts to ClickHouse
-- Or: bulk export from Postgres → import to ClickHouse
```

---

## 4. Data Models — Inputs & Events

### 4.1 Test Identification

| Field | Description | Example |
|-------|-------------|---------|
| `TestKey` | Unique identifier for a specific test administration | `checkpoint1-ela-grade1-2025-attempt1` |
| `TestGroupId` | Composite: `{TestFamily}-{Subject}-{Grade}-{Year}-{Attempt}` | `checkpoint1-ela-grade1-2025-attempt1` |
| `TestFamily` | Group of related tests | `CheckPoint1` |
| `Subject` | Subject area | `ELA`, `Math`, `Science` |
| `Grade` | Grade level | `Grade1`, `Grade2` |
| `Attempt` | Attempt/opportunity number | `Attempt1`, `Attempt2` |
| `SchoolYear` | Academic year | `2025`, `2026` |

**TestKey vs TestGroupId:** In 99%+ of cases these are identical. In rare multi-variant cases (online + paper), `TestKey` adds a variant suffix.

### 4.2 Incoming Score Event

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

### 4.3 Rescore vs. Retest

| Event | `OppKey` | `DateScored` | Postgres behavior | ClickHouse behavior |
|-------|----------|--------------|------------------|---------------------|
| Original score | `uuid-A` | T1 | INSERT new row | INSERT new row |
| **Rescore** — same sitting | `uuid-A` | T2 > T1 | ON CONFLICT DO UPDATE (T2 wins) | INSERT; `ReplacingMergeTree` retains T2 |
| **Retest** — student retakes | `uuid-B` (new) | T3 | INSERT new row | INSERT new row |
| Duplicate delivery | `uuid-A` | T1 | ON CONFLICT: WHERE fails (T1 = T1); no-op | FINAL deduplicates |
| Stale resend | `uuid-A` | T0 < T1 | ON CONFLICT: WHERE fails (T0 < T1); no-op | ReplacingMergeTree retains T1 |
| Data conflict (same key+time, different data) | `uuid-A` | T1 | Log alert; no-op | Log alert |

### 4.4 Aggregate Eligibility

Computed at ingest time by Score Processor Lambda. Stored in both Postgres and ClickHouse.

```
is_aggregate_eligible = TRUE  when:
  OppStatus IN ('scored', 'partially_scored')
  AND (ConditionCode IS NULL OR ConditionCode = 'none')

is_aggregate_eligible = FALSE  when:
  OppStatus = 'notscored'
  OR ConditionCode IN ('Invalidated', 'Expired', 'Absent', ...)
```

On rescore, the Lambda recomputes and re-delivers the flag. Both databases are updated atomically (Postgres via UPSERT; ClickHouse via new INSERT with higher `date_scored`).

### 4.5 Relationship Data (from RTS)

```
state → district → school → teacher → roster → student
```

TRS uses only **current** relationships (no effective-dated history). Cached up to one day.

---

## 5. Database Schema — Aurora PostgreSQL

Aurora is the **primary store** for all TRS data: scores, components, membership, config, and aggregate cache.

### 5.1 Score Tables

#### `trs.student_scores`

```sql
CREATE TABLE trs.student_scores
(
  id                     BIGSERIAL,
  tenant_id              VARCHAR(64)  NOT NULL,
  school_year            SMALLINT     NOT NULL,
  test_group_id          VARCHAR(128) NOT NULL,
  student_id             INTEGER      NOT NULL,
  opp_key                UUID         NOT NULL,

  test_key               VARCHAR(128) NOT NULL,
  test_event             VARCHAR(64),
  opp_status             VARCHAR(32)  NOT NULL,
  condition_code         VARCHAR(64),
  tested_date            DATE,
  date_scored            TIMESTAMPTZ  NOT NULL,

  is_aggregate_eligible  BOOLEAN      NOT NULL DEFAULT FALSE,

  overall_scale_score    REAL,
  overall_raw_score      INTEGER,
  overall_lexile_score   INTEGER,
  overall_quantile_score INTEGER,
  overall_perf_level     SMALLINT,
  overall_standard_error REAL,

  ingested_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
  raw_s3_key             VARCHAR(512),

  PRIMARY KEY (tenant_id, id),
  CONSTRAINT uq_student_scores_opp UNIQUE (tenant_id, opp_key)
)
PARTITION BY LIST (tenant_id);

-- Per-tenant partition (example: Texas)
CREATE TABLE trs.student_scores_tx
  PARTITION OF trs.student_scores
  FOR VALUES IN ('tx')
  PARTITION BY RANGE (school_year);

CREATE TABLE trs.student_scores_tx_2026
  PARTITION OF trs.student_scores_tx
  FOR VALUES FROM (2026) TO (2027);
```

**Indexes:**
```sql
-- Primary query pattern: tenant + year + test + student_id list
CREATE INDEX idx_ss_query
  ON trs.student_scores (tenant_id, school_year, test_group_id, student_id)
  WHERE is_aggregate_eligible = TRUE;

-- For per-student score list (includes all eligibility)
CREATE INDEX idx_ss_student
  ON trs.student_scores (tenant_id, school_year, test_group_id, student_id);
```

**Rescore UPSERT pattern:**
```sql
INSERT INTO trs.student_scores
  (tenant_id, school_year, test_group_id, student_id, opp_key,
   test_key, test_event, opp_status, condition_code, tested_date, date_scored,
   is_aggregate_eligible, overall_scale_score, overall_raw_score,
   overall_perf_level, overall_standard_error, raw_s3_key)
VALUES
  ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17)
ON CONFLICT (tenant_id, opp_key) DO UPDATE SET
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
WHERE EXCLUDED.date_scored > trs.student_scores.date_scored;
-- WHERE clause: stale resends are silently ignored (no update occurs)
```

#### `trs.student_component_scores`

```sql
CREATE TABLE trs.student_component_scores
(
  id                    BIGSERIAL,
  tenant_id             VARCHAR(64)  NOT NULL,
  school_year           SMALLINT     NOT NULL,
  test_group_id         VARCHAR(128) NOT NULL,
  student_id            INTEGER      NOT NULL,
  opp_key               UUID         NOT NULL,
  component_type        VARCHAR(32)  NOT NULL,  -- STANDARD | RC | WRITING_DIM
  component_id          VARCHAR(128) NOT NULL,
  perf_level            SMALLINT,
  scale_score           REAL,
  standard_error        REAL,
  condition_code        VARCHAR(64),
  is_aggregate_eligible BOOLEAN      NOT NULL DEFAULT FALSE,
  date_scored           TIMESTAMPTZ  NOT NULL,

  PRIMARY KEY (tenant_id, id),
  CONSTRAINT uq_component_scores UNIQUE (tenant_id, opp_key, component_type, component_id)
)
PARTITION BY LIST (tenant_id);

CREATE INDEX idx_scs_query
  ON trs.student_component_scores
     (tenant_id, school_year, test_group_id, student_id, component_type)
  WHERE is_aggregate_eligible = TRUE;
```

**Rescore UPSERT (component scores):**
```sql
INSERT INTO trs.student_component_scores (...)
VALUES (...)
ON CONFLICT (tenant_id, opp_key, component_type, component_id) DO UPDATE SET
  perf_level            = EXCLUDED.perf_level,
  scale_score           = EXCLUDED.scale_score,
  standard_error        = EXCLUDED.standard_error,
  condition_code        = EXCLUDED.condition_code,
  is_aggregate_eligible = EXCLUDED.is_aggregate_eligible,
  date_scored           = EXCLUDED.date_scored
WHERE EXCLUDED.date_scored > trs.student_component_scores.date_scored;
```

### 5.2 Student Attributes

```sql
CREATE TABLE trs.student_attributes
(
  tenant_id   VARCHAR(64) NOT NULL,
  student_id  INTEGER     NOT NULL,
  gender      VARCHAR(32),
  ethnicity   VARCHAR(64),
  ell         BOOLEAN NOT NULL DEFAULT FALSE,
  lep         BOOLEAN NOT NULL DEFAULT FALSE,
  section_504 BOOLEAN NOT NULL DEFAULT FALSE,
  synced_at   TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  PRIMARY KEY (tenant_id, student_id)
);
```

### 5.3 Membership & Config Tables

| Table | Purpose |
|-------|---------|
| `tenants` | Tenant/state client registry |
| `test_configs` | Test family, subject, grade, standards metadata (JSON column) |
| `standards` | Standard definitions |
| `standard_measures` | Performance level metadata per standard |
| `rosters` | Roster records per tenant |
| `roster_members` | Current student membership per roster (synced from RTS) |
| `schools` | School records per tenant |
| `school_members_today` | Current student→school assignments |
| `districts` | District records per tenant |
| `teachers` | Teacher records per tenant |
| `teacher_rosters` | Teacher→roster assignments |
| `users` | Application users with role + tenant claims |

### 5.4 `trs.aggregate_cache` — Pre-Computed Aggregate Store

Replaces DynamoDB from v1. Holds pre-computed aggregates at school, district, and state scope.
Written by the **Aggregate Refresh Lambda** (computed in ClickHouse or Postgres) and read first by the API layer.

```sql
CREATE TABLE trs.aggregate_cache
(
  tenant_id       VARCHAR(64)  NOT NULL,
  school_year     SMALLINT     NOT NULL,
  test_group_id   VARCHAR(128) NOT NULL,
  scope_type      VARCHAR(16)  NOT NULL,   -- SCHOOL | DISTRICT | STATE
  scope_id        VARCHAR(128) NOT NULL,
  aggregate_type  VARCHAR(32)  NOT NULL DEFAULT 'OVERALL',  -- OVERALL | STANDARD | RC

  students_tested INTEGER,
  pl1             INTEGER,
  pl2             INTEGER,
  pl3             INTEGER,
  pl4             INTEGER,
  pl5             INTEGER,
  avg_scale_score REAL,
  avg_standard_error REAL,
  payload         JSONB,       -- for STANDARD/RC types: full component breakdown

  computed_at     TIMESTAMPTZ  NOT NULL,
  computed_by     VARCHAR(16)  NOT NULL,   -- CLICKHOUSE | POSTGRES
  ttl_expires_at  TIMESTAMPTZ  NOT NULL,

  PRIMARY KEY (tenant_id, school_year, test_group_id, scope_type, scope_id, aggregate_type)
);

CREATE INDEX idx_agg_cache_lookup
  ON trs.aggregate_cache (tenant_id, school_year, test_group_id, scope_type, scope_id)
  INCLUDE (computed_at, ttl_expires_at, students_tested, pl1, pl2, pl3, avg_scale_score);
```

**TTL semantics:** The API Lambda checks `ttl_expires_at` before serving cached data. Stale entries trigger a background refresh (the API still serves stale data while computing, to avoid blocking the user).

### 5.5 Postgres Materialized Views

For school and district aggregates, Postgres native materialized views provide a safe pre-aggregation mechanism — because `REFRESH MATERIALIZED VIEW CONCURRENTLY` re-reads source data entirely, it is **correct for rescores** (no double-counting).

```sql
-- School-level overall aggregate MV
CREATE MATERIALIZED VIEW trs.mv_school_overall AS
SELECT
  ss.tenant_id,
  ss.school_year,
  ss.test_group_id,
  smt.school_id,
  COUNT(*)                                    AS students_tested,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)  AS pl1,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)  AS pl2,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)  AS pl3,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)  AS pl4,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)  AS pl5,
  AVG(ss.overall_scale_score)                AS avg_scale_score,
  AVG(ss.overall_standard_error)             AS avg_standard_error,
  NOW()                                      AS refreshed_at
FROM trs.student_scores ss
JOIN trs.school_members_today smt
  ON ss.tenant_id = smt.tenant_id AND ss.student_id = smt.student_id
WHERE ss.is_aggregate_eligible = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.test_group_id, smt.school_id;

-- Unique index required for CONCURRENTLY refresh
CREATE UNIQUE INDEX uq_mv_school_overall
  ON trs.mv_school_overall (tenant_id, school_year, test_group_id, school_id);

-- District-level overall aggregate MV
CREATE MATERIALIZED VIEW trs.mv_district_overall AS
SELECT
  ss.tenant_id,
  ss.school_year,
  ss.test_group_id,
  d.district_id,
  COUNT(*)                                    AS students_tested,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)  AS pl1,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)  AS pl2,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)  AS pl3,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)  AS pl4,
  COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)  AS pl5,
  AVG(ss.overall_scale_score)                AS avg_scale_score,
  NOW()                                      AS refreshed_at
FROM trs.student_scores ss
JOIN trs.school_members_today smt
  ON ss.tenant_id = smt.tenant_id AND ss.student_id = smt.student_id
JOIN trs.schools sc
  ON smt.tenant_id = sc.tenant_id AND smt.school_id = sc.school_id
JOIN trs.districts d
  ON sc.tenant_id = d.tenant_id AND sc.district_id = d.district_id
WHERE ss.is_aggregate_eligible = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.test_group_id, d.district_id;

CREATE UNIQUE INDEX uq_mv_district_overall
  ON trs.mv_district_overall (tenant_id, school_year, test_group_id, district_id);
```

**Refresh schedule (Lambda cron):**
```sql
-- Non-blocking; uses snapshot of source data; existing rows readable during refresh
REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_school_overall;
REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_district_overall;
```

| MV | Refresh frequency | Condition |
|----|------------------|-----------|
| `mv_school_overall` | Every 15 minutes during test window; nightly off-window | Triggered by Score Processor SQS depth or cron |
| `mv_district_overall` | Every 30 minutes during test window; nightly off-window | Same |
| `mv_state_overall` (future) | Nightly only | Too expensive to refresh frequently |

---

## 6. Database Schema — ClickHouse (Acceleration Layer)

ClickHouse mirrors score data from Postgres and serves fast aggregation queries. It is **not** the source of truth. If ClickHouse is unavailable, Postgres serves all queries.

### 6.1 `trs.student_scores` — ReplacingMergeTree (mirror)

```sql
CREATE TABLE trs.student_scores
(
  tenant_id        String,
  school_year      UInt16,
  test_group_id    String,
  student_id       Int32,
  opp_key          String,
  test_key         String,
  test_event       LowCardinality(String),
  opp_status       LowCardinality(String),
  condition_code   Nullable(LowCardinality(String)),
  tested_date      Date,
  date_scored      DateTime64(3),           -- version key: higher wins per opp_key
  is_aggregate_eligible UInt8 DEFAULT 0,
  overall_scale_score    Nullable(Float32),
  overall_raw_score      Nullable(Int32),
  overall_lexile_score   Nullable(Int32),
  overall_quantile_score Nullable(Int32),
  overall_perf_level     Nullable(UInt8),
  overall_standard_error Nullable(Float32),
  ingested_at    DateTime64(3) DEFAULT now(),
  raw_s3_key     String
)
ENGINE = ReplacingMergeTree(date_scored)
PARTITION BY (tenant_id, school_year)
ORDER BY (tenant_id, school_year, test_group_id, student_id, opp_key);
```

### 6.2 `trs.student_component_scores` — ReplacingMergeTree (mirror)

```sql
CREATE TABLE trs.student_component_scores
(
  tenant_id      String,
  school_year    UInt16,
  test_group_id  String,
  student_id     Int32,
  opp_key        String,
  component_type LowCardinality(String),
  component_id   String,
  perf_level     Nullable(UInt8),
  scale_score    Nullable(Float32),
  standard_error Nullable(Float32),
  condition_code Nullable(LowCardinality(String)),
  is_aggregate_eligible UInt8 DEFAULT 0,
  date_scored    DateTime64(3)
)
ENGINE = ReplacingMergeTree(date_scored)
PARTITION BY (tenant_id, school_year)
ORDER BY (tenant_id, school_year, test_group_id, student_id, opp_key, component_type, component_id);
```

### 6.3 `trs.student_scope_today` — Membership Mirror

```sql
CREATE TABLE trs.student_scope_today
(
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

### 6.4 `trs.student_attributes` — Demographics Mirror

```sql
CREATE TABLE trs.student_attributes
(
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

> **No pre-aggregated ClickHouse streaming MVs for school/district counts.** ClickHouse streaming MVs fire on incremental INSERT batches and silently double-count under rescores — even when chained on top of an `AggregatingMergeTree` like `student_scores_latest` (see §6.6). All ClickHouse live queries use `SELECT ... FINAL` on base tables OR `argMaxMerge()` on `student_scores_latest` (§6.5). Pre-aggregated results for school/district scope are written to the Postgres `aggregate_cache` by the scheduled Aggregate Refresh Lambda.

### 6.5 `trs.student_scores_latest` — AggregatingMergeTree (optional per-student optimization)

This table provides a faster way to resolve the current version of each score without `FINAL` on the base table. The Aggregate Refresh Lambda uses it as its source for school/district aggregate computations.

```sql
CREATE TABLE trs.student_scores_latest
(
  tenant_id               String,
  school_year             UInt16,
  test_group_id           String,
  student_id              Int32,
  opp_key                 String,
  overall_scale_score_s   AggregateFunction(argMax, Nullable(Float32),  DateTime64(3)),
  overall_perf_level_s    AggregateFunction(argMax, Nullable(UInt8),    DateTime64(3)),
  overall_raw_score_s     AggregateFunction(argMax, Nullable(Int32),    DateTime64(3)),
  opp_status_s            AggregateFunction(argMax, String,             DateTime64(3)),
  is_aggregate_eligible_s AggregateFunction(argMax, UInt8,              DateTime64(3)),
  gender_s                AggregateFunction(argMax, Nullable(String),   DateTime64(3)),
  ethnicity_s             AggregateFunction(argMax, Nullable(String),   DateTime64(3)),
  ell_s                   AggregateFunction(argMax, UInt8,              DateTime64(3)),
  lep_s                   AggregateFunction(argMax, UInt8,              DateTime64(3)),
  section_504_s           AggregateFunction(argMax, UInt8,              DateTime64(3))
)
ENGINE = AggregatingMergeTree()
PARTITION BY (tenant_id, school_year)
ORDER BY (tenant_id, school_year, test_group_id, student_id, opp_key);

CREATE MATERIALIZED VIEW trs.student_scores_latest_mv
TO trs.student_scores_latest
AS SELECT
  tenant_id, school_year, test_group_id, student_id, opp_key,
  argMaxState(overall_scale_score,    date_scored) AS overall_scale_score_s,
  argMaxState(overall_perf_level,     date_scored) AS overall_perf_level_s,
  argMaxState(overall_raw_score,      date_scored) AS overall_raw_score_s,
  argMaxState(opp_status,             date_scored) AS opp_status_s,
  argMaxState(is_aggregate_eligible,  date_scored) AS is_aggregate_eligible_s,
  argMaxState(gender,                 date_scored) AS gender_s,
  argMaxState(ethnicity,              date_scored) AS ethnicity_s,
  argMaxState(ell,                    date_scored) AS ell_s,
  argMaxState(lep,                    date_scored) AS lep_s,
  argMaxState(section_504,            date_scored) AS section_504_s
FROM trs.student_scores
GROUP BY tenant_id, school_year, test_group_id, student_id, opp_key;
```

**What this correctly solves:** Per-student point reads without `FINAL`. `argMaxMerge()` merges partial states and returns the field value from the row with the highest `date_scored` for each `opp_key`. Rescores are handled correctly — the partial state with the higher `date_scored` wins during merge.

```sql
-- Per-student roster query using student_scores_latest (no FINAL needed)
SELECT
  student_id,
  argMaxMerge(overall_perf_level_s)  AS overall_perf_level,
  argMaxMerge(overall_scale_score_s) AS overall_scale_score,
  argMaxMerge(opp_status_s)          AS opp_status
FROM trs.student_scores_latest
WHERE tenant_id = ? AND school_year = ? AND test_group_id = ?
  AND student_id IN (?)
GROUP BY student_id
```

**The Aggregate Refresh Lambda uses this table as its source** for school/district queries — faster and cleaner than `SELECT ... FINAL` on the base table because ClickHouse resolves the latest version via aggregate state merging rather than part-level deduplication at query time:

```sql
-- School aggregate query via student_scores_latest (used by Aggregate Refresh Lambda)
SELECT
  sst.school_id,
  countIf(argMaxMerge(ssl.is_aggregate_eligible_s) = 1) AS students_tested,
  countIf(argMaxMerge(ssl.is_aggregate_eligible_s) = 1
    AND argMaxMerge(ssl.overall_perf_level_s) = 1)      AS pl1,
  countIf(argMaxMerge(ssl.is_aggregate_eligible_s) = 1
    AND argMaxMerge(ssl.overall_perf_level_s) = 2)      AS pl2,
  countIf(argMaxMerge(ssl.is_aggregate_eligible_s) = 1
    AND argMaxMerge(ssl.overall_perf_level_s) = 3)      AS pl3,
  avgIf(argMaxMerge(ssl.overall_scale_score_s),
    argMaxMerge(ssl.is_aggregate_eligible_s) = 1)       AS avg_scale_score
FROM trs.student_scores_latest ssl
JOIN trs.student_scope_today sst FINAL USING (tenant_id, student_id)
WHERE ssl.tenant_id     = ?
  AND ssl.school_year   = ?
  AND ssl.test_group_id = ?
  AND sst.school_id     = ?
GROUP BY ssl.tenant_id, ssl.school_year, ssl.test_group_id, sst.school_id
```

**Comparison: `student_scores_latest` vs base table with `FINAL`**

| Aspect | `SELECT ... FINAL` on base table | `student_scores_latest` + `argMaxMerge` |
|--------|----------------------------------|----------------------------------------|
| Deduplication mechanism | Part-level scan + sort-merge at query time | Pre-merged partial aggregate states |
| Rescore correctness | ✅ Correct | ✅ Correct |
| Per-student point read speed | Good | Better — no FINAL overhead |
| School/district scheduled aggregate | ✅ Correct | ✅ Correct (faster) |
| School/district streaming MV | ❌ Double-counts rescores | ❌ Still double-counts (see §6.6) |

### 6.6 Why a Second-Level Aggregate MV on `student_scores_latest` Still Fails

> **A ClickHouse streaming MV chained on top of `student_scores_latest` for school/district
> aggregates does not solve the rescore problem — it re-introduces the same double-counting at the
> school level.**
>
> **Root cause:** A streaming MV fires on the **incremental INSERT batch** arriving at its source
> table — not on the full merged state. `argMaxMerge()` correctly resolves the latest score when
> the table is *queried* in full. But a chained MV never queries the table — it only observes the
> stream of raw partial states being inserted, one batch at a time.
>
> The sequence for a rescore:
>
> | Step | What the chained school MV sees |
> |------|---------------------------------|
> | Original score INSERT → base table | `student_scores_latest_mv` fires → T1 `argMaxState` batch inserted → chained school MV fires on T1 batch → T1 contribution added to school aggregate |
> | Rescore INSERT → base table | `student_scores_latest_mv` fires → T2 `argMaxState` batch inserted → chained school MV fires on T2 batch → T2 contribution added to school aggregate |
> | Net result in school aggregate MV | **T1 + T2 accumulated** — `argMaxMerge` would return T2 if you queried the table directly, but the chained MV already accumulated both batches independently and cannot subtract T1 |
>
> The `argMaxMerge` deduplication is present only in the *finalized query view* of
> `student_scores_latest`. The incremental INSERT stream that a chained MV observes still
> contains both T1 and T2 partial states as separate events.
>
> **The correct pattern is always: scheduled full query → write to `aggregate_cache`.**
> Never streaming MV for counts/sums at any scope above per-student level.

---

## 7. Data Flow

### 7.1 Score Ingestion Path

```
Scoring System
    │
    ▼ PUT JSON file
  S3 Raw (s3://trs-raw/{tenant_id}/scores/...)
    │
    ▼ S3 Event Notification
  SQS Score Queue  ──[DLQ]──▶  Dead Letter Queue
    │
    ▼ Batch trigger (BatchSize=100, MaxBatchingWindow=30s)
  Lambda: Score Processor
    │
    ├─▶ Download up to 100 S3 files
    ├─▶ Parse each score event
    ├─▶ Compute is_aggregate_eligible flag
    ├─▶ Expand StandardScores + ReportingCategoryScores → component rows
    │
    ├─▶ [PRIMARY] Batch UPSERT → Aurora PostgreSQL
    │     INSERT INTO trs.student_scores ... ON CONFLICT DO UPDATE WHERE date_scored >
    │     INSERT INTO trs.student_component_scores ... ON CONFLICT DO UPDATE WHERE date_scored >
    │
    └─▶ [SECONDARY — best-effort] Batch INSERT → ClickHouse
          INSERT INTO trs.student_scores (one INSERT per Lambda invocation)
          INSERT INTO trs.student_component_scores (one INSERT per Lambda invocation)
          [If ClickHouse INSERT fails: log warning, do NOT fail the Lambda invocation;
           Postgres is the source of truth; ClickHouse lag is acceptable]
```

**Dual-write ordering rule:**
1. Postgres UPSERT must succeed before returning success to SQS.
2. ClickHouse INSERT is fire-and-forget — Lambda proceeds regardless of ClickHouse outcome.
3. If ClickHouse is consistently lagging, the Aggregate Refresh Lambda detects stale aggregates and refreshes from Postgres.

**ClickHouse rebuild (if divergence detected):**
- Re-read raw S3 archive, re-process through Score Processor logic, re-insert to ClickHouse.
- OR: bulk export from Postgres → ClickHouse via `INSERT INTO trs.student_scores SELECT ...` from a Postgres foreign data wrapper or staging Lambda.

### 7.2 Rescore Handling

```
Score Processor receives rescore event (same OppKey, higher DateScored):

   Postgres:
     ON CONFLICT (tenant_id, opp_key) DO UPDATE ...
     WHERE EXCLUDED.date_scored > student_scores.date_scored
     → Atomic. Correct immediately.

   ClickHouse:
     INSERT new row (same opp_key, higher date_scored)
     → ReplacingMergeTree retains higher date_scored at next merge
     → FINAL deduplicates at query time before merge settles

   Aggregate Cache (Postgres):
     → TTL on aggregate_cache row will expire (within 5–30 min depending on scope)
     → Next Aggregate Refresh Lambda run recomputes and overwrites cache
     → OR: Score Processor publishes rescore event to SNS → Lambda invalidates specific cache rows
```

### 7.3 Membership Sync Path (RTS → TRS)

```
RTS external event or nightly full sync
    │
    ▼
  Lambda: RTS Membership Sync
    │
    ├─▶ Aurora: UPSERT roster_members, school_members_today, districts, schools, student_attributes
    └─▶ ClickHouse: INSERT INTO trs.student_scope_today (best-effort mirror)
                    INSERT INTO trs.student_attributes  (best-effort mirror)
```

### 7.4 Aggregate Refresh Path

```
EventBridge Cron: every 15 min (school), 30 min (district), nightly (state)
  OR
SNS rescore notification → targeted cache invalidation
    │
    ▼
  Lambda: Aggregate Refresh
    │
    ├─▶ Attempt ClickHouse aggregate query (SELECT ... FINAL GROUP BY ... scope)
    │     If success → upsert results into trs.aggregate_cache (computed_by='CLICKHOUSE')
    │
    └─▶ If ClickHouse unavailable:
          Run equivalent Postgres query (slower, but correct)
          → upsert results into trs.aggregate_cache (computed_by='POSTGRES')
    │
    ▼
  ALSO: REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_school_overall
        REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_district_overall
        (runs in Postgres regardless of ClickHouse availability)
```

---

## 8. API Fallback Logic

The API Lambda implements a **three-tier fallback chain** for every aggregate request. The chain is transparent to the caller.

### 8.1 Fallback Chain

```
API Request: GET /reports/school/{schoolId}/overall?testGroupId=...&schoolYear=...
                │
                ▼
   ┌────────────────────────────────────────────┐
   │ Tier 1: Postgres aggregate_cache           │
   │  SELECT * FROM trs.aggregate_cache         │
   │  WHERE scope_type='SCHOOL' AND scope_id=?  │
   │  AND ttl_expires_at > NOW()                │
   └────────────────┬───────────────────────────┘
                    │ HIT: serve immediately
                    │ MISS (stale or absent): proceed to Tier 2
                    ▼
   ┌────────────────────────────────────────────┐
   │ Tier 2: ClickHouse live query              │
   │  (health-checked before attempt)           │
   │  SELECT ... FINAL FROM trs.student_scores  │
   │  JOIN trs.student_scope_today FINAL ...    │
   └────────────────┬───────────────────────────┘
                    │ Success: serve + async upsert to aggregate_cache (TTL=5min)
                    │ Failure (timeout/unavailable): proceed to Tier 3
                    ▼
   ┌────────────────────────────────────────────┐
   │ Tier 3: Postgres live query                │
   │  (always available; slower for large scope)│
   │  SELECT ... FROM trs.mv_school_overall     │
   │  WHERE school_id = ?                       │
   │  (MV query; or raw query if MV stale)      │
   └────────────────┬───────────────────────────┘
                    │ Serve + async upsert to aggregate_cache (TTL=5min, computed_by='POSTGRES')
                    ▼
              Response returned
```

### 8.2 Fallback Chain — C# Implementation Sketch

```csharp
public async Task<AggregateResult> GetSchoolAggregateAsync(
    string tenantId, string schoolId, string testGroupId, int schoolYear)
{
    // Tier 1: Postgres aggregate_cache (fast — indexed lookup)
    var cached = await _pgRepo.GetAggregateCache(
        tenantId, schoolYear, testGroupId, "SCHOOL", schoolId);
    if (cached != null && cached.TtlExpiresAt > DateTime.UtcNow)
        return cached;

    // Tier 2: ClickHouse live query
    if (await _clickHouseHealth.IsAvailableAsync())
    {
        try
        {
            var chResult = await _chRepo.GetSchoolAggregateAsync(
                tenantId, schoolId, testGroupId, schoolYear);
            // Async background cache update — do not block the response
            _ = _pgRepo.UpsertAggregateCacheAsync(
                tenantId, schoolYear, testGroupId, "SCHOOL", schoolId,
                chResult, source: "CLICKHOUSE", ttlMinutes: 5);
            return chResult;
        }
        catch (Exception ex)
        {
            _logger.LogWarning(ex, "ClickHouse unavailable; falling back to Postgres");
        }
    }

    // Tier 3: Postgres live query (MV preferred; raw query if MV too stale)
    var pgResult = await _pgRepo.GetSchoolAggregateFromMvAsync(
        tenantId, schoolId, testGroupId, schoolYear);
    _ = _pgRepo.UpsertAggregateCacheAsync(
        tenantId, schoolYear, testGroupId, "SCHOOL", schoolId,
        pgResult, source: "POSTGRES", ttlMinutes: 5);
    return pgResult;
}
```

### 8.3 Roster Queries (Small Scope — No Cache)

Roster queries scan ~30 students and are fast in both ClickHouse and Postgres. No aggregate cache needed.

```csharp
public async Task<AggregateResult> GetRosterAggregateAsync(
    string tenantId, string rosterId, string testGroupId, int schoolYear)
{
    var studentIds = await _pgRepo.GetRosterMembersAsync(tenantId, rosterId);

    // Try ClickHouse first
    if (await _clickHouseHealth.IsAvailableAsync())
    {
        try { return await _chRepo.GetRosterAggregateAsync(tenantId, studentIds, testGroupId, schoolYear); }
        catch (Exception ex) { _logger.LogWarning(ex, "ClickHouse fallback to Postgres"); }
    }

    // Fallback: Postgres (sub-100ms for 30 students with index)
    return await _pgRepo.GetRosterAggregateAsync(tenantId, studentIds, testGroupId, schoolYear);
}
```

### 8.4 ClickHouse Health Check

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

---

## 9. Query Patterns

### 9.1 Roster Queries

**Q1 — Overall aggregate (ClickHouse path)**
```sql
SELECT
  count()                          AS students_tested,
  countIf(overall_perf_level = 1)  AS pl1,
  countIf(overall_perf_level = 2)  AS pl2,
  countIf(overall_perf_level = 3)  AS pl3,
  countIf(overall_perf_level = 4)  AS pl4,
  countIf(overall_perf_level = 5)  AS pl5,
  avg(overall_scale_score)         AS avg_scale_score,
  avg(overall_standard_error)      AS avg_standard_error
FROM trs.student_scores FINAL
WHERE tenant_id     = ?
  AND school_year   = ?
  AND test_group_id = ?
  AND student_id    IN (?)          -- ~30 IDs from Aurora roster_members
  AND is_aggregate_eligible = 1
```

**Q1 — Overall aggregate (Postgres fallback path)**
```sql
SELECT
  COUNT(*)                                              AS students_tested,
  COUNT(*) FILTER (WHERE overall_perf_level = 1)       AS pl1,
  COUNT(*) FILTER (WHERE overall_perf_level = 2)       AS pl2,
  COUNT(*) FILTER (WHERE overall_perf_level = 3)       AS pl3,
  COUNT(*) FILTER (WHERE overall_perf_level = 4)       AS pl4,
  COUNT(*) FILTER (WHERE overall_perf_level = 5)       AS pl5,
  AVG(overall_scale_score)                             AS avg_scale_score,
  AVG(overall_standard_error)                          AS avg_standard_error
FROM trs.student_scores
WHERE tenant_id            = $1
  AND school_year          = $2
  AND test_group_id        = $3
  AND student_id           = ANY($4)    -- array of ~30 IDs
  AND is_aggregate_eligible = TRUE
```

**Q2 — Per-student score list (Postgres fallback)**
```sql
SELECT
  student_id,
  overall_scale_score,
  overall_perf_level,
  overall_raw_score,
  overall_standard_error,
  opp_status,
  condition_code,
  tested_date
FROM trs.student_scores
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND student_id    = ANY($4)
ORDER BY student_id
```

**Q3 — Standard-level class aggregate (Postgres fallback)**
```sql
SELECT
  component_id                                             AS standard_id,
  COUNT(*)                                                 AS students_tested,
  COUNT(*) FILTER (WHERE perf_level = 1)                  AS pl1,
  COUNT(*) FILTER (WHERE perf_level = 2)                  AS pl2,
  COUNT(*) FILTER (WHERE perf_level = 3)                  AS pl3,
  AVG(scale_score)                                        AS avg_scale_score
FROM trs.student_component_scores
WHERE tenant_id            = $1
  AND school_year          = $2
  AND test_group_id        = $3
  AND student_id           = ANY($4)
  AND component_type       = 'STANDARD'
  AND is_aggregate_eligible = TRUE
GROUP BY component_id
ORDER BY component_id
```

### 9.2 School & District Queries

For school and district scope, the API first checks `aggregate_cache`, then falls back to ClickHouse or Postgres live as described in §8.

**School query — ClickHouse path (live Tier 2)**
```sql
SELECT
  count()                               AS students_tested,
  countIf(ss.overall_perf_level = 1)   AS pl1,
  countIf(ss.overall_perf_level = 2)   AS pl2,
  countIf(ss.overall_perf_level = 3)   AS pl3,
  avg(ss.overall_scale_score)          AS avg_scale_score
FROM trs.student_scores ss FINAL
JOIN trs.student_scope_today sst FINAL USING (tenant_id, student_id)
WHERE ss.tenant_id     = ?
  AND ss.school_year   = ?
  AND ss.test_group_id = ?
  AND sst.school_id    = ?
  AND ss.is_aggregate_eligible = 1
```

**School query — Postgres path (Tier 3 — uses MV)**
```sql
SELECT students_tested, pl1, pl2, pl3, pl4, pl5, avg_scale_score
FROM trs.mv_school_overall
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND school_id     = $4
```

**School query — Postgres path (Tier 3 — raw if MV too stale)**
```sql
SELECT
  COUNT(*) FILTER (WHERE ss.is_aggregate_eligible)                        AS students_tested,
  COUNT(*) FILTER (WHERE ss.is_aggregate_eligible AND ss.overall_perf_level = 1) AS pl1,
  COUNT(*) FILTER (WHERE ss.is_aggregate_eligible AND ss.overall_perf_level = 2) AS pl2,
  COUNT(*) FILTER (WHERE ss.is_aggregate_eligible AND ss.overall_perf_level = 3) AS pl3,
  AVG(ss.overall_scale_score) FILTER (WHERE ss.is_aggregate_eligible)    AS avg_scale_score
FROM trs.student_scores ss
JOIN trs.school_members_today smt
  ON ss.tenant_id = smt.tenant_id AND ss.student_id = smt.student_id
WHERE ss.tenant_id     = $1
  AND ss.school_year   = $2
  AND ss.test_group_id = $3
  AND smt.school_id    = $4
```

---

## 10. Aggregate Cache & Materialized Views

### 10.1 Serving Strategy by Scope

| Scope | Rows scanned | ClickHouse latency | Postgres MV latency | Postgres raw latency | Cache TTL |
|-------|-------------|-------------------|---------------------|---------------------|-----------|
| Roster (~30 students) | ~30 | <5ms | N/A — no MV | <10ms (index seek) | No cache |
| School (~6,800 students) | ~6,800 | <30ms | <5ms (MV lookup) | 200–800ms | 15 min |
| District ≤300k students | ≤300k | 50–200ms | <5ms (MV lookup) | 3–15s | 5 min |
| District 300k–1M | ~500k | 150–500ms | <5ms (MV lookup) | 15–60s | 5 min |
| State | 1M–5.5M | 400ms–3s | <5ms (MV lookup) | too slow — cache only | Nightly |

> **Rule for Postgres live query at district/state scope:** A direct live query against `student_scores` for large districts or states will be too slow for an interactive API response. In fallback mode, always prefer the Postgres MV (`mv_district_overall`) or the `aggregate_cache` table. Only fall back to a live Postgres raw query for school scope and below.

### 10.2 Cache Invalidation

| Trigger | Action |
|---------|--------|
| TTL expires (`ttl_expires_at < NOW()`) | API serves stale + triggers background Aggregate Refresh Lambda |
| Rescore event arrives at Score Processor | Publish to SNS `trs-rescore-events`; Aggregate Refresh Lambda subscribes and invalidates affected `aggregate_cache` rows |
| RTS Membership Sync completes | Aggregate Refresh Lambda re-runs MVs (`REFRESH MATERIALIZED VIEW CONCURRENTLY`) |
| Manual admin action | `DELETE FROM trs.aggregate_cache WHERE scope_type = ? AND scope_id = ?` |

### 10.3 Cache Population Sources

```
Priority 1: ClickHouse live query + FINAL → aggregate_cache (computed_by='CLICKHOUSE')
Priority 2: Postgres live query / MV → aggregate_cache (computed_by='POSTGRES')
```

The `computed_by` column is stored for observability — callers always receive the aggregate value regardless of how it was computed.

---

## 11. Scale & Capacity

### 11.1 State-Level Student Volumes

| State | Total Students | Largest District | Largest School |
|-------|----------------|-----------------|----------------|
| Texas | 5,543,751 | 189,934 (Houston ISD) | 6,798 (Allen HS) |
| Virginia | 1,259,958 | 179,858 (Fairfax County) | 5,100 (Alexandria City) |
| Illinois | 1,850,074 | 321,666 (Chicago PS) | 4,300 (Lane Tech) |
| Indiana | 1,009,888 | 31,000 (Indianapolis PS) | 5,400 (Carmel HS) |

### 11.2 Annual Row Counts

| Grain | Texas rows/year | Postgres storage | ClickHouse storage |
|-------|----------------|-----------------|-------------------|
| `student_scores` | ~55M | ~22 GB (uncompressed) | ~2 GB (compressed) |
| `student_component_scores` | ~1.65B | ~660 GB (uncompressed) | ~9–13 GB (compressed) |

> **Postgres storage note:** At ~660 GB for Texas component scores per year, an Aurora Serverless v2 instance at the large tier is adequate. Aurora auto-scales storage at ~$0.10/GB/month → ~$66/month for component scores. Consider table partitioning by `school_year` to control per-partition size and enable partition pruning on year-bound queries.

### 11.3 Postgres Query Performance at Scale

With proper indexes and table partitioning:

| Query | Students | Expected Postgres latency |
|-------|---------|--------------------------|
| Roster aggregate (Q1) | 30 | <20ms (index seek on student_id array) |
| School aggregate (raw) | 6,800 | 200–600ms (partial index + bitmap scan) |
| School aggregate (MV) | N/A | <5ms (index lookup) |
| District aggregate (MV) | N/A | <5ms (index lookup) |
| District aggregate (raw, Houston ISD) | 189,934 | 15–60s (too slow for live; use MV/cache) |

The **aggregate_cache and Postgres MV** design is what makes Postgres a viable fallback at district/state scope — the raw live Postgres path is reserved for school scale and below.

### 11.4 SQS Ingestion Throughput

- Up to 5,500 students/minute at TX peak.
- BatchSize=100, MaxBatchingWindow=30s → ~50 concurrent Lambda invocations.
- Each invocation: 1 Postgres batch UPSERT + 1 ClickHouse INSERT.
- Postgres batch UPSERT of 100 rows: ~50–100ms. Well within Lambda timeout.

---

## 12. Key Engineering Decisions

| # | Decision | Rationale |
|---|----------|-----------|
| 1 | **Aurora PostgreSQL as primary store for all data including scores** | Single source of truth; AWS-managed HA/failover; transactional UPSERT handles rescores atomically; enables full fallback without ClickHouse |
| 2 | **ClickHouse retained as optional acceleration layer** | 10–100× faster aggregations at large scope; acceptable to lose access temporarily |
| 3 | **Dual-write: Postgres first (must succeed), ClickHouse second (best-effort)** | Postgres UPSERT success = data durability guarantee; ClickHouse write failure does not fail the ingestion pipeline |
| 4 | **Postgres `aggregate_cache` table replaces DynamoDB** | Eliminates a separate cache service; co-located with score data; serves pre-computed aggregates during ClickHouse downtime |
| 5 | **Postgres `MATERIALIZED VIEW CONCURRENTLY` for school/district pre-aggregation** | REFRESH re-reads source data → fully correct for rescores (no double-counting). Non-blocking refresh preserves availability. |
| 6 | **No ClickHouse streaming MVs for aggregate pre-computation** | ClickHouse MVs fire only on INSERT — not on DELETE. Even a delete + insert rescore strategy does not fix this: the DELETE is invisible to the MV, so old values remain in the accumulator and new values are added → permanent silent double-counting. Use `SELECT ... FINAL` for live queries and Postgres MVs for pre-aggregation. |
| 7 | **Three-tier API fallback: aggregate_cache → ClickHouse live → Postgres live/MV** | Provides sub-millisecond cached responses when warm; fast ClickHouse live queries as primary real-time path; Postgres as always-available fallback |
| 8 | **Postgres UPSERT with `WHERE EXCLUDED.date_scored > existing`** | Rescores are handled atomically and correctly; stale resends are silently ignored without extra reads or conflict checks |
| 9 | **Demographics in `student_attributes` (not in score rows)** | Score rows remain stable after ingest; demographic syncs do not touch score data; JOINs at query time are acceptable at all scopes that have cached aggregates |
| 10 | **`school_id`/`district_id` NOT in score rows** | Mid-year transfers: if denormalized, all score rows must be re-written with no safe version key; `school_members_today` / `student_scope_today` handle transfers cleanly |
| 11 | **`is_aggregate_eligible` computed at ingest, stored in both databases** | Rescores re-deliver the flag; no query-time eligibility logic; correct in both Postgres (UPSERT) and ClickHouse (ReplacingMergeTree) |
| 12 | **Postgres table partitioning by `tenant_id` (LIST) + `school_year` (RANGE)** | Partition pruning on year-bound queries; enables efficient per-tenant maintenance (detach/archive old year partition) |
| 13 | **Aggregate Refresh Lambda writes `computed_by` metadata to aggregate_cache** | Observability: operators can detect if ClickHouse has been unavailable (cache populated from Postgres) and investigate root cause |

---

## 13. MVP Scope & Future Enhancements

### 13.1 MVP Scope

- ✅ Roster-level aggregates (overall, per-student, standard-level, standard perf-level list)
- ✅ School and district aggregates (overall) — via aggregate_cache + Postgres MVs + ClickHouse live
- ✅ Rescore handling (Postgres UPSERT `WHERE date_scored > existing`)
- ✅ Retest storage (separate rows per unique `opp_key`)
- ✅ RTS membership sync
- ✅ Test configuration management
- ✅ Full API fallback: ClickHouse unavailable → Postgres serves all queries
- ✅ Postgres aggregate_cache and materialized views for school/district scope
- ✅ Red Hat SSO authentication, multi-tenant isolation

### 13.2 Deferred Enhancements

| # | Enhancement | Notes |
|---|-------------|-------|
| 1 | State-level Postgres MV | `mv_state_overall` — nightly refresh only; too expensive to run more frequently at 5.5M student scale |
| 2 | Multi-opp selection rule | Best/latest rule per TestFamily when a student has multiple `OppKey`s for the same `TestKey` |
| 3 | Standard-level aggregates for school/district | Extend `aggregate_cache` and MVs for `component_type = 'STANDARD'` |
| 4 | Demographic slice reports | `student_attributes` JOIN already designed; add `aggregate_cache` rows keyed on demographic dimension |
| 5 | Reporting Category (RC) and Writing Dimension scores | Stored at ingest (`component_type = 'RC'`/`'WRITING_DIM'`); display views deferred |
| 6 | ClickHouse rehydration automation | Lambda to automatically detect ClickHouse divergence and re-populate from Postgres/S3 |
| 7 | Postgres query performance monitoring | Add slow-query alerts for fallback Postgres path; alert if district-scope live queries are triggered (indicates MV staleness) |
| 8 | Cross-year trend comparisons | Out of scope |

---

## 14. Tech Stack Reference

| Layer | Technology | Notes |
|-------|-----------|-------|
| Language | C# (.NET 8) | Lambda functions, API layer |
| Primary DB | Aurora PostgreSQL Serverless v2 | **Source of truth for all data**: scores, membership, config, aggregate cache |
| Analytical DB | ClickHouse 24.x (optional) | Self-hosted on EC2 r6i series; acceleration layer only |
| Messaging | AWS SQS (+ DLQ) | Score ingestion queue |
| Object storage | AWS S3 | Raw score files; ClickHouse backups |
| Compute | AWS Lambda | Score Processor, RTS Sync, API handlers, Aggregate Refresh |
| API | AWS API Gateway (HTTP API) | REST endpoints |
| Auth | Red Hat SSO (OIDC) | OIDC `id_token` (RS256); Lambda authorizer validates against RH SSO JWKS endpoint |
| Front-end | React (SPA) | Hosted on CloudFront |
| CDN | AWS CloudFront | SPA delivery |
| IaC | AWS CDK (C#) | Infrastructure as code |
| Postgres client | `Npgsql` NuGet v8.x | C# PostgreSQL driver; supports batch UPSERT and array parameters |
| ClickHouse client | `ClickHouse.Client` NuGet v7.x | C# ClickHouse ADO.NET driver |

### 14.1 Key NuGet Packages

```xml
<PackageReference Include="Npgsql"                      Version="8.*" />
<PackageReference Include="ClickHouse.Client"           Version="7.*" />
<PackageReference Include="Amazon.Lambda.Core"          Version="2.*" />
<PackageReference Include="Amazon.Lambda.SQSEvents"     Version="3.*" />
<PackageReference Include="AWSSDK.S3"                   Version="3.*" />
<PackageReference Include="Amazon.CDK.Lib"              Version="2.*" />
```

### 14.2 Postgres Connection & Batch UPSERT (C#)

```csharp
// Connection (read replica for report queries; writer for ingestion)
var connString = $"Host={auroraHost};Database=trs;Username=app;Password={password};" +
                 "Pooling=true;MinPoolSize=2;MaxPoolSize=20;CommandTimeout=30";

// Batch UPSERT using Npgsql binary COPY + upsert alternative:
// For score ingestion, use Npgsql's ExecuteNonQueryAsync with parameterized batches
await using var cmd = new NpgsqlCommand(upsertSql, connection);
// -- OR use UNNEST for set-based batch insert:
var upsertSql = @"
INSERT INTO trs.student_scores
  (tenant_id, school_year, test_group_id, student_id, opp_key, ...)
SELECT * FROM UNNEST($1::varchar[], $2::smallint[], $3::varchar[], $4::int[], $5::uuid[], ...)
ON CONFLICT (tenant_id, opp_key) DO UPDATE SET
  date_scored = EXCLUDED.date_scored, ...
WHERE EXCLUDED.date_scored > trs.student_scores.date_scored";
```

### 14.3 ClickHouse Bulk Insert (C# — unchanged from v1)

```csharp
var bulkCopy = new ClickHouseBulkCopy(connection)
{
    DestinationTableName = "trs.student_scores",
    BatchSize = 10_000
};
await bulkCopy.InitAsync();
// Best-effort: wrap in try/catch; failure does not affect Postgres primary write
try { await bulkCopy.WriteToServerAsync(rows); }
catch (Exception ex) { _logger.LogWarning(ex, "ClickHouse write failed; Postgres is authoritative"); }
```

---

*Document maintained by: Principal Software Architect*  
*Last updated: 2026-03-02*  
*Supersedes: TRS_Design_Document.md (v1)*
