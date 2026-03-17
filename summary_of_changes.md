# TRS Design Document — Summary of Changes (v5 → v6)

**Prepared:** 2026-03-17
**Based on:** Design review conversation covering Issues & Risks 1–8

---

## Change Categories

- **[CORRECTNESS]** — Fixes a bug that would cause incorrect aggregate scores or counts
- **[SCHEMA]** — New or modified table/column DDL
- **[ARCH]** — Architectural change
- **[FIX]** — Document error corrected (naming, placeholder, gap)
- **[OPS]** — Operational / monitoring improvement

---

## 1. Remove Gold Streaming MVs — Silver-Based Aggregation [CORRECTNESS] [ARCH]

### Problem (Issues 1, 2, 3)

The v5 Gold layer (`school_aggregates_gold`, `district_aggregates_gold`, `component_aggregates_gold`) was populated via streaming Materialized Views that fired on every Bronze INSERT. These MVs had three correctness defects:

| # | Defect | Root Cause |
|---|--------|------------|
| 1 | Membership attribution never updates | JOIN on `membership_school_mirror` bakes school attribution at Bronze INSERT time; student transfers are never reflected |
| 2 | Double-count on rescore | `uniqState(student_id)` cannot subtract on sign=-1 rows; each rescore increments the distinct-student counter even though it's the same student |
| 3 | `avgState` denominator inflation | `avgState(score * sign)` increments the denominator on both the undo (-1) and redo (+1) row of a rescore pair; average is permanently diluted |

### Fix

**Remove all three Gold streaming MVs and their physical tables.** Replace with Silver-based aggregation:

- Silver layer holds **per-student, per-opportunity** deduplicated state (no school/district attribution).
- Membership JOIN happens **at query time** against current `membership_school_mirror` / `membership_district_mirror`.
- Aggregate Refresh Lambda now queries Silver + membership JOIN → writes to `report_cache`.
- API Tier 2 now queries Silver + membership JOIN directly (not Gold).

### Tables Removed

- `trs.school_aggregates_gold` (physical table)
- `trs.bronze_to_school_gold_mv` (streaming MV)
- `trs.district_aggregates_gold` (physical table)
- `trs.bronze_to_district_gold_mv` (streaming MV)
- `trs.component_aggregates_gold` (physical table)
- `trs.bronze_to_component_gold_mv` (streaming MV)

### New Silver Read Query — School Scope (Overall Scores)

```sql
-- Used by Aggregate Refresh Lambda (all schools batch) and API Tier 2 (single school)
SELECT
    m.school_id,
    count(*)                      AS students_tested,
    avg(s.latest_score)           AS avg_score,
    countIf(s.perf_level = 1)     AS pl_1,
    countIf(s.perf_level = 2)     AS pl_2,
    countIf(s.perf_level = 3)     AS pl_3,
    countIf(s.perf_level = 4)     AS pl_4
    -- repeat countIf for pl_5 … pl_10 as needed
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

**Why this is correct:**
- Multi-enrolled students: the JOIN produces N rows (one per school/district); each aggregate correctly counts the student.
- Score arrived before membership: score sits in Silver; next Lambda run JOINs current membership state.
- Rescore: `argMaxMerge(latest_score, date_scored)` picks the latest score; no denominator inflation; no double-count.
- Retest: `HAVING use_for_agg = 1` includes only the selected opp per student; one row per student contributes to aggregates.

---

## 2. Retest Handling — `use_for_aggregation` Flag [SCHEMA] [ARCH]

### Problem

A student can take a test up to 3 times (different `opp_key` per attempt). v5 had no mechanism to select which attempt counts toward aggregates. Deferred as Enhancement #2 in v5 §17.2.

### Fix — Promoted to MVP

#### `trs.test_aliases` — new column

```sql
ALTER TABLE trs.test_aliases
    ADD COLUMN retest_selection_rule TEXT NULL
        CHECK (retest_selection_rule IN ('latest_date', 'highest_score'));
-- NULL = retests not permitted for this test alias
```

#### `trs.student_opportunities` — two new columns

```sql
ALTER TABLE trs.student_opportunities
    ADD COLUMN use_for_aggregation        BOOLEAN      NOT NULL DEFAULT TRUE,
    ADD COLUMN use_for_aggregation_set_at TIMESTAMPTZ  NOT NULL DEFAULT NOW();
```

- `use_for_aggregation = TRUE` → this opp is included in aggregates.
- `use_for_aggregation = FALSE` → this opp is excluded (superseded by a later retest).
- `use_for_aggregation_set_at` → version key for the flag; updated to `NOW()` whenever the flag changes. Required because `date_scored` does not change when the flag is toggled, creating an `argMaxState` tie.

#### Updated index

```sql
CREATE INDEX idx_so_aggregation
    ON trs.student_opportunities (tenant_id, school_year, testalias_id, student_id)
    WHERE is_aggregate_eligible = TRUE AND use_for_aggregation = TRUE;
```

#### Score Processor Lambda behavior on retest arrival

```
New retest opp arrives (new opp_key, same student + testalias_id):

  1. INSERT new opp with use_for_aggregation = TRUE
  2. In same transaction:
     UPDATE existing opp(s) SET use_for_aggregation = FALSE,
                                 use_for_aggregation_set_at = NOW()
     WHERE tenant_id = ? AND school_year = ? AND testalias_id = ? AND student_id = ?
       AND opp_key != new_opp_key

  Single transaction guarantees no window where two opps are simultaneously TRUE after commit.
  CDC propagates both events in LSN order.
```

#### ClickHouse Bronze additions

```sql
-- Added to student_scores_bronze:
use_for_aggregation        UInt8         DEFAULT 1,
use_for_aggregation_set_at DateTime64(3) DEFAULT now64(3)
```

#### ClickHouse Silver additions

```sql
-- Added to student_scores_silver:
use_for_aggregation AggregateFunction(argMax, UInt8, DateTime64(3))

-- Added to student_scores_silver_mv SELECT:
argMaxState(use_for_aggregation, use_for_aggregation_set_at) AS use_for_aggregation
```

**Why `use_for_aggregation_set_at` instead of `date_scored`:**
When `use_for_aggregation` changes via UPDATE, the CDC undo and redo rows both carry the original `date_scored` (the score timestamp is unchanged). `argMaxState(use_for_aggregation, date_scored)` would be non-deterministic on the tie. `use_for_aggregation_set_at` is a separate column updated to `NOW()` on every flag change, guaranteeing the latest flag state always wins.

---

## 3. Component Scores — Silver Layer [SCHEMA] [ARCH]

### Problem

v5 had `component_aggregates_gold` populated via a streaming MV with the same correctness defects as the overall score Gold MVs (membership JOIN at INSERT time, `avgState` inflation, `uniqState` double-count).

### Fix

Add a Silver layer for component scores. Remove component Gold streaming MV. Use Silver + opp_key JOIN at query time to inherit `use_for_aggregation` from parent opportunity.

#### New: `trs.student_component_scores_silver`

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

**Component scores do NOT get their own `use_for_aggregation` flag.** They inherit selection from the parent `opp_key` via a JOIN against `student_scores_silver` at query time. This avoids CDC explosion (10–30 component UPDATE events per opp per retest flag change).

#### Optimized Component Score Query — School Scope

```sql
WITH school_students AS (
    SELECT DISTINCT student_id
    FROM trs.membership_school_mirror FINAL
    WHERE tenant_id = ? AND school_id = ? AND is_deleted = 0
),
selected_opps AS (
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
       avg(scale_score)        AS avg_scale_score,
       count(*)                AS students_tested,
       countIf(perf_level = 1) AS pl_1,
       countIf(perf_level = 2) AS pl_2,
       countIf(perf_level = 3) AS pl_3,
       countIf(perf_level = 4) AS pl_4
FROM component_resolved
GROUP BY component_type, component_id
ORDER BY component_type, component_id;
```

**Performance (school scope, TX scale):**
- Push school membership filter first (~6,800 students for largest TX school).
- Component rows touched: ~100–136k (6,800 students × 15–20 components).
- Tier 2 live query: 80–250ms.
- Lambda batch (all schools, one testalias): 1–4s query + ~200ms cache write.
- **District component queries**: add 5s timeout gate; return HTTP 503 if Silver + JOIN exceeds limit.
- **State component aggregates**: nightly Lambda only (82–110M rows at TX scale — not viable live).

---

## 4. `trs.score_ingest_rejections` DDL Added [FIX] [SCHEMA]

### Problem (Issue 4)

§8.5 referenced `trs.score_ingest_rejections` for audit logging and replay tracking but the DDL was deferred to `implementation/Score_Reject_Handling.md`. The main document had no schema definition, making the replay runbook unverifiable.

### Fix — DDL added to §6.8

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

---

## 5. `trs.test_configs` → `trs.test_keys` Naming Fix [FIX]

### Problem (Issue 5)

Two sections referenced a table `trs.test_configs` which does not exist in the schema:

| Location | Incorrect reference | Correct table |
|----------|--------------------|-|
| §8.5 rejection reason `UNKNOWN_TEST_CONFIG` | "no matching row in `trs.test_configs`" | `trs.test_keys` |
| §9.5 EmbargoService lookup | "`embargo_until` for the `testalias_id` from `trs.test_configs`" | `trs.test_aliases` |

### Fix

- §8.5 rejection code renamed: `UNKNOWN_TEST_CONFIG` → `UNKNOWN_TEST_KEY`; description updated to reference `trs.test_keys`.
- §9.5 EmbargoService lookup source corrected to `trs.test_aliases` (the table that actually holds `embargo_until`).

---

## 6. Postgres MV — Complete `pl_1` through `pl_10` Columns [FIX]

### Problem (Issue 6)

`mv_school_overall`, `mv_district_overall`, and `mv_school_standards_current` DDL used `....` / `...` placeholders for performance level columns pl_4 through pl_9, creating ambiguity about the actual schema.

### Fix

All placeholders replaced with explicit columns `pl_1` through `pl_10`:

```sql
-- School and district MVs — full perf level columns
COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)  AS pl_1,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)  AS pl_2,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)  AS pl_3,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)  AS pl_4,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)  AS pl_5,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 6)  AS pl_6,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 7)  AS pl_7,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 8)  AS pl_8,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 9)  AS pl_9,
COUNT(*) FILTER (WHERE ss.overall_perf_level = 10) AS pl_10,
```

Also removed stray `AVG(ss.overall_percent_correct) AS avg_percent_correct` reference — this column does not exist in `trs.student_opportunities`.

---

## 7. ClickHouse Health Probe — Circuit Breaker [OPS]

### Problem (Issue 7)

v5 §9.1 described a single 200ms timeout probe on `GET /ping` before each Tier 2 query. During a sustained ClickHouse outage, every API request incurred a 200ms penalty before falling through to Tier 3. No cached health state.

### Fix — Circuit Breaker Pattern

The API Lambda implements an in-process circuit breaker with three states:

| State | Behavior | Transition |
|-------|----------|------------|
| **CLOSED** | Normal; Tier 2 queries attempted | → OPEN on probe failure |
| **OPEN** | Skip Tier 2 entirely; go directly to Tier 3 | → HALF_OPEN after 30s TTL |
| **HALF_OPEN** | Allow one probe attempt | → CLOSED on success; → OPEN on failure |

**Implementation:**
- Health state cached in-process per Lambda container (not shared across containers — acceptable; each container independently discovers health within 30s).
- OPEN TTL: 30 seconds (configurable via environment variable `CH_CIRCUIT_OPEN_TTL_SEC`).
- On first probe failure: immediately transition to OPEN; no retry.
- CloudWatch metric `TRS/API/CircuitBreakerOpen` emitted when state transitions to OPEN.
- Alarm: `TRS/API/CircuitBreakerOpen` sustained > 5 minutes → SNS → ops.

---

## 8. WAL Replication Slot Monitoring Alert [OPS]

### Problem (Issue 8)

v5 §16.3 documented `max_slot_wal_keep_size = 10GB` as a safety cap but defined no CloudWatch alarm for when WAL retention approaches the cap. If the cap is hit, the replication slot is **invalidated** and PeerDB requires a full Initial Load snapshot to recover — a 30–90 minute rebuild at TX scale.

### Fix — WAL Slot Size Monitoring Lambda

A new lightweight monitoring Lambda runs every 5 minutes via EventBridge Scheduler:

```sql
-- Queries Aurora via read replica connection
SELECT
    slot_name,
    active,
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) / 1024 / 1024 AS slot_lag_mb
FROM pg_replication_slots
WHERE slot_name = 'peerdb_trs_slot';
```

Emits custom CloudWatch metric: `TRS/CDC/WALSlotSizeMB` (namespace `TRS/CDC`, dimension `SlotName`).

**Alarm:**

| Threshold | Action |
|-----------|--------|
| > 8,192 MB (80% of 10 GB cap) | CloudWatch alarm → SNS → ops (`WARN`: investigate PeerDB lag) |
| > 9,216 MB (90% of cap) | CloudWatch alarm → SNS → ops (`CRITICAL`: imminent slot invalidation) |

Added to §14 (Operational Prerequisites) and §16 (Tech Stack — Observability metrics list).

---

## 9. Architecture Diagram Update [ARCH]

The v5 diagram showed:
```
Silver ──▶ Gold (school/district cubes)
           ↑
           JOIN membership at MV write time
```

v6 diagram shows:
```
Silver (per-student state, no attribution)
    +
membership_school_mirror  ──▶  JOIN at query time  ──▶  aggregates
membership_district_mirror
```

Gold layer box removed from ClickHouse section. API Tier 2 label updated from `ClickHouse Gold live query (<200ms)` to `ClickHouse Silver + membership JOIN (<250ms school; <2s district)`.

---

## 10. Updated Key Engineering Decisions

| # | Decision | Change |
|---|----------|--------|
| 6 | ~~No streaming MVs for pre-aggregation~~ | **Updated**: Gold streaming MVs removed entirely. Root cause: membership JOIN at INSERT time + `avgState`/`uniqState` defects. Silver-based query-time aggregation replaces Gold. |
| New | **Silver is the aggregation source** | Per-student state (no attribution) in Silver; membership JOIN at query time. Correctness guaranteed for transfers, multi-enrollment, rescores, and retests. |
| New | **`use_for_aggregation` flag for retest selection** | Score Processor sets flag in single Postgres transaction (INSERT new + UPDATE old → FALSE). `use_for_aggregation_set_at` prevents argMaxState tie when flag changes. |
| New | **Component scores inherit retest selection via opp_key JOIN** | No `use_for_aggregation` on component rows; JOIN component Silver against overall score Silver at query time. Avoids 10–30 CDC UPDATE events per opp per retest. |

---

## 11. ClickHouse Deployment Order Update [OPS]

v6 deployment order removes Gold tables/MVs and adds component Silver:

1. `trs.student_scores_bronze`
2. `trs.student_scores_silver`
3. `trs.student_scores_silver_mv`
4. `trs.student_component_scores_bronze`
5. `trs.student_component_scores_silver`
6. `trs.student_component_scores_silver_mv`
7. `trs.membership_school_mirror`
8. `trs.membership_district_mirror`
9. `trs.student_attributes`
10. `trs.student_scope_today` (diagnostic only)

**Removed from deployment order:** `school_aggregates_gold`, `bronze_to_school_gold_mv`, `district_aggregates_gold`, `bronze_to_district_gold_mv`, `component_aggregates_gold`, `bronze_to_component_gold_mv`.

---

## 12. MVP Scope Update

| Item | v5 Status | v6 Status |
|------|-----------|-----------|
| Retest selection rule (`use_for_aggregation`) | Deferred (Enhancement #2) | **MVP** |
| School / district component aggregates (standards, RC) | Deferred (Enhancement #3) | **MVP** |
| Standard-level ClickHouse queries via Silver | Not described | **MVP** |

---

## Summary Table

| Issue # | Description | Fix Type | Sections Affected |
|---------|-------------|----------|-------------------|
| 1 | Gold MV membership JOIN baked at write time | ARCH — Silver + query-time JOIN | §2, §7, §8.4, §9 |
| 2 | `uniqState` double-count on rescore | ARCH — Silver `count(*)` after dedup | §7, §8.4, §9 |
| 3 | `avgState` denominator inflation | ARCH — Silver `argMaxMerge` | §7, §8.4, §9 |
| 4 | `score_ingest_rejections` DDL missing | SCHEMA — DDL added | §6.8 (new) |
| 5 | `test_configs` vs `test_keys` naming | FIX — rename + correct table refs | §8.5, §9.5 |
| 6 | Postgres MV pl_4–pl_10 placeholder | FIX — explicit columns | §6.5, §6.6, §6.7 |
| 7 | No circuit breaker on health probe | OPS — circuit breaker pattern | §9.1, §14 |
| 8 | No WAL slot size alarm | OPS — monitoring Lambda + alarm | §14, §16 |
| — | Retest handling (`use_for_aggregation`) | SCHEMA + ARCH | §5.3, §5.4 (new), §6.1, §6.3, §7.1, §7.2 |
| — | Component Silver layer | SCHEMA + ARCH | §7.3 (new), §8.4 |
