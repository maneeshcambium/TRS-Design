
## 10. Query Patterns

### 10.1 Roster Queries (Tier 2 — ClickHouse Silver or direct Bronze)

**Overall aggregate (ClickHouse Silver path):**
```sql
SELECT
    student_id,
    argMaxMerge(latest_score)      AS score,
    argMaxMerge(latest_perf_level) AS perf_level,
    argMaxMerge(is_eligible)       AS eligible,
    maxMerge(last_updated)         AS scored_at
FROM trs.student_scores_silver
WHERE tenant_id = ? AND school_year = ? AND test_group_id = ?
  AND student_id IN (?)           -- ~30 IDs resolved from Aurora roster_members
GROUP BY student_id, opp_key
HAVING argMaxMerge(is_deleted) = 0;
```

**Overall aggregate (Postgres fallback):**
```sql
SELECT
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE overall_perf_level = 1)        AS pl1,
    COUNT(*) FILTER (WHERE overall_perf_level = 2)        AS pl2,
    COUNT(*) FILTER (WHERE overall_perf_level = 3)        AS pl3,
    COUNT(*) FILTER (WHERE overall_perf_level = 4)        AS pl4,
    AVG(overall_scale_score)                              AS avg_scale_score
FROM trs.student_opportunities
WHERE tenant_id            = $1
  AND school_year          = $2
  AND test_group_id        = $3
  AND student_id           = ANY($4)
  AND is_aggregate_eligible = TRUE;
```

**Per-student score list (always Postgres — primary data):**
```sql
SELECT
    student_id,
    opp_key,
    overall_scale_score,
    overall_perf_level,
    overall_raw_score,
    overall_standard_error,
    opp_status,
    condition_code,
    tested_date
FROM trs.student_opportunities
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND student_id    = ANY($4)
ORDER BY student_id;
```

**Standard-level class aggregate (Postgres):**
```sql
SELECT
    component_id                                          AS standard_id,
    COUNT(*)                                              AS students_tested,
    COUNT(*) FILTER (WHERE perf_level = 1)               AS pl1,
    COUNT(*) FILTER (WHERE perf_level = 2)               AS pl2,
    COUNT(*) FILTER (WHERE perf_level = 3)               AS pl3,
    AVG(scale_score)                                     AS avg_scale_score
FROM trs.student_component_scores
WHERE tenant_id            = $1
  AND school_year          = $2
  AND test_group_id        = $3
  AND student_id           = ANY($4)
  AND component_type       = 'STANDARD'
  AND is_aggregate_eligible = TRUE
GROUP BY component_id
ORDER BY component_id;
```

### 10.2 School / District Queries (Tier 2 — ClickHouse Gold)

**School aggregate — ClickHouse Gold (Tier 2):**
```sql
SELECT
    school_id,
    test_group_id,
    avgMerge(avg_score)                  AS average_score,
    uniqMerge(student_count)             AS students_tested,
    groupUniqArrayMerge(pl_distribution) AS perf_level_bands
FROM trs.school_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND school_id IN (?)
GROUP BY school_id, test_group_id;
```

**School aggregate — Postgres MV (Tier 3a):**
```sql
SELECT students_tested, pl1, pl2, pl3, pl4, pl5, avg_scale_score
FROM trs.mv_school_overall
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND school_id     = $4;
```

**District aggregate — ClickHouse Gold (Tier 2):**
```sql
SELECT
    district_id,
    test_group_id,
    avgMerge(avg_score)                                       AS average_score,
    uniqMerge(student_count)                                  AS students_tested,
    sumMerge(pl_1_count)                                      AS pl_1,
    sumMerge(pl_2_count)                                      AS pl_2,
    sumMerge(pl_3_count)                                      AS pl_3,
    sumMerge(pl_4_count)                                      AS pl_4,
    round(sumMerge(pl_1_count)*100.0/nullIf(uniqMerge(student_count),0),1) AS pl_1_pct
FROM trs.district_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND district_id IN (?)
GROUP BY district_id, test_group_id;
```

**District aggregate — Postgres MV (Tier 3a):**
```sql
SELECT students_tested, pl1, pl2, pl3, pl4, pl5, avg_scale_score
FROM trs.mv_district_overall
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND district_id   = $4;
```

---

## 11. Aggregate Cache & Materialized Views
### 11.1 Serving Strategy by Scope

#### Overall Score Aggregates

| Scope | Rows scanned | ClickHouse Gold latency | Postgres MV latency | Postgres raw latency | Cache TTL |
|-------|-------------|------------------------|--------------------|---------------------|------------|
| Roster (~30 students) | ~30 | <5 ms | N/A | <10 ms (index seek) | No cache |
| School (~6,800 students) | ~6,800 | <30 ms | <5 ms (MV lookup) | 200–800 ms | 15 min |
| District ≤300k | ≤300k | 50–200 ms | <5 ms (MV lookup) | 3–15 s | 5 min |
| District 300k–2M | ~500k–2M | 150–500 ms | <5 ms (MV lookup) | 15–120 s | 5 min |
| State (TX 5.5M) | 5.5M | 400 ms–3 s | <5 ms (MV lookup) | Too slow — cache only | Nightly |

> **Rule for Postgres raw path:** A direct `GROUP BY` on `student_opportunities` for large
> districts or state is too slow for an interactive API response. At district/state scope,
> always prefer the Postgres MV or `report_cache`. Only fall back to a live Postgres raw query
> for school scope and below.

#### Component Score Aggregates (Standards, Reporting Categories, Writing Dimensions)

`student_component_scores` can scale to **1.65 billion rows/year** at Texas scale. The materialization strategy differs from overall scores:

| Scope | ClickHouse Strategy | Postgres Strategy |
|-------|---------------------|-------------------|
| **Roster** | Direct index seek via `idx_scs_query` | **Primary path** — always live |
| **School** | `component_aggregates_gold` `avgMerge`/`sumMerge` | Current-year: `mv_school_standards_current`; historical: long-TTL `report_cache` entry |
| **District** | `component_aggregates_gold` school-id roll-up | **Disabled** — no live or MV fallback; return HTTP 503 if both cache and ClickHouse unavailable |

**Guardrails for component Tier 3b (live Postgres):**
- Live Postgres component scans are permitted **only for school scope** (bounded by `idx_scs_query`).
- **Historical school components** (years other than current): if ClickHouse is down and no `report_cache` entry exists, compute live from `student_component_scores` and store in `report_cache` with a long TTL (e.g. 24 hours) to prevent re-scanning on every request.
- **District-wide live component aggregates on Postgres are disabled** to prevent Aurora exhaustion. A `report_cache` miss + ClickHouse unavailability at district scope returns HTTP 503 with a `Retry-After` header.

### 11.2 Cache Invalidation

| Trigger | Action |
|---------|--------|
| TTL expires (`expires_at < NOW()`) | API serves stale cache + triggers async Aggregate Refresh Lambda |
| Rescore event arrives at Score Processor | Publish to SNS `trs-rescore-events`; Aggregate Refresh Lambda subscribes, invalidates affected `report_cache` rows, re-queries ClickHouse Gold |
| RTS Membership Sync completes | Aggregate Refresh Lambda re-runs Postgres MV refreshes |
| Manual admin action | `DELETE FROM trs.report_cache WHERE cache_key LIKE 'district#tx#d-456%'` |
| **Embargo applied to a previously visible test** | Ops must manually purge affected `report_cache` entries: `DELETE FROM trs.report_cache WHERE cache_key LIKE '%#{test_group_id}%'`. Until purged, existing cache entries remain accessible to anyone who holds the exact key — in practice negligible since regular users only discover tests via the discovery endpoint (which already filters by embargo). Also invalide the in-process `EmbargoService` cache by waiting up to 60 s or redeploying. |
| **Embargo lifted** (`embargo_until` set to past or NULL) | No action required. The `report_cache` entries that were populated by `EMBARGO_VIEWER` users remain valid and will be served immediately once regular users can access the test. The in-process cache expires within 60 s automatically. |

### 11.3 Nightly Pre-Warm Job (EventBridge Scheduler, 02:00 AM per tenant)

Pre-computes state and school aggregates so the first user of the day always hits Tier 1. Runs as a separate Lambda.

```
For each (tenant_id, school_year, test_group_id) combination:
  1. Query ClickHouse Gold → build state aggregate
  2. INSERT INTO report_cache (cache_key='state#...',  expires_at='tomorrow 02:00', computed_by='nightly_job')
  3. For each school in tenant:
     Query ClickHouse Gold → build school aggregate
     INSERT INTO report_cache (cache_key='school#...', expires_at=now()+15min, computed_by='nightly_job')
  4. REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_school_overall
  5. REFRESH MATERIALIZED VIEW CONCURRENTLY trs.mv_district_overall
```

---