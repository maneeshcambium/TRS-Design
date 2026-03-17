# TRS — Query Patterns Reference

SQL queries for all API access patterns, organized by scope and tier.
See [TRS_Design_Document_v4.md](../TRS_Design_Document_v4.md) §9–§10 for the fallback logic.

---

## Roster Queries (Tier 2 / Tier 3)

### Overall Aggregate — ClickHouse Silver Path

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

### Overall Aggregate — Postgres Fallback

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

### Per-Student Score List (always Postgres — primary data)

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

### Standard-Level Class Aggregate (Postgres)

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

---

## School Aggregate Queries

### ClickHouse Gold (Tier 2)

```sql
SELECT
    school_id,
    test_group_id,
    avgMerge(avg_score)                  AS average_score,
    uniqMerge(student_count)             AS students_tested,
    groupUniqArrayMerge(pl_distribution) AS perf_level_bands
FROM trs.school_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND school_id IN (?)
GROUP BY school_id, test_group_id
ORDER BY school_id;
```

### Postgres MV (Tier 3a)

```sql
SELECT students_tested, pl1, pl2, pl3, pl4, pl5, avg_scale_score
FROM trs.mv_school_overall
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND school_id     = $4;
```

---

## District Aggregate Queries

### ClickHouse Gold (Tier 2)

```sql
SELECT
    district_id,
    test_group_id,
    avgMerge(avg_score)       AS average_score,
    uniqMerge(student_count)  AS students_tested,
    sumMerge(pl_1_count)      AS pl_1,
    sumMerge(pl_2_count)      AS pl_2,
    sumMerge(pl_3_count)      AS pl_3,
    sumMerge(pl_4_count)      AS pl_4,
    round(sumMerge(pl_1_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_1_pct,
    round(sumMerge(pl_2_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_2_pct,
    round(sumMerge(pl_3_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_3_pct,
    round(sumMerge(pl_4_count) * 100.0 / nullIf(uniqMerge(student_count), 0), 1) AS pl_4_pct
FROM trs.district_aggregates_gold
WHERE tenant_id = ? AND school_year = ? AND district_id IN (?)
GROUP BY district_id, test_group_id
ORDER BY district_id;
```

### Postgres MV (Tier 3a)

```sql
SELECT students_tested, pl1, pl2, pl3, pl4, pl5, avg_scale_score
FROM trs.mv_district_overall
WHERE tenant_id     = $1
  AND school_year   = $2
  AND test_group_id = $3
  AND district_id   = $4;
```

---

## Implementation Notes

The C# API Lambda fallback logic (embargo check → report_cache → ClickHouse Gold → Postgres MV → Postgres live) is documented in [API_Lambda.md](API_Lambda.md).

The ClickHouse in-process circuit breaker (200 ms `GET /ping`, 10 s check interval) is also in [API_Lambda.md](API_Lambda.md).

*See [TRS_Design_Document_v4.md](../TRS_Design_Document_v4.md) §9 for fallback chain design.*
