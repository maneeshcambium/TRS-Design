# ClickHouse Gold Layer — School & District Aggregates

**Status:** Design Proposal — **Tier 2 (Deferred)**
**Date:** 2026-03-18
**Context:** Supplement to TRS Design Document v6, Section 7 (ClickHouse Medallion Architecture)

> **Purpose:** This document describes a two-tiered approach to school and district
> aggregate queries in ClickHouse:
>
> - **Tier 1 (Launch):** Silver `AggregatingMergeTree` + `membership_school_mirror FINAL`
>   JOIN at query time. No Gold layer. Simpler to deploy and operate. Correct by default.
>
> - **Tier 2 (Deferred — deploy if Tier 1 performance is insufficient):** Gold layer using
>   `AggregatingMergeTree` with `sumState` / `sumMapState` partial aggregate states. Two
>   streaming Materialized Views — one from Bronze (score changes), one from the membership
>   mirror (membership changes) — maintain Gold using sign semantics. No direct DELETEs.
>   No ad-hoc Lambdas. Same append-only, CDC-driven pipeline.
>
> **Decision:** Ship Tier 1 first. Gold is an additive optimization that can be deployed
> at any time without changing Bronze, Silver, or the membership mirrors.

---

## Table of Contents

1. [Tiered Approach — Launch vs. Deferred](#1-tiered-approach--launch-vs-deferred)
2. [Motivation for Gold (Tier 2)](#2-motivation-for-gold-tier-2)
3. [Why Streaming MVs Were Previously Rejected](#3-why-streaming-mvs-were-previously-rejected)
4. [Why This Design Is Different](#4-why-this-design-is-different)
5. [Membership CDC Path — How Membership Reaches ClickHouse](#5-membership-cdc-path--how-membership-reaches-clickhouse)
6. [Gold Table Schema](#6-gold-table-schema)
7. [MV1 — Bronze → Gold (Score Changes)](#7-mv1--bronze--gold-score-changes)
8. [MV2 — Membership Mirror → Gold (Membership Changes)](#8-mv2--membership-mirror--gold-membership-changes)
9. [District Gold (Identical Pattern)](#9-district-gold-identical-pattern)
10. [Query-Time Read](#10-query-time-read)
11. [Correctness Matrix](#11-correctness-matrix)
12. [Why sumState Instead of uniqState / countState / avgState](#12-why-sumstate-instead-of-uniqstate--countstate--avgstate)
13. [Timing & Edge Cases](#13-timing--edge-cases)
14. [Pitfalls](#14-pitfalls)
15. [Performance Impact](#15-performance-impact)
16. [Medallion Layer Summary (Updated)](#16-medallion-layer-summary-updated)
17. [Deployment Order](#17-deployment-order)

---

## 1. Tiered Approach — Launch vs. Deferred

### Tier 1 — Silver + Membership JOIN (Launch Default)

At launch, school and district aggregates are computed at query time by JOINing Silver
(`AggregatingMergeTree`) against the membership mirror (`ReplacingMergeTree FINAL`):

```sql
-- Tier 1 query: school aggregates from Silver + membership JOIN
SELECT
    m.school_id,
    count(*)                                              AS students_tested,
    avg(argMaxMerge(s.latest_score))                      AS avg_scale_score,
    avg(argMaxMerge(s.latest_raw_score))                  AS avg_raw_score
FROM trs.student_scores_silver AS s FINAL
INNER JOIN trs.membership_school_mirror FINAL AS m
    ON s.tenant_id = m.tenant_id
    AND s.student_id = m.student_id
WHERE s.tenant_id = {tenant_id:String}
  AND s.school_year = {school_year:UInt16}
  AND s.testalias_id = {testalias_id:String}
  AND argMaxMerge(s.is_deleted)          = 0
  AND argMaxMerge(s.is_eligible)         = 1
  AND argMaxMerge(s.use_for_aggregation) = 1
GROUP BY m.school_id;
```

**Properties:**
- **Always correct** — `FINAL` on both tables guarantees latest state. No stale partial
  states, no drift, no sign arithmetic to reason about.
- **No additional tables or MVs** — Bronze, Silver, and membership mirrors are already
  deployed for student-level queries. School/district aggregates ride on the same
  infrastructure.
- **Membership changes reflected immediately** — `FINAL` resolves the latest membership
  at query time. No MV2 propagation delay.
- **Simpler operations** — fewer moving parts to monitor, no Gold backfill step, no
  floating-point drift maintenance.

**Estimated performance (TX scale, single test):**
- Rows scanned: ~5.5M student rows + ~5.5M membership rows
- Estimated latency: 200–800 ms (depending on ClickHouse cluster size and concurrent load)
- Acceptable for API responses backed by `report_cache` (Tier 1 cache → Tier 2 Silver
  query → Tier 3 Postgres fallback)

### Tier 2 — Gold AggregatingMergeTree (Deploy If Needed)

If Tier 1 query latency becomes unacceptable, deploy the Gold layer described in sections
6–17 of this document. Gold pre-aggregates partial states per school/district × test,
reducing query-time row scans from O(students) to O(schools).

**Estimated performance with Gold (TX scale, single test):**
- Rows scanned: ~8,000 school rows (vs. ~5.5M)
- Estimated latency: < 5 ms (vs. 200–800 ms)

### When to Escalate from Tier 1 to Tier 2

| Trigger | Threshold | Why it matters |
|---------|-----------|---------------|
| Tier 2 query latency (Silver + JOIN) consistently exceeds SLA | > 500 ms p95 | API response time budget is shared with report_cache lookup + API overhead |
| `report_cache` hit rate drops below target | < 80% hit rate | More requests fall through to Silver + JOIN, amplifying latency impact |
| End-of-year / peak reporting causes query contention | Silver CPU > 70% during aggregate queries | Bulk reporting bursts overlap with score ingest, both competing for Silver reads |
| New tenant onboarded at scale comparable to TX | > 5M students | Tier 1 latency scales linearly with student count |

### Why Tier 2 Is Safe to Defer

Gold is **purely additive** to the existing pipeline:

- **No schema changes** to Bronze, Silver, or membership mirrors
- **No code changes** to the Score Processor, Sign-Pair Transformer, or PeerDB jobs
- **No migration** — Gold tables and MVs are new objects, not modifications of existing ones
- **Backfill from current state** — the one-time INSERT INTO Gold query reads Silver + membership FINAL at deployment time; no historical replay required
- **Rollback is DROP TABLE** — if Gold causes issues, drop the MVs and Gold tables. Silver + JOIN continues to work as before.

---

## 2. Motivation for Gold (Tier 2)

The current architecture computes school and district aggregates by querying Silver +
`membership_school_mirror FINAL` JOIN at query time. This scans every student row in Silver
for the given tenant/year/test and JOINs membership to attribute students to schools.

| Metric | Silver + JOIN (current) | Gold AggregatingMergeTree |
|--------|------------------------|--------------------------|
| Rows scanned (TX, 1 test) | ~5.5M student rows + membership | ~8,000 school rows |
| Query pattern | O(students) | O(schools) |
| Estimated speedup | — | ~700x fewer rows read |

Gold does **not** replace `report_cache` or the Aggregate Refresh Lambda. It provides a
faster on-the-fly aggregation path for school/district queries when Silver + JOIN latency
is insufficient. With Gold deployed, the API fallback chain becomes:
`report_cache` → Gold `sumMerge`/`sumMapMerge` → Silver + membership JOIN → Postgres fallback.

---

## 3. Why Streaming MVs Were Previously Rejected

TRS Design Document v6, Decision #6 identifies three correctness defects in a naive Gold
streaming MV:

1. **Membership JOIN baked at INSERT time** — student transfers never reflected
2. **`uniqState` double-counts on rescore** — HyperLogLog cannot subtract
3. **`avgState` denominator inflated on rescore pair** — internal `(sum, count)` state adds
   to both on each sign row

These defects apply to a **single MV** using `uniqState`/`avgState` that only fires on
Bronze inserts. They do not apply to the two-MV architecture described below.

---

## 4. Why This Design Is Different

### Two triggers, two MVs

School/district Gold aggregates can become stale from two distinct events:

| Event | Source table | MV |
|-------|-------------|-----|
| Score inserted / rescored / retested | `student_scores_bronze` | MV1 |
| Student joins or leaves a school | `membership_school_mirror` | MV2 |

Both MVs write to the **same** Gold `AggregatingMergeTree` table. Both use sign semantics
(±1) to add or subtract contributions. No DELETEs. Pure append-only.

### `sumState` replaces `uniqState` / `avgState`

- `sumState(sign)` provides exact student counts — rescore sign pairs net to +1 per active
  student, membership removal subtracts via -1. No HyperLogLog approximation.
- `sumMapState([perf_level], [sign])` provides exact per-performance-level counts —
  a rescore from level 3 to level 4 subtracts from level 3 and adds to level 4 via the
  sign pair.
- `avg` is derived at query time as `sumMerge(sum_scale_score) / sumMerge(n_tested)`.
  No inflated denominator.

---

## 5. Membership CDC Path — How Membership Reaches ClickHouse

### Two Separate PeerDB CDC Jobs

PeerDB runs **two independent CDC replication jobs** on Fargate, each with its own
replication slot:

| PeerDB Job | Aurora Source Table | ClickHouse Target | Engine | Passes through Sign-Pair Transformer? |
|---|---|---|---|---|
| Score CDC | `student_opportunities`, `student_component_scores` | Bronze tables | `VersionedCollapsingMergeTree` | **Yes** — Transformer emits `-1/+1` sign pairs |
| Membership CDC | `school_student`, `district_student` | `membership_school_mirror`, `membership_district_mirror` | `ReplacingMergeTree` | **No** — direct replication, no Transformer |

Membership tables do **not** need sign pairs. `ReplacingMergeTree(version, is_deleted)`
handles mutations natively — PeerDB inserts every CDC event as a new row; ClickHouse keeps
the highest-`version` row per ORDER BY key during background merge. `is_deleted = 1` is
the soft-delete sentinel.

### End-to-End Flow

```
RTS (external system)
    |
    v  Event or nightly full sync
Lambda: RTS Membership Sync
    |
    |-> Aurora: UPSERT school_student, district_student
    |          (also: roster_student, district_school, teacher_roster, student_attributes)
    |
    v  Aurora WAL (logical replication slot)
PeerDB Engine (Fargate)
    |  Reads WAL Before/After images
    |  NO Transformer -- direct INSERT to ClickHouse
    v
ClickHouse: membership_school_mirror   (ReplacingMergeTree)
ClickHouse: membership_district_mirror (ReplacingMergeTree)
    |
    +-> Gold MV2 fires on each INSERT (see section 8)
```

### Initial Snapshot

PeerDB's **Initial Load** handles the first-time population of membership mirrors:

1. PeerDB performs a parallel `SELECT` snapshot of `school_student` / `district_student`
   from Aurora
2. Bulk-inserts all rows into the ClickHouse `ReplacingMergeTree` mirrors
3. Automatically switches to **streaming mode** (WAL consumption) once the snapshot
   completes
4. Catches up on any WAL changes that occurred *during* the snapshot — no gap in CDC
   continuity

**Timing:** Membership tables are small relative to score tables. For TX scale (~5.5M
student-school pairs), the membership snapshot completes in minutes. Score table snapshot
is the long pole: 30–90 minutes.

**Ordering dependency:** Membership mirrors must be fully loaded *before* Gold MVs are
created. Otherwise MV1 (Bronze → Gold) fires on score inserts but the membership JOIN
returns no rows → scores are silently dropped from Gold. See [§16 Deployment Order](#16-deployment-order).

### `REPLICA IDENTITY FULL` Requirement

Both `school_student` and `district_student` must have `REPLICA IDENTITY FULL` set in
Aurora. Without it, PeerDB UPDATE/DELETE Before images contain only PK columns — all
other fields are null. For membership mirrors this means `is_deleted` could be null on a
DELETE event, making it impossible to distinguish joins from leaves.

```sql
ALTER TABLE trs.school_student    REPLICA IDENTITY FULL;
ALTER TABLE trs.district_student  REPLICA IDENTITY FULL;
```

### Membership Mirror Schema (for reference)

```sql
CREATE TABLE trs.membership_school_mirror (
    tenant_id    String,
    school_id    String,
    student_id   String,
    school_year  UInt16,
    is_deleted   UInt8       DEFAULT 0,
    version      UInt64      -- PeerDB-assigned monotonic version
)
ENGINE = ReplacingMergeTree(version, is_deleted)
ORDER BY (tenant_id, school_id, student_id);
```

`ReplacingMergeTree(version, is_deleted)`:
- During background merge, keeps only the row with the highest `version` per ORDER BY key.
- If that row has `is_deleted = 1`, the row is physically removed during merge.
- Before merge completes, multiple versions coexist. `FINAL` forces an on-the-fly merge
  at query time to resolve to the latest version.

---

## 6. Gold Table Schema

### `trs.school_aggregates_gold`

```sql
CREATE TABLE trs.school_aggregates_gold (
    tenant_id           String,
    school_year         UInt16,
    school_id           String,
    testalias_id        String,

    -- Partial aggregate states (sign-aware)
    n_tested            AggregateFunction(sum, Float64),
    sum_scale_score     AggregateFunction(sum, Float64),
    sum_raw_score       AggregateFunction(sum, Float64),
    perf_level_counts   AggregateFunction(sumMap, Array(Int16), Array(Float64))
)
ENGINE = AggregatingMergeTree()
PARTITION BY tenant_id
ORDER BY (tenant_id, school_year, testalias_id, school_id);
```

**Key properties:**
- `AggregatingMergeTree` — partial states are accumulated during background merges.
- ORDER BY matches the primary query pattern: tenant + year + test → all schools.
- All aggregate functions use `sum` / `sumMap` — both support negative values, enabling
  sign-based subtraction.
- No `uniqState`, `countState`, or `avgState` — see [§12](#12-why-sumstate-instead-of-uniqstate--countstate--avgstate).

---

## 7. MV1 — Bronze → Gold (Score Changes)

Fires on every INSERT into `student_scores_bronze`. The Bronze `sign` column (+1 or -1)
drives the aggregate direction — rescores and retests are handled by the sign pair.

```sql
CREATE MATERIALIZED VIEW trs.bronze_to_school_gold_mv
TO trs.school_aggregates_gold AS
SELECT
    b.tenant_id,
    b.school_year,
    m.school_id,
    b.testalias_id,

    sumState(toFloat64(b.sign))                                  AS n_tested,
    sumState(toFloat64(b.overall_scale_score) * toFloat64(b.sign)) AS sum_scale_score,
    sumState(toFloat64(b.overall_raw_score)   * toFloat64(b.sign)) AS sum_raw_score,
    sumMapState(
        [toInt16(b.overall_perf_level)],
        [toFloat64(b.sign)]
    )                                                            AS perf_level_counts

FROM trs.student_scores_bronze AS b
INNER JOIN trs.membership_school_mirror AS m
    ON b.tenant_id = m.tenant_id
    AND b.student_id = m.student_id
WHERE b.is_aggregate_eligible = 1
  AND b.use_for_aggregation   = 1
GROUP BY b.tenant_id, b.school_year, m.school_id, b.testalias_id;
```

**How sign semantics flow through:**

| Bronze event | `b.sign` | Effect on Gold |
|-------------|---------|----------------|
| New score (INSERT) | +1 | Adds +1 to `n_tested`, +score to `sum_scale_score`, +1 to perf level bucket |
| Rescore undo | -1 | Subtracts old score and old perf level from Gold |
| Rescore redo | +1 | Adds new score and new perf level to Gold |
| Retest deactivation (`use_for_aggregation` toggle) | -1 then +1 | Old opp: undo row passes filter (`use_for_aggregation=1` on Before image), redo row filtered out (`use_for_aggregation=0`). Net: old opp subtracted. New opp: separate INSERT adds it. |
| DELETE | -1 | Subtracts from Gold |

**Multi-enrollment:** If a student belongs to schools A and B, the JOIN produces two rows
per Bronze insert — one for each `school_id`. Both schools' Gold states are updated
independently. No double-counting within a single school.

---

## 8. MV2 — Membership Mirror → Gold (Membership Changes)

Fires on every INSERT into `membership_school_mirror` (PeerDB CDC from
`trs.school_student`). The `is_deleted` flag drives the sign — `0` means the student
joined, `1` means the student left.

```sql
CREATE MATERIALIZED VIEW trs.membership_to_school_gold_mv
TO trs.school_aggregates_gold AS
SELECT
    m.tenant_id,
    s.school_year,
    m.school_id,
    s.testalias_id,

    sumState(IF(m.is_deleted = 0, 1.0, -1.0))                           AS n_tested,
    sumState(
        argMaxMerge(s.latest_score) * IF(m.is_deleted = 0, 1.0, -1.0)
    )                                                                    AS sum_scale_score,
    sumState(
        toFloat64(argMaxMerge(s.latest_raw_score)) * IF(m.is_deleted = 0, 1.0, -1.0)
    )                                                                    AS sum_raw_score,
    sumMapState(
        [toInt16(argMaxMerge(s.latest_perf_level))],
        [IF(m.is_deleted = 0, 1.0, -1.0)]
    )                                                                    AS perf_level_counts

FROM trs.membership_school_mirror AS m
INNER JOIN trs.student_scores_silver AS s
    ON m.tenant_id = s.tenant_id
    AND m.student_id = s.student_id
GROUP BY m.tenant_id, s.school_year, m.school_id, s.testalias_id,
         s.student_id, s.opp_key
HAVING argMaxMerge(s.is_deleted)          = 0
   AND argMaxMerge(s.is_eligible)         = 1
   AND argMaxMerge(s.use_for_aggregation) = 1;
```

**How membership sign semantics flow through:**

| Membership event | `is_deleted` | Sign | Effect on Gold |
|-----------------|-------------|------|----------------|
| Student joins school | 0 | +1 | JOINs Silver FINAL → adds all eligible scores for this student to the school's Gold states |
| Student leaves school | 1 | -1 | JOINs Silver FINAL → subtracts all eligible scores for this student from the school's Gold states |

**Transfer (School A → School B):** PeerDB emits two CDC events for the Aurora transaction
that updates `school_student`:

1. `is_deleted=1` for old `(school_id=A, student_id=X)` → MV2 subtracts from School A
2. `is_deleted=0` for new `(school_id=B, student_id=X)` → MV2 adds to School B

No DELETE from Gold. Both events are appended to Gold as negative and positive partial
states respectively. `AggregatingMergeTree` merges them during background merge.

---

## 9. District Gold (Identical Pattern)

### `trs.district_aggregates_gold`

```sql
CREATE TABLE trs.district_aggregates_gold (
    tenant_id           String,
    school_year         UInt16,
    district_id         String,
    testalias_id        String,

    n_tested            AggregateFunction(sum, Float64),
    sum_scale_score     AggregateFunction(sum, Float64),
    sum_raw_score       AggregateFunction(sum, Float64),
    perf_level_counts   AggregateFunction(sumMap, Array(Int16), Array(Float64))
)
ENGINE = AggregatingMergeTree()
PARTITION BY tenant_id
ORDER BY (tenant_id, school_year, testalias_id, district_id);
```

### MV1 — Bronze → District Gold

```sql
CREATE MATERIALIZED VIEW trs.bronze_to_district_gold_mv
TO trs.district_aggregates_gold AS
SELECT
    b.tenant_id,
    b.school_year,
    m.district_id,
    b.testalias_id,

    sumState(toFloat64(b.sign))                                    AS n_tested,
    sumState(toFloat64(b.overall_scale_score) * toFloat64(b.sign)) AS sum_scale_score,
    sumState(toFloat64(b.overall_raw_score)   * toFloat64(b.sign)) AS sum_raw_score,
    sumMapState(
        [toInt16(b.overall_perf_level)],
        [toFloat64(b.sign)]
    )                                                              AS perf_level_counts

FROM trs.student_scores_bronze AS b
INNER JOIN trs.membership_district_mirror AS m
    ON b.tenant_id = m.tenant_id
    AND b.student_id = m.student_id
WHERE b.is_aggregate_eligible = 1
  AND b.use_for_aggregation   = 1
GROUP BY b.tenant_id, b.school_year, m.district_id, b.testalias_id;
```

### MV2 — Membership → District Gold

```sql
CREATE MATERIALIZED VIEW trs.membership_to_district_gold_mv
TO trs.district_aggregates_gold AS
SELECT
    m.tenant_id,
    s.school_year,
    m.district_id,
    s.testalias_id,

    sumState(IF(m.is_deleted = 0, 1.0, -1.0))                           AS n_tested,
    sumState(
        argMaxMerge(s.latest_score) * IF(m.is_deleted = 0, 1.0, -1.0)
    )                                                                    AS sum_scale_score,
    sumState(
        toFloat64(argMaxMerge(s.latest_raw_score)) * IF(m.is_deleted = 0, 1.0, -1.0)
    )                                                                    AS sum_raw_score,
    sumMapState(
        [toInt16(argMaxMerge(s.latest_perf_level))],
        [IF(m.is_deleted = 0, 1.0, -1.0)]
    )                                                                    AS perf_level_counts

FROM trs.membership_district_mirror AS m
INNER JOIN trs.student_scores_silver AS s
    ON m.tenant_id = s.tenant_id
    AND m.student_id = s.student_id
GROUP BY m.tenant_id, s.school_year, m.district_id, s.testalias_id,
         s.student_id, s.opp_key
HAVING argMaxMerge(s.is_deleted)          = 0
   AND argMaxMerge(s.is_eligible)         = 1
   AND argMaxMerge(s.use_for_aggregation) = 1;
```

---

## 10. Query-Time Read

### School aggregates (all schools for a tenant/year/test)

```sql
SELECT
    school_id,
    toUInt32(sumMerge(n_tested))                           AS students_tested,
    sumMerge(sum_scale_score) / sumMerge(n_tested)         AS avg_scale_score,
    sumMerge(sum_raw_score)   / sumMerge(n_tested)         AS avg_raw_score,
    sumMapMerge(perf_level_counts)                         AS perf_level_counts
    -- Returns: {1: 45, 2: 120, 3: 98, 4: 37}
FROM trs.school_aggregates_gold
WHERE tenant_id = {tenant_id:String}
  AND school_year = {school_year:UInt16}
  AND testalias_id = {testalias_id:String}
GROUP BY school_id;
```

### Single school

```sql
SELECT
    toUInt32(sumMerge(n_tested))                           AS students_tested,
    sumMerge(sum_scale_score) / sumMerge(n_tested)         AS avg_scale_score,
    sumMerge(sum_raw_score)   / sumMerge(n_tested)         AS avg_raw_score,
    sumMapMerge(perf_level_counts)                         AS perf_level_counts
FROM trs.school_aggregates_gold
WHERE tenant_id = {tenant_id:String}
  AND school_year = {school_year:UInt16}
  AND testalias_id = {testalias_id:String}
  AND school_id = {school_id:String};
```

### District aggregates

Same pattern, substituting `district_aggregates_gold` and `district_id`.

---

## 11. Correctness Matrix

| Event | `n_tested` | `sum_scale_score` | `perf_level_counts` |
|-------|-----------|------------------|---------------------|
| **New score** (student in 1 school) | +1 | +score | +1 at level X |
| **New score** (student in 2 schools) | +1 in each school | +score in each school | +1 at level X in each school |
| **Rescore** (level 3 → 4) | net 0 (−1+1) | net new score (−old+new) | −1 at level 3, +1 at level 4 |
| **Retest** (new opp replaces old) | net 0 (−1+1) | net new score | level adjusted |
| **Transfer** (School A → B) | −1 from A, +1 to B | −scores from A, +scores to B | −1 at level from A, +1 at level to B |
| **Multi-enrollment** (Schools A + B) | counted in both | scored in both | level counted in both |
| **DELETE** | −1 | −score | −1 at level X |

All metrics are exact — no HyperLogLog approximation, no denominator inflation.

---

## 12. Why `sumState` Instead of `uniqState` / `countState` / `avgState`

| Function | Subtraction support | Problem |
|----------|-------------------|---------|
| `uniqState` | No — HyperLogLog is add-only | Cannot un-count a student on transfer or rescore. Drifts permanently. |
| `countState` | No — always adds +1 | Cannot subtract. Rescore undo adds +1 instead of -1. |
| `avgState` | No — internal `(sum, count)` both increase | Rescore undo adds to both sum and count → inflated denominator. |
| **`sumState`** | **Yes — negative values merge correctly** | `sumState(-100)` + `sumState(100)` = 0 at merge time. |
| **`sumMapState`** | **Yes — per-key subtraction** | `sumMapState([3], [-1])` subtracts 1 from key 3. |

`avg_scale_score` is derived at query time: `sumMerge(sum_scale_score) / sumMerge(n_tested)`.
This avoids the `avgState` denominator problem entirely.

---

## 13. Timing & Edge Cases

### Score arrives before membership

MV1 fires on Bronze INSERT. The JOIN against `membership_school_mirror` finds no row for
the student → no Gold row emitted. When the membership CDC arrives later, MV2 fires and
JOINs Silver FINAL to pick up the student's existing scores → Gold correctly updated.

**Net result:** Correct, with a delay equal to membership CDC lag.

### Membership arrives before scores

MV2 fires on membership INSERT. The JOIN against `student_scores_silver` returns no rows
for the student → no Gold row emitted (student has no scores yet). When scores later arrive,
MV1 fires and JOINs `membership_school_mirror` → Gold correctly updated.

**Net result:** Correct, with no delay beyond normal score CDC latency.

### Concurrent score + membership changes

Both MVs fire independently. `AggregatingMergeTree` accumulates partial states from both.
No transactional coordination required — the sum of all sign contributions converges to the
correct value after both events are processed.

### End-of-year bulk transfers

A burst of membership CDC events fires MV2 for each affected student. Each MV2 invocation
JOINs Silver FINAL (reading current Silver state, not the real-time merged state). The
`FINAL` keyword forces ClickHouse to apply all pending merges for the queried parts —
this is more expensive per-row than a non-FINAL read but is bounded by the number of scores
for the transferred student.

**Concern:** If thousands of students transfer simultaneously, MV2 fires thousands of times,
each doing a Silver FINAL JOIN. This is CPU-intensive but bounded — each MV2 invocation
touches only one student's Silver rows. PeerDB batch size and ClickHouse's internal MV
batching mitigate thundering-herd effects.

### MV1 membership mirror read consistency

MV1 JOINs `membership_school_mirror` **without `FINAL`**. This is intentional:
- `FINAL` on every Bronze INSERT would be prohibitively expensive at score ingest rates.
- The `ReplacingMergeTree` mirror may contain unmerged duplicate rows for recently
  transferred students. In the worst case, a score is briefly attributed to both the old
  and new school until the next background merge.
- MV2 corrects this: the membership CDC event fires MV2 with the authoritative sign,
  producing the correct net contribution.

---

## 14. Pitfalls

### Pitfall 1: Initial Load — MV2 fires during membership snapshot

When PeerDB performs the initial snapshot of `school_student`, it bulk-inserts millions of
rows into `membership_school_mirror`. Each inserted row fires MV2, which JOINs Silver.

**Problem:** If the score initial load has not completed yet (or hasn't started), Silver is
empty. MV2 fires millions of times, each JOIN returns zero rows, and Gold gets nothing.
When scores later arrive, MV1 fires and picks up the membership JOIN — so Gold eventually
converges. But the burst of MV2 fires against an empty Silver table is wasted CPU.

**Mitigation:** Create Gold MVs **after** both the score and membership initial loads are
complete. Then run the one-time backfill query (see [§17](#17-deployment-order)). Streaming
MVs handle all subsequent CDC events.

### Pitfall 2: MV2 double-counting on membership mirror re-INSERT

`ReplacingMergeTree` deduplicates rows during background merge, not at INSERT time. If
PeerDB re-delivers a membership CDC event (e.g., connector restart with replay), the mirror
temporarily has two rows for the same `(tenant_id, school_id, student_id)`. MV2 fires on
the re-delivered INSERT and adds *another* +1 contribution to Gold.

**Impact:** Student is briefly double-counted in Gold for that school until:
1. `ReplacingMergeTree` background merge deduplicates the membership mirror rows, AND
2. A subsequent score or membership event triggers a corrective Gold contribution.

**Mitigation:** This is self-healing — the next score INSERT (MV1) or membership change
(MV2) for the affected student produces the correct net contribution. For critical
reporting, fall back to Tier 1 (`report_cache`) which is computed from Silver + FINAL JOIN
and is always consistent.

### Pitfall 3: `REPLICA IDENTITY FULL` not set on Aurora source tables

If `school_student` or `district_student` tables lack `REPLICA IDENTITY FULL`, PeerDB
UPDATE/DELETE Before images contain only primary key columns. All non-PK columns (including
`is_deleted`, `school_year`) arrive as null in the ClickHouse mirror.

**Impact:** MV2 cannot distinguish a student joining (`is_deleted=0`) from a student leaving
(`is_deleted=1`) — the sign is always derived from `is_deleted`, which would be 0 (default)
for all events. Transfers would add to the new school but never subtract from the old.
Gold drifts permanently.

**Mitigation:** Set `REPLICA IDENTITY FULL` before enabling PeerDB replication:
```sql
ALTER TABLE trs.school_student    REPLICA IDENTITY FULL;
ALTER TABLE trs.district_student  REPLICA IDENTITY FULL;
```
This is a prerequisite, not an optional optimization. Verify with:
```sql
SELECT relname, relreplident FROM pg_class
WHERE relname IN ('school_student', 'district_student');
-- Expected: relreplident = 'f' (FULL)
```

### Pitfall 4: PeerDB batch delivery and MV atomicity

PeerDB delivers CDC events in batches. ClickHouse processes each batch as a single INSERT
block. All MVs attached to the target table fire once per block, not once per row.

**Impact on MV1:** A batch of 1,000 Bronze rows triggers MV1 once. The MV's SELECT operates
on all 1,000 rows, JOINing each against the membership mirror. The GROUP BY aggregates
across all rows in the batch — this is correct and efficient.

**Impact on MV2:** A batch of membership rows triggers MV2 once. The MV's SELECT JOINs all
membership rows in the batch against Silver. If the batch contains both the `is_deleted=1`
(leave School A) and `is_deleted=0` (join School B) events for the same student transfer,
both are processed in the same MV invocation — the subtraction from A and addition to B
happen atomically within the batch. This is the best-case scenario for transfer consistency.

### Pitfall 5: Gold state drift over time

Floating-point arithmetic in `sumState` is not perfectly associative. Over millions of
sign-pair additions and subtractions, rounding errors may accumulate — e.g., `n_tested`
might drift to `299.9999999997` instead of `300`.

**Impact:** Negligible for `n_tested` (cast to `toUInt32` at query time rounds correctly)
and for averages (rounding error in sum/count is sub-penny). Performance level counts are
integer-valued and cast at query time.

**Mitigation:** Periodic full recompute of Gold from Silver + membership FINAL JOIN
(the same backfill query used during initial load) can be run as a maintenance task to
reset accumulated drift. Frequency: monthly or after bulk operations.

### Pitfall 6: Multi-enrollment and MV2 scope

When a student is enrolled in Schools A and B, and their membership in School A changes
(`is_deleted=1`), MV2 fires and subtracts the student's scores from School A. It does
**not** touch School B's Gold state — correctly, since the student is still enrolled there.

**However:** If the student is simultaneously removed from School A and added to School C
in the same Aurora transaction, PeerDB emits CDC events for both `school_student` rows.
MV2 correctly handles this: subtract from A, add to C, leave B unchanged.

**Edge case:** If the student has no scores in Silver yet, both MV2 invocations JOIN Silver
and find nothing — Gold is unchanged. When scores later arrive, MV1 fires and JOINs the
membership mirror, adding scores to Schools B and C (the current memberships). School A
gets nothing since the student is no longer enrolled there. Correct.

---

## 15. Performance Impact

### Query performance (Gold read)

| Query | Rows scanned | Estimated latency |
|-------|-------------|-------------------|
| All schools for 1 test (TX) | ~8,000 Gold rows | < 5 ms |
| Single school for 1 test | 1 Gold row | < 1 ms |
| All districts for 1 test (TX) | ~1,200 Gold rows | < 2 ms |

Compare to current Silver + JOIN: ~5.5M student rows → 200–800 ms.

### Write overhead

| Event | Current (Bronze → Silver) | With Gold (Bronze → Silver + Gold) |
|-------|--------------------------|-----------------------------------|
| Score INSERT | 1 Silver MV fire | 1 Silver MV + 1 Gold MV (MV1) |
| Score UPDATE (rescore) | 2 Silver MV fires (sign pair) | 2 Silver MV + 2 Gold MV fires |
| Membership change | None | 1 Gold MV fire (MV2) with Silver FINAL JOIN |

The Gold MV fires add ~1 JOIN lookup per Bronze INSERT (MV1 reads membership mirror) and
~1 Silver FINAL read per membership change (MV2). Both are lightweight for single-student
operations.

### Storage

Gold rows are small (one row per school × test). For TX scale: ~8,000 schools × ~50 tests
= ~400K rows per year. Negligible compared to Bronze/Silver.

---

## 16. Medallion Layer Summary (Updated)

| Feature | Bronze | Silver (Overall) | Silver (Component) | **Gold (School/District)** |
|---|---|---|---|---|
| **Engine** | `VersionedCollapsingMergeTree` | `AggregatingMergeTree` | `AggregatingMergeTree` | **`AggregatingMergeTree`** |
| **Granularity** | 1 row per CDC event | 1 row per student opportunity | 1 row per student × opp × component | **1 row per school/district × test** |
| **Primary Question** | "What changed and when?" | "Latest score per student?" | "Latest component score?" | **"What are the aggregate stats for this school/district?"** |
| **Populated By** | Sign-Pair Transformer | Streaming MV from Bronze | Streaming MV from Bronze | **Two streaming MVs: MV1 from Bronze, MV2 from membership mirror** |
| **Rescore Handling** | Signs cancel on merge | `argMax` keeps latest | `argMax` keeps latest | **Sign pair: −old + new via `sumState`** |
| **Retest Handling** | Both opps in Bronze | `HAVING use_for_aggregation = 1` | Inherited via opp_key JOIN | **Sign pair on `use_for_aggregation` toggle** |
| **Membership Attribution** | N/A | Query-time JOIN | Via overall Silver JOIN | **MV1: JOIN at score time. MV2: corrects on membership change** |
| **Transfer Handling** | N/A | Always current via FINAL | Via overall Silver | **MV2 subtracts from old school, adds to new school** |

---

## 17. Deployment Order

Add after step 6 (Silver MV) in the existing deployment order:

```
 7. trs.membership_school_mirror     (already exists — required before Gold MVs)
 8. trs.membership_district_mirror   (already exists — required before Gold MVs)
 9. Run PeerDB initial load for membership tables (wait for completion)
10. Run PeerDB initial load for score tables (wait for completion — long pole)
11. trs.school_aggregates_gold       (Gold table — must exist before MVs)
12. trs.district_aggregates_gold     (Gold table — must exist before MVs)
13. One-time backfill: INSERT INTO school_aggregates_gold SELECT ... FROM Silver JOIN membership FINAL
14. One-time backfill: INSERT INTO district_aggregates_gold SELECT ... FROM Silver JOIN membership FINAL
15. trs.bronze_to_school_gold_mv     (MV1 — score changes → school Gold)
16. trs.membership_to_school_gold_mv (MV2 — membership changes → school Gold)
17. trs.bronze_to_district_gold_mv   (MV1 — score changes → district Gold)
18. trs.membership_to_district_gold_mv (MV2 — membership changes → district Gold)
```

**Critical:** Steps 13–14 (backfill) must happen *before* steps 15–18 (MV creation).
If MVs are created before backfill, any CDC events arriving during the backfill INSERT
will be double-counted — once by the backfill, once by the MV. Creating MVs after backfill
ensures the MVs only process events that arrive *after* the backfill snapshot.

**Initial Load Backfill Query (School):**

```sql
INSERT INTO trs.school_aggregates_gold
SELECT
    s.tenant_id,
    s.school_year,
    m.school_id,
    s.testalias_id,

    sumState(toFloat64(1))                                          AS n_tested,
    sumState(toFloat64(argMaxMerge(s.latest_score)))                AS sum_scale_score,
    sumState(toFloat64(argMaxMerge(s.latest_raw_score)))            AS sum_raw_score,
    sumMapState(
        [toInt16(argMaxMerge(s.latest_perf_level))],
        [toFloat64(1)]
    )                                                               AS perf_level_counts

FROM trs.student_scores_silver AS s FINAL
INNER JOIN trs.membership_school_mirror FINAL AS m
    ON s.tenant_id = m.tenant_id
    AND s.student_id = m.student_id
WHERE argMaxMerge(s.is_deleted)          = 0
  AND argMaxMerge(s.is_eligible)         = 1
  AND argMaxMerge(s.use_for_aggregation) = 1
GROUP BY s.tenant_id, s.school_year, m.school_id, s.testalias_id;
```
