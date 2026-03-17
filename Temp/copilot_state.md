# TRS Design — Session Handoff: Source of Truth

**Date:** 2026-03-03  
**Status:** Design complete; v4 document NOT yet created.  
**Immediate next task:** Create `TRS_Design_Document_v4.md` (copy of v3 with all changes below applied).

---

## Root Problem Fixed in v4

`LAYOUT(HASHED())` ClickHouse dictionaries store **exactly one value per primary key**.
`roster_dict PRIMARY KEY student_id` silently drops the second school for multi-enrolled students.
Result: silent under-counting + phantom Gold rows under `school_id = ''`.

**Fix pattern:** Replace all `dictGet` calls in Gold MVs with:
```sql
INNER JOIN (
    SELECT DISTINCT tenant_id, <scope>_id, student_id
    FROM trs.membership_<scope>_mirror FINAL
    WHERE is_deleted = 0
) AS m ON s.tenant_id = m.tenant_id AND s.student_id = m.student_id
```
`DISTINCT` closes the race window before `ReplacingMergeTree` background merges complete.
`FINAL` forces merge-read at MV insert time for correctness.

---

## New Aurora (RTS) Normalized Tables — Replace `school_roster_members`

RTS now provides 5 separate relationship tables. PeerDB replicates directly from them.

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

---

## New ClickHouse Membership Mirror Tables — Replace `student_scope_today`

PeerDB replicates `school_student` → `membership_school_mirror` and `district_student` → `membership_district_mirror`.  
`version` = `_peerdb_version` (Postgres LSN). Higher LSN wins. `is_deleted = 1` handles deletes.

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

## Replaced / Retired Artifacts

| Artifact | Disposition |
|---|---|
| `school_roster_members` (Aurora) | **DROP** — replaced by `school_student` |
| `v_student_school_current` (Aurora view) | **DROP** — no consumers remain |
| `trs.roster_dict` (ClickHouse dict) | **Retire from Gold MV critical path** — diagnostic utility only |
| `trs.school_district_dict` (ClickHouse dict) | **Retire from Gold MV critical path** — diagnostic utility only |
| `trs.student_scope_today` (ClickHouse table) | **Demote to diagnostic** — not used in any MV |

---

## v4 Edit Checklist (all sections of `TRS_Design_Document_v3.md`)

### Header / §1
- [ ] Change title to `v4`; update date to `2026-03-03`
- [ ] Add "Key changes from v3" bullet: **Multi-enrollment fix** — replaced dictGet Gold lookups with membership mirror JOIN FINAL; introduced 5 normalized RTS relationship tables
- [ ] §1.2 table: add row `Multi-enrollment attribution | dictGet 1:1 lookup | Membership mirror JOIN FINAL`

### §2.1 Architecture Diagram
- [ ] Replace `student_scope_today` line in ClickHouse box with `membership_school_mirror / membership_district_mirror`
- [ ] Remove `roster_dict` / `school_district_dict` from Dictionaries line in diagram (or mark as diagnostic)

### §6.3 Membership & Config Tables
- [ ] Remove `school_roster_members` row from table
- [ ] Add 5 new RTS normalized tables with full DDL (see above): `district_student`, `school_student`, `roster_student`, `district_school`, `teacher_roster`

### §6.4 — REMOVE ENTIRE SECTION
- [ ] Delete `v_student_school_current` view section — no consumers remain
- [ ] Renumber: old §6.5 becomes §6.4, old §6.6 becomes §6.5, old §6.7 becomes §6.6

### §6.7 (now §6.6) Postgres Materialized Views
- [ ] `mv_school_overall`: change `JOIN trs.school_roster_members smt` → `JOIN trs.school_student ss_m ON ss.tenant_id = ss_m.tenant_id AND ss.student_id = ss_m.student_id WHERE ss_m.active = TRUE`; alias `ss_m.school_id`
- [ ] `mv_district_overall`: remove two-table join chain (`school_roster_members` + `schools`); replace with `JOIN trs.district_student ds ON ss.tenant_id = ds.tenant_id AND ss.student_id = ds.student_id WHERE ds.active = TRUE`; `GROUP BY … ds.district_id`

### §7.3 Gold — School (rename MV + replace dictGet)
- [ ] Rename `trs.school_agg_mv` → `trs.bronze_to_school_gold_mv`
- [ ] Remove `dictGet('trs.roster_dict', 'school_id', student_id) AS school_id`
- [ ] Replace Bronze source with JOIN:
```sql
CREATE MATERIALIZED VIEW trs.bronze_to_school_gold_mv
TO trs.school_aggregates_gold AS
SELECT
    s.tenant_id, s.school_year, m.school_id, s.test_group_id,
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

### §7.4 Gold — District (rename MV + replace nested dictGet)
- [ ] Rename `trs.district_agg_mv` → `trs.bronze_to_district_gold_mv`
- [ ] Remove two-hop `dictGet(school_district_dict, dictGet(roster_dict, …))`
- [ ] Replace with JOIN; retain all 4 PL count `sumState` columns:
```sql
CREATE MATERIALIZED VIEW trs.bronze_to_district_gold_mv
TO trs.district_aggregates_gold AS
SELECT
    s.tenant_id, s.school_year, m.district_id, s.test_group_id,
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

### §7.5 Dictionaries
- [ ] Mark both `roster_dict` and `school_district_dict` as **"Retired from Gold MV critical path — diagnostic utility only"**
- [ ] Remove them from deployment order (steps 10, 11 in §16.5)

### §7.6 Membership Mirror Tables
- [ ] Replace `student_scope_today` DDL with `membership_school_mirror` + `membership_district_mirror` full DDL (see above)
- [ ] Add `student_scope_today` in new sub-section "Diagnostic Tables" with note: "not used in any MV; retained for ad-hoc investigations only"
- [ ] Retain `student_attributes` unchanged

### §7.7 Medallion Layer Summary Table
- [ ] Update "Dictionary Lookup" row: Gold-School = `membership_school_mirror FINAL (JOIN)`; Gold-District = `membership_district_mirror FINAL (JOIN)`

### §8.3 Membership Sync Path
- [ ] Change Aurora UPSERT targets from `school_roster_members` to `district_student`, `school_student`, `roster_student`
- [ ] Remove note: "ClickHouse roster_dict and school_district_dict auto-refresh from Aurora on their configured LIFETIME schedule"
- [ ] Add note: "PeerDB mirrors `school_student` → `membership_school_mirror` and `district_student` → `membership_district_mirror` via CDC replication jobs"

### §15 Key Engineering Decisions
- [ ] Remove decision #11 (dictionaries backed by Aurora views) — retired
- [ ] Add new decision: **Membership mirror JOIN FINAL replaces dictGet lookups** — `HASHED()` layout stores 1 value per primary key, silently dropping multi-enrollment; `ReplacingMergeTree` mirrors natively support N rows per student; `FINAL` + `DISTINCT` subquery provides correct multi-enrollment attribution at MV write time

### §16 Operational Prerequisites
- [ ] §16.1: Add `ALTER TABLE trs.school_student REPLICA IDENTITY FULL;` and `ALTER TABLE trs.district_student REPLICA IDENTITY FULL;`
- [ ] §16.4: Remove `v_student_school_current` Aurora view requirement (section can be dropped or replaced with note about 5 new RTS tables requiring no Aurora views)
- [ ] §16.5 Deployment Order: replace steps 8–11 with:
  - Step 8: `trs.membership_school_mirror`
  - Step 9: `trs.membership_district_mirror`
  - Step 10: `trs.student_attributes`
  - Step 11: `trs.student_scope_today` (diagnostic only; deploy last, no dependents)
  - Remove steps for `trs.roster_dict` and `trs.school_district_dict` from critical path

---

## Key Invariants (do NOT change these)

- Column name is `overall_scale_score` everywhere — NOT `score_value`
- `pl_1_count` through `pl_4_count` in `district_aggregates_gold` table AND MV — retained
- All Bronze / Silver schema — **unchanged**
- `student_attributes` table — **unchanged**
- Three-tier API fallback logic — **unchanged**
- Postgres `report_cache` — **unchanged**
- `trs.score_ingest_log` — **unchanged**

---

## Roster Query Pattern (no Gold MV needed)

Roster queries are always live — no ClickHouse Gold cube for roster scope. Student IDs are resolved from Aurora `roster_student` directly:

```sql
-- Step 1: resolve student IDs from Aurora
SELECT student_id FROM trs.roster_student
WHERE tenant_id = $1 AND roster_id = $2 AND active = TRUE;

-- Step 2: query Silver FINAL with student ID list
SELECT student_id, argMaxMerge(latest_score) AS score, ...
FROM trs.student_scores_silver
WHERE tenant_id = ? AND school_year = ? AND test_group_id = ?
  AND student_id IN (?)
GROUP BY student_id, opp_key
HAVING argMaxMerge(is_deleted) = 0;
```

---

## Files

| File | Action |
|---|---|
| `TRS_Design_Document_v3.md` | Source — read-only reference |
| `TRS_Design_Document_v4.md` | **CREATE THIS** — copy of v3 with all checklist items above applied |
| `implementation/RTS_Sync_Lambda.md` | Needs updating: change `school_roster_members` references to 5 normalized tables |
