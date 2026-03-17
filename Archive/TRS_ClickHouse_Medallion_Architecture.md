# TRS ClickHouse Medallion Architecture

| Field | Value |
| :--- | :--- |
| **Author** | Principal Software Architect |
| **Date** | 2026-03-03 |
| **Status** | Final for Implementation |
| **Project** | Teacher Reporting System (TRS) — Texas-Scale (5.5M Students) |

---

## Overview

The Medallion Architecture in ClickHouse transforms raw, volatile CDC events into high-performance, immutable reporting cubes. By using the `AggregatingMergeTree` engine at both the Silver and Gold layers, the system ensures that rescores are resolved **mathematically** rather than through expensive table mutations.

```
[Bronze Layer]                      [Silver Layer]                    [Gold Layer]
VersionedCollapsingMergeTree   →    AggregatingMergeTree         →   AggregatingMergeTree
Raw CDC sign-pair stream             Deduplicated per-student          School / District cubes
(-1/+1 rescore events)               opportunity state                 Pre-aggregated for UI
```

---

## Layer 1: Bronze (Raw CDC Stream)

The Bronze layer is the landing zone for all CDC events emitted by PeerDB. It stores raw sign-paired rows directly from the Write-Ahead Log transform.

- Engine: `VersionedCollapsingMergeTree`
- One row per CDC event, with `sign = -1` (undo) or `sign = +1` (redo) for every rescore
- **DELETE events** emit a single `sign = -1` row with `is_deleted = 1` — no `+1` counterpart
- Background merges physically collapse `-1/+1` pairs; a lone `-1` (delete) collapses against the original `+1` insert, physically removing the row
- Never queried directly by the API — feeds Silver and Gold via Materialized Views

---

## Layer 2: Silver (Deduplicated State)

The Silver layer is the **Clean Room** where the raw Bronze stream is collapsed into a single authoritative version per student opportunity.

The table uses `AggregateFunction` types to store the intermediate state of the `argMax` function. This allows ClickHouse to combine new inserts with existing data without losing the context of which score is actually the latest.

### 2.1 Silver Table Schema

```sql
CREATE TABLE trs.student_scores_silver
(
    tenant_id         String,
    school_year       UInt16,
    test_group_id     String,
    student_id        Int32,
    opp_key           UUID,
    -- Intermediate states for deduplication logic
    latest_score      AggregateFunction(argMax, Float32,      DateTime64(3)),
    latest_perf_level AggregateFunction(argMax, UInt8,        DateTime64(3)),
    is_eligible       AggregateFunction(argMax, UInt8,        DateTime64(3)),
    last_updated      AggregateFunction(max,    DateTime64(3)),
    -- Delete sentinel: argMax on date_scored means the latest event always wins.
    -- DELETE sets is_deleted=1 with the newest timestamp → wins until a re-insert
    -- arrives with an even newer timestamp and flips it back to 0.
    is_deleted        AggregateFunction(argMax, UInt8,        DateTime64(3))
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, test_group_id, student_id, opp_key);
```

### 2.2 Silver Materialized View

The Materialized View acts as the transformation trigger. It uses the `-State` suffix to convert incoming raw Bronze values into the aggregate states required by the Silver table.

```sql
CREATE MATERIALIZED VIEW trs.student_scores_silver_mv
TO trs.student_scores_silver AS
SELECT
    tenant_id,
    school_year,
    test_group_id,
    student_id,
    opp_key,
    argMaxState(score_value,             date_scored)  AS latest_score,
    argMaxState(overall_perf_level,      date_scored)  AS latest_perf_level,
    argMaxState(is_aggregate_eligible,   date_scored)  AS is_eligible,
    maxState(date_scored)                              AS last_updated,
    argMaxState(is_deleted,              date_scored)  AS is_deleted
FROM trs.student_scores_bronze
GROUP BY tenant_id, school_year, test_group_id, student_id, opp_key;
```

### 2.3 Querying Silver (Read Path)

Use the `-Merge` suffix combinators to materialise final values at read time:

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
-- Exclude soft-deleted opportunities; zero overhead unless row was actually deleted
HAVING argMaxMerge(is_deleted) = 0;
```

> **Role:** The Silver layer replaces per-student roster queries in ClickHouse. In the Postgres-primary architecture (see [CDC Architectures — Section 8](CDC%20Architectures_%20Debezium%20vs.%20PeerDB.md#8-cost-recalibration-postgres-primary-clickhouse-aggregation-only)), per-student reads go directly to Postgres; Silver is retained here for completeness and for future reporting features that require ClickHouse-side deduplication.

---

## Layer 3: Gold (Reporting Cubes)

The Gold layer consists of **specialized aggregation tables** optimised for specific UI components. Rather than a single master aggregate table, separate tables exist for School and District levels to:

1. Prevent double-counting of students across organisational boundaries.
2. Allow Dictionary Lookups to resolve the student-to-school mapping at write time.
3. Enable sub-10ms dashboard queries without `FINAL` or on-the-fly `GROUP BY`.

### 3.1 Gold School Table Schema

```sql
CREATE TABLE trs.school_aggregates_gold
(
    tenant_id       String,
    school_year     UInt16,
    school_id       String,
    test_group_id   String,
    -- Pre-calculated states for sub-10ms queries
    avg_score       AggregateFunction(avg,           Float32),
    student_count   AggregateFunction(uniq,          Int32),
    pl_distribution AggregateFunction(groupUniqArray, UInt8)
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, school_id, test_group_id);
```

### 3.2 Gold Materialized View with Dictionary Lookup

The Gold MV is where the **Sign-Pair math** and **Dictionary Lookups** converge. It dynamically attributes scores to the correct organisation based on the student's current roster status held in Postgres (exposed to ClickHouse via a dictionary).

```sql
CREATE MATERIALIZED VIEW trs.school_agg_mv
TO trs.school_aggregates_gold AS
SELECT
    tenant_id,
    school_year,
    -- Resolve school_id for this student from the Postgres-backed roster dictionary
    dictGet('trs.roster_dict', 'school_id', student_id) AS school_id,
    test_group_id,
    -- Multiply by sign: -1 undo rows subtract, +1 redo rows add
    avgState(score_value * sign)  AS avg_score,
    uniqState(student_id)         AS student_count
FROM trs.student_scores_bronze
WHERE is_aggregate_eligible = 1
GROUP BY tenant_id, school_year, school_id, test_group_id;
```

### 3.3 Querying Gold — School (Read Path)

```sql
SELECT
    school_id,
    test_group_id,
    avgMerge(avg_score)              AS average_score,
    uniqMerge(student_count)         AS students_tested,
    groupUniqArrayMerge(pl_distribution) AS perf_level_bands
FROM trs.school_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND school_id IN (?)
GROUP BY school_id, test_group_id
ORDER BY school_id;
```

---

### 3.4 Gold District Table Schema

The District table mirrors the School table but is keyed on `district_id`. It is a **separate physical table** — not a roll-up of the school table — to prevent double-counting students who may appear in multiple school rosters (e.g., transfers, shared programs).

```sql
CREATE TABLE trs.district_aggregates_gold
(
    tenant_id       String,
    school_year     UInt16,
    district_id     String,
    test_group_id   String,
    -- Performance level breakdown for bar/donut charts
    pl_1_count      AggregateFunction(sum,  Int64),
    pl_2_count      AggregateFunction(sum,  Int64),
    pl_3_count      AggregateFunction(sum,  Int64),
    pl_4_count      AggregateFunction(sum,  Int64),
    avg_score       AggregateFunction(avg,  Float32),
    student_count   AggregateFunction(uniq, Int32)
)
ENGINE = AggregatingMergeTree()
ORDER BY (tenant_id, school_year, district_id, test_group_id);
```

> **Why separate PL count columns instead of `groupUniqArray`?** At district scale (tens of thousands of students per district) `groupUniqArray` materialises the full student ID array in memory just to count it. Using `sumState` over pre-bucketed `sign`-adjusted indicator columns is orders of magnitude cheaper and still supports Sign-Pair subtraction.

### 3.5 Gold District Materialized View

The District MV uses a **two-level dictionary lookup** — first resolving `school_id` from `roster_dict`, then resolving `district_id` from `school_district_dict` — ensuring each student score is attributed to the correct district regardless of mid-year transfers.

```sql
CREATE MATERIALIZED VIEW trs.district_agg_mv
TO trs.district_aggregates_gold AS
SELECT
    tenant_id,
    school_year,
    -- Two-hop dictionary lookup: student_id → school_id → district_id
    dictGet(
        'trs.school_district_dict',
        'district_id',
        dictGet('trs.roster_dict', 'school_id', student_id)
    ) AS district_id,
    test_group_id,
    -- Sign-pair PL indicator columns: sign=-1 subtracts, sign=+1 adds
    sumState(if(overall_perf_level = 1, sign, 0))  AS pl_1_count,
    sumState(if(overall_perf_level = 2, sign, 0))  AS pl_2_count,
    sumState(if(overall_perf_level = 3, sign, 0))  AS pl_3_count,
    sumState(if(overall_perf_level = 4, sign, 0))  AS pl_4_count,
    avgState(score_value * sign)                    AS avg_score,
    uniqState(student_id)                           AS student_count
FROM trs.student_scores_bronze
WHERE is_aggregate_eligible = 1
GROUP BY tenant_id, school_year, district_id, test_group_id;
```

### 3.6 Querying Gold — District (Read Path)

```sql
SELECT
    district_id,
    test_group_id,
    avgMerge(avg_score)      AS average_score,
    uniqMerge(student_count) AS students_tested,
    sumMerge(pl_1_count)     AS pl_1,
    sumMerge(pl_2_count)     AS pl_2,
    sumMerge(pl_3_count)     AS pl_3,
    sumMerge(pl_4_count)     AS pl_4,
    -- Percentage breakdown for UI charts (avoid division by zero)
    round(sumMerge(pl_1_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_1_pct,
    round(sumMerge(pl_2_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_2_pct,
    round(sumMerge(pl_3_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_3_pct,
    round(sumMerge(pl_4_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_4_pct
FROM trs.district_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND district_id IN (?)
GROUP BY district_id, test_group_id
ORDER BY district_id;
```

### 3.7 Dictionary Definitions

Both Gold MVs depend on two ClickHouse dictionaries backed by Aurora Postgres.

#### `trs.roster_dict` — Student → School mapping

```sql
CREATE DICTIONARY trs.roster_dict
(
    student_id  Int32,
    school_id   String
)
PRIMARY KEY student_id
SOURCE(POSTGRESQL(
    host        'aurora-cluster.cluster-xxxx.us-east-1.rds.amazonaws.com'
    port        5432
    user        'trs_readonly'
    password    '...'
    db          'trs'
    table       'v_student_school_current'   -- view returning current enrolment only
))
LIFETIME(MIN 300 MAX 600)   -- refresh every 5–10 min; covers mid-year transfers
LAYOUT(HASHED());
```

#### `trs.school_district_dict` — School → District mapping

```sql
CREATE DICTIONARY trs.school_district_dict
(
    school_id   String,
    district_id String
)
PRIMARY KEY school_id
SOURCE(POSTGRESQL(
    host        'aurora-cluster.cluster-xxxx.us-east-1.rds.amazonaws.com'
    port        5432
    user        'trs_readonly'
    password    '...'
    db          'trs'
    table       'schools'
))
LIFETIME(MIN 3600 MAX 7200)  -- school→district mapping changes rarely
LAYOUT(HASHED());
```

> **Transfer student handling:** Dictionary `LIFETIME` controls how stale a student's school attribution can be. A 5-minute lifetime means a transferred student's scores will be re-attributed to their new school within the next dictionary refresh. Historical Bronze rows retain the old attribution permanently — only new inserts use the refreshed mapping. This is the correct behaviour: historical aggregates reflect enrolment at time of score; current aggregates reflect current enrolment.

---

## Layer Comparison

| Feature | Bronze | Silver | Gold — School | Gold — District |
| :--- | :--- | :--- | :--- | :--- |
| **Engine** | `VersionedCollapsingMergeTree` | `AggregatingMergeTree` | `AggregatingMergeTree` | `AggregatingMergeTree` |
| **Granularity** | 1 row per CDC event | 1 row per student opportunity | 1 row per school/test | 1 row per district/test |
| **Primary Question** | "What changed and when?" | "What is the latest score per student?" | "What is the school average?" | "What is the district PL breakdown?" |
| **Query Type** | Not queried directly | Roster-level point reads | School dashboard reads | District dashboard reads |
| **Key Functions** | `sign` + `VersionedCollapsing` | `argMaxState` / `argMaxMerge` | `avgState`, `uniqState`, `groupUniqArrayState` | `sumState` (PL buckets), `avgState`, `uniqState` |
| **Dictionary Lookup** | None | None | `roster_dict` (student→school) | `roster_dict` + `school_district_dict` (student→school→district) |
| **Rescore Handling** | Signs cancel via background merge | `argMax` keeps highest `date_scored` | `sign * score_value` cancels old value | `sign`-adjusted PL indicator columns cancel atomically |

---

## Sign-Pair Event Flows (End-to-End)

### UPDATE (Rescore)

```
Postgres UPDATE  →  PeerDB (Before + After images)
        │
        ▼
C# Transformer
  ├─ INSERT (opp_key, old_score, sign=-1, is_deleted=0, date_scored=T1)  ← Undo
  └─ INSERT (opp_key, new_score, sign=+1, is_deleted=0, date_scored=T2)  ← Redo
        │
        ▼
Bronze — background merge cancels [-1,+1] pair → net: new_score survives
        │
        ├──▶ Silver MV:        argMaxState(T2 > T1) → new score wins; is_deleted=0
        ├──▶ Gold School MV:   avgState(score * sign) → school avg corrected atomically
        └──▶ Gold District MV: sumState(if(pl=N,sign,0)) → PL band corrected atomically
```

### DELETE

```
Postgres DELETE  →  PeerDB (Before image only — no After)
        │
        ▼
C# Transformer
  └─ INSERT (opp_key, old_score, sign=-1, is_deleted=1, date_scored=T_delete)  ← Undo only
        │
        ▼
Bronze — lone -1 collapses against original +1 → row physically removed on merge
        │
        ├──▶ Silver MV:        argMaxState(T_delete is latest) → is_deleted=1 wins
        │                       HAVING argMaxMerge(is_deleted)=0 hides row at read time
        ├──▶ Gold School MV:   avgState(score * -1) → score subtracted from school avg ✅
        └──▶ Gold District MV: sumState(if(pl=N,-1,0)) → PL band decremented ✅
```

### DELETE → Re-Insert

```
T1: Postgres INSERT  → Bronze +1 (is_deleted=0)
T2: Postgres DELETE  → Bronze -1 (is_deleted=1)  Silver: is_deleted=1 wins (T2>T1)
T3: Postgres INSERT  → Bronze +1 (is_deleted=0)  Silver: is_deleted=0 wins (T3>T2) ✅
                                                  Gold:   score re-added correctly ✅
```

---

## CDC Event Type Reference

| CDC Event | Before | After | Signs Emitted | `is_deleted` | Bronze Result | Silver Result | Gold Result |
| :--- | :---: | :---: | :---: | :---: | :--- | :--- | :--- |
| **INSERT** | ❌ | ✅ | `+1` | `0` | Row added | `argMax` picks new row | Score added to avg / PL |
| **UPDATE (rescore)** | ✅ | ✅ | `-1` + `+1` | `0` | Pair collapses → net new score | `argMax` picks T_after row | Old subtracted, new added |
| **DELETE** | ✅ | ❌ | `-1` only | `1` | Collapses against original `+1` | `is_deleted=1` wins; hidden by `HAVING` | Score subtracted ✅ |
| **DELETE → re-INSERT** | ✅→✅ | ❌→✅ | `-1` then `+1` | `1` then `0` | Net zero then re-added | `is_deleted=0` wins at T_reinsert | Score re-added ✅ |

---

## Prerequisites for Success

Two operational requirements must be enforced before PeerDB replication is enabled. Skipping either will cause silent data corruption.

### 1. `REPLICA IDENTITY FULL` on Aurora Postgres

Without this, Postgres only writes the primary key columns to the WAL on DELETE and UPDATE events. PeerDB will deliver a Before image with `null` for every non-key column — the C# Transformer will have nothing to subtract from the aggregates.

```sql
-- Run once per replicated table on the Aurora Postgres instance
ALTER TABLE trs.student_opportunities REPLICA IDENTITY FULL;
ALTER TABLE trs.student_scores        REPLICA IDENTITY FULL;
```

> **WAL size impact:** `REPLICA IDENTITY FULL` increases WAL volume because every UPDATE and DELETE writes the full row image. At TRS scale this is acceptable — score mutations are low-frequency relative to read volume.

### 2. Millisecond Precision on `date_scored` / `last_updated`

`argMaxState` resolves ties by the version timestamp. If two events (e.g., a rapid rescore and its acknowledgement) arrive with the same timestamp, `argMax` picks arbitrarily and the wrong score may be retained permanently.

```sql
-- Correct — use DateTime64(3) everywhere a version key appears
date_scored   DateTime64(3),   -- millisecond precision
last_updated  DateTime64(3)

-- Wrong — DateTime (second precision) creates tie windows of up to 1000ms
date_scored   DateTime
```

> Enforce `DateTime64(3)` in the C# Transformer's Bronze insert payload, the Silver schema, and the upstream Postgres column definition.

---

## Safety Net Summary

The architecture is designed to be self-healing. Every known failure scenario has a deterministic recovery path with no manual data repair.

| Failure Scenario | How the Design Survives |
| :--- | :--- |
| **Out-of-order events** | `argMaxState` on `date_scored` ensures the latest timestamp always wins, regardless of delivery order. |
| **Duplicate events** | `VersionedCollapsingMergeTree` (Bronze) and sign-pair math make duplicate inserts idempotent — a second `+1` for the same `opp_key` + `date_scored` pair nets to zero on merge. |
| **Delete then re-insert** | `is_deleted` flips from `1` back to `0` when the re-insert timestamp exceeds the delete timestamp. Fully automatic. |
| **ClickHouse downtime** | Postgres WAL buffers all CDC events in the PeerDB replication slot until ClickHouse is back online. API falls back to Postgres Tier 1/3 transparently. |
| **PeerDB Fargate restart** | PeerDB reconnects to the same replication slot and resumes from its last confirmed LSN. No data loss. |
| **Total EC2 loss** | PeerDB Initial Load re-snapshots Aurora into the new instance, then switches to streaming. Postgres is the source of truth; ClickHouse is always rebuildable. |

---

## Next Steps

- [ ] Set `REPLICA IDENTITY FULL` on all replicated Aurora Postgres tables before enabling PeerDB.
- [ ] Confirm `date_scored` column is `DateTime64(3)` in Postgres, Bronze, and Silver — not plain `DateTime`.
- [ ] Create Aurora view `v_student_school_current` returning active enrolment only (used by `roster_dict`).
- [ ] Deploy `trs.roster_dict` (student → school, 5-min refresh) and `trs.school_district_dict` (school → district, 1-hr refresh).
- [ ] Deploy Bronze, Silver, and both Gold tables + all four Materialized Views in order: Bronze → Silver MV → School Gold MV → District Gold MV.
- [ ] Implement C# `BackgroundService` query layer using `argMaxMerge` (Silver), `avgMerge` + `uniqMerge` (School Gold), and `sumMerge` + PL percentage logic (District Gold).
- [ ] Validate Sign-Pair idempotency under duplicate CDC delivery using production-scale replay tests.
- [ ] Validate transfer-student attribution: insert a roster change, wait for `roster_dict` refresh, confirm new scores land in new school/district aggregate.
- [ ] Add demographic slicing (gender, ethnicity, ELL) to both Gold tables as additional `ORDER BY` dimensions or via separate cube tables once base aggregates are validated.
