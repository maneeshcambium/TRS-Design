# TRS Design Amendment: One-Year Data Window + Per-Tenant Rollover

**Status**: Draft
**Date**: 2026-03-22
**Amends**: TRS Engineering Design Document v7

---

## Overview

The original design accumulates data across school years (partitioned by `school_year`). This amendment changes the model so that **each tenant retains only one year of data**. At rollover, the old year is wiped and the new year starts clean. Each tenant has its own rollover date, making this a rolling process across the fleet.

This bounded data model also makes **state-level aggregates** feasible on a tighter refresh schedule.

---

## 1. Tenant Configuration — New Rollover Schedule

The `configs.tenants` table needs per-tenant rollover metadata:

```sql
ALTER TABLE configs.tenants ADD COLUMN
    fiscal_year_start_month  SMALLINT NOT NULL DEFAULT 7,  -- July = typical US school year
    fiscal_year_start_day    SMALLINT NOT NULL DEFAULT 1,
    rollover_lead_days       SMALLINT NOT NULL DEFAULT 3,  -- maintenance window before new year
    current_school_year      SMALLINT NOT NULL,            -- e.g. 2026
    rollover_status          TEXT     NOT NULL DEFAULT 'active';
    -- 'active' | 'rollover_pending' | 'rolling_over' | 'rollover_complete'
```

**Why `current_school_year` is explicit, not derived**: During the rollover window, the "current" year is ambiguous. Making it an explicit, operator-controlled field avoids edge cases where time-based derivation disagrees with the actual data state.

---

## 2. Partition Strategy — Only One Active Year Per Tenant

### Current Design

L1 = `LIST(tenant_id)`, L2 = `LIST(school_year)` — accumulates years.

### New Model

Same partitioning scheme, but **operationally only one L2 partition exists per tenant at any time**. The partition structure is the *mechanism* for instant year wipe:

```
scores.student_opportunities
  └─ p_tx          (tenant_id = 'tx')
  │   └─ p_tx_2026   (school_year = 2026)     ← only active partition
  └─ p_va          (tenant_id = 'va')
      └─ p_va_2026   (school_year = 2026)     ← only active partition
```

### Rollover Partition Swap (example: TX, July 1)

```sql
-- Step 1: Create new year partition
CREATE TABLE scores.student_opportunities_tx_2027
  PARTITION OF scores.student_opportunities_tx
  FOR VALUES IN (2027);

-- Step 2: Drop old year partition (instant, no row-by-row delete)
DROP TABLE scores.student_opportunities_tx_2026;
```

This is **O(1)** regardless of row count — Postgres drops the underlying file. No vacuum, no lock contention on other tenants.

Same applies to `student_component_scores`, `score_ingest_log`, etc.

---

## 3. Rollover Orchestration — New Component

A **Year Rollover Orchestrator** (Step Function or dedicated Lambda) runs per tenant:

```
┌─────────────────────────────────────────────────────────────────┐
│                  Rollover State Machine (per tenant)            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ACTIVE ──► ROLLOVER_PENDING ──► ROLLING_OVER ──► ACTIVE       │
│                                                                 │
│  EventBridge cron     Drain window:       Partition swap:       │
│  checks daily if      - Stop ingestion    - Create new year    │
│  rollover_date -      - Flush SQS queue     partition           │
│  lead_days reached    - Wait for CDC lag  - Drop old year      │
│                         to hit 0            partition           │
│                       - Set status =      - Flush report_cache │
│                         'rolling_over'      for tenant          │
│                                           - Wipe ClickHouse    │
│                                             partitions          │
│                                           - Resume ingestion   │
│                                           - Set current_school │
│                                             _year = new year   │
│                                           - Set status =       │
│                                             'active'           │
└─────────────────────────────────────────────────────────────────┘
```

### Critical Ordering

1. **Pause ingestion** for this tenant — reject SQS messages back to queue with a visibility timeout, or filter at Score Processor by checking `rollover_status`
2. **Drain CDC pipeline** — wait until PeerDB replication slot lag = 0 and Transformer queue is empty for this tenant
3. **Postgres partition swap** — create new year partition, drop old year partition
4. **ClickHouse partition drop** — `ALTER TABLE ... DROP PARTITION ('tx', 2026)` on Bronze, Silver, and component tables
5. **Flush `report_cache`** — delete all cached entries for this tenant
6. **Refresh Postgres MVs** — `REFRESH MATERIALIZED VIEW CONCURRENTLY` (they come back empty for this tenant)
7. **Resume ingestion**

### Rollover Window Duration

Steps 3-6 take seconds. The drain in step 2 is the bottleneck — typically < 30 seconds if CDC lag is healthy. **Total tenant downtime: under 1 minute.**

Other tenants are completely unaffected — all operations are tenant-scoped via partitions.

---

## 4. Score Processor Changes

Two additions to the Score Processor Lambda:

```csharp
// 1. Reject scores during rollover
var tenant = await GetTenantConfig(tenantId);
if (tenant.RolloverStatus == "rolling_over")
{
    // Return message to SQS with 60s visibility timeout
    // It will be retried after rollover completes
    throw new RolloverInProgressException();
}

// 2. Validate school_year matches current_school_year
if (score.SchoolYear != tenant.CurrentSchoolYear)
{
    // Reject — we don't accept scores for prior years
    await LogRejection("WRONG_SCHOOL_YEAR", ...);
    return;
}
```

---

## 5. State Aggregates — Now Feasible

### Why the Original Design Deferred State Aggregates

With unbounded historical data, state aggregates over 5.5M students were too expensive for real-time:
- Postgres: > 60s
- ClickHouse: 30-60s
- Only run nightly

### Why One-Year Fixes This

Data volume per tenant is **bounded and predictable** — it never grows beyond one year. We can pre-compute state aggregates on a tighter schedule.

### State Aggregate Strategy

| Approach | Frequency | Latency | Where |
|----------|-----------|---------|-------|
| ClickHouse Silver + full membership JOIN | Every 2-4 hours | 30-60s query time | Aggregate Refresh Lambda |
| Postgres MV (`mv_state_overall`) | Nightly | < 5ms read | New materialized view |
| `report_cache` (state scope) | Refreshed by above | ~3ms | API Tier 1 |

### New Materialized View

```sql
CREATE MATERIALIZED VIEW scores.mv_state_overall AS
SELECT
    so.tenant_id,
    so.school_year,
    so.testalias_id,
    count(*)                                          AS students_tested,
    avg(so.overall_scale_score)                       AS avg_scale_score,
    count(*) FILTER (WHERE so.overall_perf_level = 1) AS pl_1,
    count(*) FILTER (WHERE so.overall_perf_level = 2) AS pl_2,
    count(*) FILTER (WHERE so.overall_perf_level = 3) AS pl_3,
    count(*) FILTER (WHERE so.overall_perf_level = 4) AS pl_4
FROM scores.student_opportunities so
WHERE so.is_aggregate_eligible = TRUE
  AND so.use_for_aggregation = TRUE
GROUP BY so.tenant_id, so.school_year, so.testalias_id;
```

**Why state aggregates don't need a membership JOIN**: At state scope, every student in the tenant's `student_opportunities` table is "in the state" — no organizational filtering needed. The membership JOIN is only required for school/district/roster scopes.

### Updated Cache TTLs

| Scope | TTL | Change from Original |
|-------|-----|----------------------|
| Roster | No cache (live) | No change |
| School | 15 min | No change |
| District | 5 min | No change |
| **State** | **2-4 hours** | **Was nightly — now tighter because dataset is bounded** |

### Updated API Scope Routing — State Row Added

| Scope | Tier 1 (Cache) | Tier 2 (ClickHouse) | Tier 3 (Postgres) | TTL |
|-------|----------------|---------------------|--------------------|-----|
| Roster | No cache | Silver per student | Aurora live query | N/A |
| School | `report_cache` | Silver + school mirror JOIN | MV then raw live | 15 min |
| District | `report_cache` | Silver + district mirror JOIN | MV then raw live | 5 min |
| **State** | **`report_cache`** | **Silver (no membership JOIN)** | **`mv_state_overall`** | **2-4 hours** |
| School (components) | `report_cache` | Component Silver + opp JOIN | MV (current year) | 15 min |
| District (components) | `report_cache` | Component Silver + opp JOIN | HTTP 503 if unavailable | 15 min |

---

## 6. ClickHouse Changes

### Partition Drop During Rollover

```sql
-- Bronze
ALTER TABLE trs.student_scores_bronze
  DROP PARTITION ('tx', 2026);

-- Silver
ALTER TABLE trs.student_scores_silver
  DROP PARTITION ('tx', 2026);

-- Component tables
ALTER TABLE trs.student_component_scores_bronze
  DROP PARTITION ('tx', 2026);
ALTER TABLE trs.student_component_scores_silver
  DROP PARTITION ('tx', 2026);
```

These are instant — ClickHouse drops partition directories on disk.

**No schema changes needed** — the existing `PARTITION BY (tenant_id, school_year)` already supports this.

---

## 7. Report Cache — Tenant-Scoped Flush

### Current cache_key format

`"scope#tenant#entity#year#test"` — e.g. `"school#tx#s-789#2026#cp1-g5-ela"`

### Add tenant_id column for efficient deletion

```sql
ALTER TABLE scores.report_cache ADD COLUMN tenant_id TEXT;
CREATE INDEX idx_report_cache_tenant ON scores.report_cache(tenant_id);
```

Rollover flush becomes:

```sql
DELETE FROM scores.report_cache WHERE tenant_id = 'tx';
```

Without the column, fallback is prefix-based:

```sql
DELETE FROM scores.report_cache
WHERE cache_key LIKE '%#tx#%';
```

---

## 8. API Changes

### Year is No Longer a Query Parameter

The API derives year from tenant config instead of accepting it from the caller:

```csharp
// Before: caller passes school_year
var year = request.SchoolYear;

// After: derive from tenant config
var tenant = await GetTenantConfig(tenantId);
if (tenant.RolloverStatus == "rolling_over")
    return StatusCode(503, "Year rollover in progress. Retry shortly.");
var year = tenant.CurrentSchoolYear;
```

During the rollover window, the API returns **503** for that tenant only. Other tenants are unaffected.

---

## 9. Storage & Cost Impact

With only one year of data retained, storage is bounded:

| Resource | Before (accumulating) | After (1 year) | Impact |
|----------|----------------------|-----------------|--------|
| Postgres `student_opportunities` | Grows ~22 GB/year/tenant | Fixed ~22 GB/tenant | No storage growth |
| Postgres `student_component_scores` | Grows ~660 GB/year/tenant | Fixed ~660 GB/tenant | Huge savings |
| ClickHouse total | Grows ~15 GB/year/tenant | Fixed ~15 GB/tenant | Can downsize EC2 |
| Aurora Serverless ACU | Scales with data | Bounded | Predictable cost |
| Backup size | Grows | Bounded | Cheaper snapshots |

---

## 10. Summary of All Changes

| Area | Change | Complexity |
|------|--------|------------|
| `configs.tenants` table | Add rollover schedule columns | Low |
| **Rollover Orchestrator** | **New Step Function / Lambda** | **High** — most complex new component |
| Score Processor | Reject wrong-year scores; pause during rollover | Low |
| Postgres partitions | Operational: only 1 year partition per tenant | No schema change |
| ClickHouse partitions | Drop old year during rollover | No schema change |
| `report_cache` | Add `tenant_id` column; flush on rollover | Low |
| Postgres MVs | Add `mv_state_overall`; refresh on rollover | Medium |
| API Lambda | Derive year from tenant config; 503 during rollover; serve state scope | Medium |
| Aggregate Refresh | Add state scope on 2-4 hour schedule | Low |
| Cache TTLs | State scope: nightly to 2-4 hours | Config change |

### Key Insight

The partition-based architecture in the original design is already well-suited for this. The `LIST(school_year)` L2 partition was the right call — it makes year wipes instantaneous via `DROP TABLE` on the partition. The **Rollover Orchestrator** is the only genuinely new component; everything else is adjustments to existing pieces.
