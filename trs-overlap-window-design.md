# TRS Design Amendment: Overlap Window — Dual-Year Queryability During Rollover

**Status**: Draft
**Date**: 2026-03-22
**Amends**: TRS One-Year Rollover Design

---

## Overview

This amendment revises the immediate-wipe rollover model. Instead of dropping the old year's data at rollover, both years coexist for a configurable overlap period (1-2 months). During overlap, users can pick the previous year and view older scores. The old year is read-only — no new scores are accepted for it. After the overlap window expires, a cleanup job drops the old year partition.

This model is simpler to build than the immediate-wipe approach because the rollover itself is just a partition create (seconds), and the heavy cleanup happens later when nobody cares about the old data.

---

## Rollover Lifecycle

```
ACTIVE (year N only)
  ──► ROLLING_OVER (create year N+1 partition, brief ~seconds)
  ──► OVERLAP (both N and N+1 live, 1-2 months)
  ──► CLEANUP_PENDING (overlap expired, old year about to be dropped)
  ──► ACTIVE (year N+1 only)
```

### Concrete Timeline Example (Texas, July 1 rollover)

```
June 30 (day before rollover):
  └─ Status: ACTIVE, year=2026, prior_year=NULL

July 1 (rollover):
  └─ Create 2027 partition (~seconds)
  └─ Status: OVERLAP, year=2027, prior_year=2026
  └─ New scores → 2027, old scores → read-only
  └─ UI shows year picker: [2026] [2027]

July 1 – Aug 31 (overlap period):
  └─ Both years queryable
  └─ Prior year cached aggressively (frozen data)
  └─ No user-facing downtime at any point

Sept 1 (cleanup):
  └─ DROP 2026 partitions (Postgres + ClickHouse)
  └─ Flush 2026 cache entries
  └─ Status: ACTIVE, year=2027, prior_year=NULL
  └─ UI shows only [2027]
```

---

## 1. Tenant Configuration

### Schema Changes

```sql
ALTER TABLE configs.tenants ADD COLUMN
    fiscal_year_start_month   SMALLINT NOT NULL DEFAULT 7,
    fiscal_year_start_day     SMALLINT NOT NULL DEFAULT 1,
    overlap_days              SMALLINT NOT NULL DEFAULT 60,
    current_school_year       SMALLINT NOT NULL,
    prior_school_year         SMALLINT,              -- NULL when no overlap active
    overlap_expires_at        DATE,                  -- NULL when no overlap active
    rollover_status           TEXT     NOT NULL DEFAULT 'active';
```

### Rollover Status Values

| Status | Meaning |
|--------|---------|
| `active` | Single year, normal operation |
| `rolling_over` | Brief pause (~seconds) while new partition is created |
| `overlap` | Both years readable; new scores go to new year only |
| `cleanup_pending` | Overlap expired, old year about to be dropped |

---

## 2. Rollover Orchestrator — Two-Phase

### Phase 1: Rollover (brief, ~seconds)

Triggered by EventBridge cron when the tenant's rollover date is reached.

```
1. Set rollover_status = 'rolling_over'
2. Create new year partition (N+1) in Postgres
   - scores.student_opportunities
   - scores.student_component_scores
   - scores.score_ingest_log
3. Set current_school_year = N+1
4. Set prior_school_year = N
5. Set overlap_expires_at = rollover_date + overlap_days
6. Set rollover_status = 'overlap'
```

No need to:
- Drain the CDC pipeline (old data stays in place)
- Flush the report cache (old year entries remain valid)
- Drop any ClickHouse partitions (both years coexist)
- Pause ingestion for more than a few seconds

### Phase 2: Cleanup (runs after `overlap_days` expires)

Triggered by EventBridge cron that checks daily for tenants where `overlap_expires_at <= today`. **All steps are instant metadata/DDL operations — a single Lambda (256 MB, 30s timeout) handles this in under 2 seconds.**

```
1. Set rollover_status = 'cleanup_pending'
2. DROP old year partition in Postgres (year N)
   - DROP TABLE scores.student_opportunities_{tenant}_{year_n}
   - DROP TABLE scores.student_component_scores_{tenant}_{year_n}
   - DROP TABLE scores.score_ingest_log_{tenant}_{year_n}
3. DROP old year partition in ClickHouse (HTTP calls)
   - ALTER TABLE trs.student_scores_bronze DROP PARTITION ('{tenant}', {year_n})
   - ALTER TABLE trs.student_scores_silver DROP PARTITION ('{tenant}', {year_n})
   - ALTER TABLE trs.student_component_scores_bronze DROP PARTITION ('{tenant}', {year_n})
   - ALTER TABLE trs.student_component_scores_silver DROP PARTITION ('{tenant}', {year_n})
4. DELETE report_cache entries for year N + this tenant
5. Set prior_school_year = NULL
6. Set overlap_expires_at = NULL
7. Set rollover_status = 'active'
```

All partition drops are O(1) — Postgres and ClickHouse both drop underlying files instantly.

**Why no MV refresh step:** The original draft included `REFRESH MATERIALIZED VIEW CONCURRENTLY` here, but this is unnecessary and would be the only slow operation (~30-60s at Texas scale, re-reading 55M+ rows). The existing Aggregate Refresh Lambda already refreshes MVs on a 15-30 minute EventBridge schedule. The next scheduled refresh after the partition drop will naturally exclude old year rows because the source data is gone. No special action needed — the normal cycle handles it.

---

## 3. Score Processor Changes

Only new-year scores are accepted. No ingestion pause needed during overlap.

```csharp
var tenant = await GetTenantConfig(tenantId);

// Brief rollover window (~seconds) — retry later
if (tenant.RolloverStatus == "rolling_over")
{
    throw new RolloverInProgressException(); // SQS visibility timeout retry
}

// Only accept scores for current year
if (score.SchoolYear != tenant.CurrentSchoolYear)
{
    await LogRejection("WRONG_SCHOOL_YEAR", ...);
    return;
}
```

---

## 4. API Changes

### Year Parameter — Kept, With Validation

The API accepts an optional `school_year` parameter. It defaults to the current year and validates against the set of queryable years.

```csharp
var tenant = await GetTenantConfig(tenantId);

// Default to current year if not specified
var year = request.SchoolYear ?? tenant.CurrentSchoolYear;

// Build set of allowed years
var allowedYears = new HashSet<int> { tenant.CurrentSchoolYear };
if (tenant.PriorSchoolYear.HasValue)
    allowedYears.Add(tenant.PriorSchoolYear.Value);

if (!allowedYears.Contains(year))
    return StatusCode(404, "Data for this school year is not available.");
```

### Available Years Endpoint

The UI needs to know whether to show a year picker:

```
GET /api/v1/tenant/{tenant_id}/available-years
```

```json
{
  "current_year": 2027,
  "prior_year": 2026,
  "overlap_expires": "2027-09-01"
}
```

When `prior_year` is `null`, the UI shows no year picker. When non-null, the UI renders a selector.

---

## 5. Cache Strategy During Overlap

Prior year data is frozen — no new scores, no membership changes affect it. This means we can cache it very aggressively.

### Cache TTLs by Year

| Scope | Current Year TTL | Prior Year TTL (overlap) |
|-------|-----------------|--------------------------|
| Roster | No cache (live) | 1 hour |
| School | 15 min | 24 hours |
| District | 5 min | 24 hours |
| State | 2-4 hours | 24 hours |

### Prior Year Cache Prewarming

At rollover, the Aggregate Refresh Lambda can precompute and cache prior year aggregates with long TTLs:

```sql
INSERT INTO scores.report_cache (cache_key, payload, computed_at, expires_at, tenant_id)
VALUES (
    'state#tx#all#2026#cp1-g5-ela',
    '{"students_tested": 5543751, ...}',
    now(),
    '2027-09-01',   -- lives until overlap expires
    'tx'
);
```

Since prior year data is immutable during overlap, these cached values are guaranteed correct for the entire overlap period. One compute, zero refreshes.

---

## 6. Report Cache — Tenant-Scoped Operations

### Schema Addition

```sql
ALTER TABLE scores.report_cache ADD COLUMN tenant_id TEXT;
CREATE INDEX idx_report_cache_tenant ON scores.report_cache(tenant_id);
```

### Cleanup Phase Flush

At cleanup, only prior year entries are deleted:

```sql
DELETE FROM scores.report_cache
WHERE tenant_id = 'tx'
  AND cache_key LIKE '%#2026#%';
```

Current year cache entries are untouched.

---

## 7. ClickHouse — Full CDC Lifecycle During Rollover

This section traces exactly what ClickHouse sees at every phase of the rollover lifecycle, from normal operation through cleanup. Understanding this is critical because **ClickHouse never receives an explicit "delete year N" signal through CDC** — Postgres partition drops are DDL operations that do not emit row-level WAL events.

### 7.1 Recap: How Scores Reach ClickHouse (Normal Operation)

```
Score JSON → SQS → Score Processor Lambda
  → UPSERT into Postgres scores.student_opportunities (year 2026 partition)
  → Postgres WAL captures the INSERT or UPDATE row event
  → PeerDB reads the WAL replication slot
  → PeerDB emits before/after row images to Sign-Pair Transformer
  → Transformer emits -1/+1 sign pairs → HTTP batch INSERT to ClickHouse Bronze
  → Bronze VersionedCollapsingMergeTree stores the sign-pair rows
  → Silver Materialized View fires on Bronze INSERT
  → Silver AggregatingMergeTree accumulates argMaxState per (student, opp_key)
```

Key details:
- PeerDB operates on **logical replication** — it sees row-level changes on the logical table `scores.student_opportunities`, not on individual partitions
- PeerDB does not know or care about Postgres partitioning — a row inserted into `student_opportunities_tx_2026` appears in the WAL as an insert to `scores.student_opportunities`
- The `school_year` column is just another data column that flows through CDC
- ClickHouse routes rows to partitions automatically based on `PARTITION BY (tenant_id, school_year)` — a row with `school_year=2026` lands in the `('tx', 2026)` partition, a row with `school_year=2027` lands in `('tx', 2027)`

### 7.2 Phase 1: Rollover (Create Year N+1 Partition in Postgres)

**What happens in Postgres:**
```sql
CREATE TABLE scores.student_opportunities_tx_2027
  PARTITION OF scores.student_opportunities_tx
  FOR VALUES IN (2027);
```

**What PeerDB sees:** Nothing. `CREATE TABLE` is DDL. The logical replication slot emits no row-level events for DDL. PeerDB's replication slot cursor (LSN) advances past this WAL entry silently.

**What the Sign-Pair Transformer sees:** Nothing. No events to process.

**What ClickHouse sees:** Nothing. No inserts arrive.

**What changes for subsequent scores:** When a new score arrives with `school_year=2027`, the Score Processor UPSERTs it into Postgres. The WAL emits a normal INSERT event. PeerDB picks it up. The Transformer emits a `+1` sign-pair. ClickHouse receives the insert, and because the row contains `school_year=2027`, ClickHouse's `PARTITION BY (tenant_id, school_year)` expression automatically creates the `('tx', 2027)` partition directory on disk and writes the part there.

**ClickHouse does not need its partitions pre-created.** Unlike Postgres (where you must `CREATE TABLE ... PARTITION OF` before inserting), ClickHouse creates partition directories on the fly based on the partition expression. The first row with `school_year=2027` creates the partition.

### 7.3 Overlap Period (Both Years Coexist)

**CDC flow:**
- New scores (2027) flow through the pipeline normally: Postgres → WAL → PeerDB → Transformer → Bronze → Silver
- Old year (2026) data sits in ClickHouse partitions `('tx', 2026)` across Bronze and Silver — untouched, no CDC events reference it
- No rescores or retests arrive for 2026 because the Score Processor rejects `school_year != current_school_year`

**ClickHouse partition state during overlap:**

```
trs.student_scores_bronze
  └─ ('tx', 2026)   ← static, no new parts written, fully merged
  └─ ('tx', 2027)   ← actively receiving CDC inserts

trs.student_scores_silver
  └─ ('tx', 2026)   ← static, argMaxState fully resolved after final merge
  └─ ('tx', 2027)   ← actively accumulating via MV

trs.student_component_scores_bronze
  └─ ('tx', 2026)   ← static
  └─ ('tx', 2027)   ← active

trs.student_component_scores_silver
  └─ ('tx', 2026)   ← static
  └─ ('tx', 2027)   ← active
```

**Query behavior during overlap:**
- `WHERE tenant_id = 'tx' AND school_year = 2026` → reads only the 2026 partition. Data is fully merged and static, so queries are fast (no unmerged parts to resolve at read time).
- `WHERE tenant_id = 'tx' AND school_year = 2027` → reads only the 2027 partition. May have unmerged parts (normal for active ingestion).

**Merge behavior:** ClickHouse's background merge process will eventually fully merge the 2026 partition's parts since no new data arrives. This means 2026 queries become progressively cheaper over time — `argMaxMerge` has less work to do when parts are already merged. This is a nice side effect of the frozen-data model.

### 7.4 Phase 2: Cleanup — The Critical Postgres-ClickHouse Disconnect

This is where the important subtlety lives.

**What happens in Postgres:**
```sql
DROP TABLE scores.student_opportunities_tx_2026;
```

**What PeerDB sees:** Nothing useful. `DROP TABLE` on a partition is DDL. Logical replication does not emit row-level DELETE events for a partition drop. The WAL contains the DDL command, but PeerDB's logical decoding plugin (pgoutput) does not surface DDL to consumers. PeerDB's LSN advances past it.

**What the Sign-Pair Transformer sees:** Nothing. Zero events.

**What ClickHouse sees:** Nothing. The 2026 data in Bronze and Silver remains exactly as it was. ClickHouse has no idea the Postgres partition was dropped.

**This means: ClickHouse will serve stale 2026 data forever unless we explicitly drop the ClickHouse partitions.**

This is not a CDC problem — it's an architectural reality. CDC replicates row-level DML (INSERT, UPDATE, DELETE). Partition drops are DDL. No row-level events are emitted. There is no way for PeerDB or any logical replication-based CDC tool to propagate a partition drop as row deletions.

### 7.5 ClickHouse Cleanup: Explicit Partition Drop

The Rollover Orchestrator Phase 2 must explicitly drop ClickHouse partitions as a separate step, **after** the Postgres partition drop:

```sql
-- 1. Drop Bronze partitions (removes raw CDC sign-pair history)
ALTER TABLE trs.student_scores_bronze
  DROP PARTITION ('tx', 2026);

ALTER TABLE trs.student_component_scores_bronze
  DROP PARTITION ('tx', 2026);

-- 2. Drop Silver partitions (removes aggregated student-level state)
ALTER TABLE trs.student_scores_silver
  DROP PARTITION ('tx', 2026);

ALTER TABLE trs.student_component_scores_silver
  DROP PARTITION ('tx', 2026);
```

**What `DROP PARTITION` does in ClickHouse:**
- Removes the partition's data directories from disk immediately
- No row-by-row deletion — the entire partition directory is detached and deleted
- This is an O(1) metadata operation regardless of how many rows the partition contains
- The operation is atomic per table — either the partition is fully dropped or not at all
- Other partitions (e.g., `('tx', 2027)`) are completely unaffected
- Active inserts to the 2027 partition continue uninterrupted during the drop

**What about in-flight CDC events?** By the time we reach Phase 2 cleanup (60+ days after rollover), there are zero CDC events in flight for 2026. The Score Processor has been rejecting 2026 scores since rollover. The CDC pipeline has been idle for 2026 for weeks. There is no race condition.

### 7.6 Ordering: Postgres Drop Before or After ClickHouse Drop?

**Recommended: Postgres first, ClickHouse second.**

```
1. DROP Postgres partition (year N)
   └─ CDC sees nothing (DDL)
   └─ PeerDB LSN advances

2. DROP ClickHouse partitions (year N)
   └─ Removes orphaned data
```

Why this order:
- If ClickHouse drop fails (e.g., ClickHouse is temporarily down), **no correctness issue** — we just have orphaned data consuming storage. The cleanup can retry.
- If Postgres drop fails, we haven't touched ClickHouse yet — clean rollback.
- Doing Postgres first ensures that even if the orchestrator crashes between steps 1 and 2, no new CDC data for year N can ever appear in ClickHouse (the source partition is gone). The ClickHouse cleanup becomes idempotent and safe to retry.

If ClickHouse is down during cleanup, the orchestrator should:
1. Complete the Postgres partition drop
2. Record a pending ClickHouse cleanup task
3. Retry the ClickHouse drops when ClickHouse comes back online
4. The orphaned data doesn't cause correctness issues — queries for 2026 would return stale results, but the API no longer allows 2026 queries after `prior_school_year` is set to NULL

### 7.7 Membership Mirror Tables During Rollover

Membership mirrors (`membership_school_mirror`, etc.) use `ReplacingMergeTree` and are **not partitioned by school_year** — they represent current organizational relationships, not score data.

```sql
-- Membership mirrors: no year partitioning
CREATE TABLE trs.membership_school_mirror (
    tenant_id String, school_id String, student_id Int32,
    is_deleted UInt8 DEFAULT 0,
    version Int64 DEFAULT 0
) ENGINE = ReplacingMergeTree(version, is_deleted)
PARTITION BY tenant_id
ORDER BY (tenant_id, school_id, student_id);
```

**During rollover and overlap:** Membership mirrors continue syncing normally via the RTS Membership Sync Lambda → Aurora `rts.*` → PeerDB CDC → ClickHouse mirrors. They are unaffected by the year transition.

**At cleanup:** No membership mirror changes needed. These tables don't carry year-scoped data.

### 7.8 PeerDB Initial Load — Rebuild Scenario During Overlap

If ClickHouse needs to be rebuilt from scratch during the overlap period (e.g., total EC2 + EBS loss):

1. PeerDB runs an Initial Load snapshot from Aurora
2. Aurora has both 2026 and 2027 partitions during overlap
3. PeerDB reads all rows from `scores.student_opportunities` (both years)
4. Transformer emits `+1` sign-pairs for every row
5. ClickHouse Bronze and Silver are populated with both years
6. Both years become queryable again

If ClickHouse needs to be rebuilt **after cleanup** (2026 partition already dropped in Postgres):

1. PeerDB Initial Load snapshots only 2027 data (2026 partition is gone)
2. ClickHouse gets only 2027 data
3. This is correct — 2026 data is no longer needed

**No special handling needed.** The Initial Load process naturally reflects whatever partitions exist in Postgres at the time of the snapshot.

### 7.9 Summary: ClickHouse at Each Lifecycle Phase

| Phase | CDC Events for Year N | CDC Events for Year N+1 | ClickHouse Year N Data | ClickHouse Year N+1 Data |
|-------|----------------------|-------------------------|------------------------|--------------------------|
| **Active (year N only)** | Normal flow (inserts, rescores) | None | Active, receiving writes | Does not exist |
| **Rolling Over (~seconds)** | Final events draining | None yet | Active | Does not exist |
| **Overlap (1-2 months)** | **Zero** (Score Processor rejects) | Normal flow | **Static, frozen, fully merged** | Active, receiving writes |
| **Cleanup** | Zero | Normal flow | **Explicitly dropped via ALTER TABLE DROP PARTITION** | Active, unaffected |
| **Active (year N+1 only)** | Zero | Normal flow | Gone | Active |

### 7.10 What Could Go Wrong

| Scenario | Impact | Mitigation |
|----------|--------|------------|
| ClickHouse partition drop fails during cleanup | Orphaned 2026 data consumes storage; no query correctness issue since API blocks 2026 queries | Retry mechanism; alert on failed drops |
| ClickHouse is down during entire cleanup window | Same as above — orphaned data | Pending cleanup queue; execute when ClickHouse recovers |
| Late CDC event for 2026 arrives during overlap (shouldn't happen) | Row lands in 2026 Bronze partition; no harm since it's frozen | Score Processor year validation prevents this at the source |
| PeerDB lag causes 2026 events to arrive after Postgres partition drop | Cannot happen — PeerDB reads WAL sequentially; if the partition is dropped, no more 2026 DML exists in the WAL after that LSN | Sequential WAL ordering guarantees this |
| Someone manually inserts a 2026 score into Postgres after rollover | CDC would propagate it to ClickHouse 2026 partition | Postgres CHECK constraint on the 2027 partition prevents this; no 2026 partition exists to receive it |

---

## 8. Postgres Materialized Views

During overlap, MVs cover both years since they already group by `school_year`:

- **Current year MVs**: Refreshed on normal schedule (15-30 min)
- **Prior year MVs**: Data is frozen, so one final refresh at rollover is sufficient

At cleanup, the next MV refresh automatically drops prior year rows (the source partition is gone).

---

## 9. State Aggregates During Overlap

State aggregates work for both years independently:

| Year | Compute Strategy | Cache TTL |
|------|-----------------|-----------|
| Current year (2027) | ClickHouse Silver query every 2-4 hours | 2-4 hours |
| Prior year (2026) | **Compute once at rollover, cache until cleanup** | Until overlap expires |

Prior year state aggregates are essentially free — one query, then serve from cache for 60 days.

---

## 10. Storage During Overlap

Worst case is 2x the single-year footprint for the overlap duration:

| Resource | Single Year | During Overlap | After Cleanup |
|----------|------------|----------------|---------------|
| Postgres `student_opportunities` | ~22 GB/tenant | ~44 GB/tenant | ~22 GB/tenant |
| Postgres `student_component_scores` | ~660 GB/tenant | ~1.3 TB/tenant | ~660 GB/tenant |
| ClickHouse total | ~15 GB/tenant | ~30 GB/tenant | ~15 GB/tenant |
| Report cache entries | N entries | ~2N entries | N entries |

This is temporary and bounded. Aurora Serverless v2 handles the storage elastically. The 2x peak lasts at most `overlap_days`.

---

## 11. Comparison: Overlap Model vs. Immediate-Wipe Model

| Aspect | Immediate Wipe | Overlap Model |
|--------|---------------|---------------|
| Rollover downtime | ~1 min (drain + drop) | **~seconds** (just create partition) |
| Old data availability | Gone immediately | **Readable for 1-2 months** |
| CDC drain required at rollover | Yes (must flush before drop) | **No** (old data untouched) |
| API year parameter | Removed (derived) | **Kept** (with validation against allowed set) |
| Prior year cache TTL | N/A | **Very long** (data is frozen, cache until cleanup) |
| Peak storage | 1x | **2x for overlap duration** |
| Rollover orchestrator complexity | High (drain, drop, flush in one pass) | **Lower** (two simple phases) |
| User experience at rollover | Abrupt cutover, no prior year access | **Smooth transition with year picker** |

---

## 12. Summary of All Changes

| Area | Change | Complexity |
|------|--------|------------|
| `configs.tenants` table | Add `overlap_days`, `prior_school_year`, `overlap_expires_at` | Low |
| Rollover Orchestrator Phase 1 | Create new partition, update tenant config | Low |
| Rollover Orchestrator Phase 2 | Drop old partition, flush old cache (runs weeks later) | Low |
| Score Processor | Reject wrong-year scores | Low |
| API Lambda | Accept optional year param, validate against allowed years | Medium |
| Available Years endpoint | New endpoint for UI year picker | Low |
| Report cache | Add `tenant_id` column; aggressive TTL for prior year | Low |
| Cache prewarming | Compute prior year aggregates once at rollover | Low |
| ClickHouse | No schema changes; partition drops at cleanup | Low |
| Postgres MVs | One final refresh at rollover for prior year | Low |
| State aggregates | Prior year: compute once, cache for overlap duration | Low |
| UI | Year picker when `prior_year` is non-null | Medium |
