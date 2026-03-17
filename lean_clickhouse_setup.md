# Lean ClickHouse Setup

**Goal:** Keep ClickHouse permanently lean by giving each school year its own short-lived instance, then retiring it once the year stabilizes. ClickHouse never accumulates more than 1–2 years of raw data.

---

## The Annual Lifecycle

| Phase | Period | State |
|---|---|---|
| **Active** | Aug – Jul | 1 CH instance (current year only) |
| **Parallel Overlap** | Aug – Dec | 2 CH instances: current year (full compute) + trailing year (downsized, e.g. `t4g.medium`) |
| **Snapshot** | Nov – Dec | Aggregate Refresh Lambda does a final read of the trailing CH instance and bulk-upserts into Postgres `school_overall_aggregates` |
| **Sunset** | Dec | Trailing CH instance terminated; PeerDB mirror for that year stopped |

The trailing instance runs at reduced compute because it only handles occasional rescores, not high-volume ingestion.

---

## Four Key Mechanisms

### 1. Year-Filtered PeerDB Mirrors
Each CH instance receives CDC for its year only:
- **Mirror A:** `SELECT * FROM student_opportunities WHERE school_year = 2025` → CH Instance 2025
- **Mirror B:** `SELECT * FROM student_opportunities WHERE school_year = 2026` → CH Instance 2026

At sunset, Mirror A is stopped and the replication slot released. Mirror B continues unaffected.

### 2. `year_registry` Table (in ConfigsDb)
The API Lambda queries this table before every request to determine where to route it:

| Year Status | Route |
|---|---|
| `Active` | ClickHouse instance for that year |
| `Archived` | Postgres snapshot table (`school_overall_aggregates`) |

No code changes are needed at year boundaries — just a status update in this table.

### 3. Snapshot Table (in ScoresDb — Postgres)
`trs.school_overall_aggregates` stores pre-aggregated Gold-layer data permanently:
- Columns: `tenant_id`, `school_year`, `school_id`, `test_group_id`, `students_tested`, `avg_scale_score`, `pl_1_count` … `pl_10_count`
- Partitioned by `tenant_id`, primary key on `(school_year, test_group_id, school_id)`
- Historical summary queries hit this directly — no ClickHouse needed after archival

### 4. Component Score Fallback (Archived Years)
The 1.6B-row `student_component_scores` table is **not** fully snapshotted (too many dimension combinations). Instead:
- Active years → ClickHouse
- Archived years → live query against the Postgres partition (fast via partition pruning) + 7-day TTL in `report_cache`

---

## Sunset Process (Step-by-Step)

1. **Final Aggregation** — Aggregate Refresh Lambda queries CH Gold layer for the trailing year one last time.
2. **Bulk Upsert** — Results written to `trs.school_overall_aggregates` in ScoresDb.
3. **Archive Flag** — `year_registry` updated: trailing year status → `Archived`.
4. **API Redirection** — API Lambda stops routing that year to ClickHouse automatically.
5. **Stop Mirror** — PeerDB mirror for the trailing year is disabled.
6. **Terminate Instance** — Trailing CH EC2 instance is shut down. Billing stops immediately.

---

## What This Buys

- **ClickHouse query performance stays fast** — each instance only holds 1 year of data; no index bloat over time
- **~50% compute cost reduction each December** — terminating the trailing instance is the biggest lever
- **Historical data remains queryable** — it moves from ClickHouse → Postgres, not deleted
- **No schema migration risk** — each new year starts from a clean DDL script

---

## Postgres MVs (Companion Cleanup)

Year-specific Materialized Views in Postgres are created with a hardcoded year filter:
```sql
-- Active MV — only scans the 2026 partition
CREATE MATERIALIZED VIEW mv_standards_2026 AS
  SELECT ... FROM student_component_scores WHERE school_year = 2026;
```
Once a year is archived, its MV is `DROP`ped. Only the current year's MV remains, keeping Postgres index sizes stable.
