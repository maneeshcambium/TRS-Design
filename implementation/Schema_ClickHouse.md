# TRS — ClickHouse Medallion Schema (DDL Reference)

Full DDL for all ClickHouse tables and Materialized Views.
See [TRS_Design_Document_v4.md](../TRS_Design_Document_v4.md) §7 for design rationale.

## Deployment Order

Deploy tables and MVs in this strict order (MVs must be created after their source tables exist):

1. `trs.student_scores_bronze`
2. `trs.student_scores_silver`
3. `trs.student_scores_silver_mv` (Bronze → Silver)
4. `trs.membership_school_mirror`
5. `trs.membership_district_mirror`
6. `trs.school_aggregates_gold`
7. `trs.bronze_to_school_gold_mv` (Bronze + school mirror → Gold School)
8. `trs.district_aggregates_gold`
9. `trs.bronze_to_district_gold_mv` (Bronze + district mirror → Gold District)
10. `trs.student_attributes`
11. `trs.student_scope_today` (diagnostic only; no dependents)

---

## Layer 1: Bronze (Raw CDC Stream)

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

---

## Layer 2: Silver (Deduplicated Per-Student State)

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

---

## Layer 3: Gold — School Aggregates

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

---

## Layer 3: Gold — District Aggregates

A **separate physical table** from School Gold — not a roll-up — to prevent double-counting
students enrolled in multiple schools (transfers, shared programs).

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

---

## Membership Mirror Tables

PeerDB replicates `school_student` → `membership_school_mirror` and
`district_student` → `membership_district_mirror` via dedicated CDC replication jobs.
`version` = `_peerdb_version` (Postgres LSN); higher LSN wins. `is_deleted = 1` handles deletes.

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

---

## Auxiliary Tables

### `trs.student_attributes` (Demographics)

```sql
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

### `trs.student_scope_today` (Diagnostic Only)

> **Not used in any Materialized View.** Retained for ad-hoc investigations only.

```sql
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

---

*See [TRS_Design_Document_v4.md](../TRS_Design_Document_v4.md) for full design rationale.*
