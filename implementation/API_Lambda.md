# Implementation: API Lambda — Request Routing & Query Logic

**Belongs to:** [TRS ClickHouse Architecture Design](../TRS_ClickHouse_Design.md)  
**Component:** API Layer — AWS Lambda behind API Gateway  
**Auth:** Red Hat SSO (OIDC) — external IdP and sole source of user identity and roles. The Lambda authorizer validates the `id_token` (RS256) against the RH SSO JWKS endpoint before the request reaches the API Lambda. JWT carries `tenant_id` and `roles` (string array) claims. **TRS has no local users table** — all role resolution is from JWT claims only. Known roles: `STATE_ADMIN`, `DISTRICT_ADMIN`, `SCHOOL_ADMIN`, `TEACHER`, `EMBARGO_VIEWER`.

---

## Purpose

The API Lambda is the single serving layer for all TRS reporting queries. PostgreSQL (Aurora) is the **primary data store** — it holds all scores, membership, config, and the aggregate cache. ClickHouse is a **pure compute engine** — it accelerates aggregation queries and writes results back to Aurora. If ClickHouse is unavailable, Aurora computes aggregates directly.

No DynamoDB. A single `trs.report_cache` table in Aurora serves as the cache store.

---

## Three-Tier Fallback Strategy

```
Incoming request
        │
        ▼
[Tier 1] Aurora report_cache ──── HIT (not expired) ──▶  Return result (<5ms)
        │ MISS or expired
        ▼
[Tier 2] ClickHouse healthy? ──── YES ──▶  Compute aggregate on ClickHouse
        │                                   Write result → Aurora report_cache
        │                                   Return result
        │ NO (timeout / 5xx / unreachable)
        ▼
[Tier 3] Aurora live aggregate ──────────▶  Compute aggregate directly on Aurora
                                            Write result → Aurora report_cache
                                            Return result  (may be slower for large districts)
                                            Log metric: TRS/API/ClickHouseFallback
```

ClickHouse health is checked via a 200ms timeout probe on `GET /ping`. A failure routes immediately to Tier 3 — no partial ClickHouse query is attempted.

---

## Aurora report_cache Table

```sql
CREATE TABLE trs.report_cache (
    cache_key    TEXT        PRIMARY KEY,
    -- Key format: "{scope}#{tenant_id}#{scope_id}#{school_year}#{test_group_id}"
    -- Examples:
    --   "district#tenantX#d-456#2026#cp1-g5-ela"
    --   "school#tenantX#s-789#2026#cp1-g5-ela"
    --   "state#tenantX#TX#2026#cp1-g5-ela"
    payload      JSONB       NOT NULL,
    computed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ NOT NULL,
    computed_by  TEXT        NOT NULL  -- 'clickhouse' | 'postgres_live' | 'nightly_job'
);
CREATE INDEX ON trs.report_cache (expires_at);  -- for cleanup job
```

**Cache TTLs by scope:**

| Scope | TTL | Rationale |
|-------|-----|-----------|
| Roster | No cache | Always live — per-teacher, ~30 students, <5ms |
| School | 5 minutes | Fast ClickHouse query; short TTL keeps it fresh |
| District | 5 minutes | Balances freshness vs. cost for large districts |
| State | Until next nightly run | State aggregates change slowly; nightly is sufficient |

**Cleanup:** pg_cron job runs nightly:
```sql
DELETE FROM trs.report_cache WHERE expires_at < now();
```

---

## Scope Routing Table

| Scope | Tier 1 (Postgres Cache) | Tier 2 (ClickHouse) | Tier 3 (Aurora Live) | Cache TTL |
|-------|------------------------|---------------------|----------------------|-----------|
| Roster | — (no cache) | Live query Q1–Q4 | Aurora live query | N/A — always live |
| School | report_cache | JOIN student_scope_today | Aurora aggregate query | 5 minutes |
| District | report_cache | JOIN student_scope_today | Aurora aggregate query | 5 minutes |
| State | report_cache | Live query (cold start only) | Aurora aggregate query | Until next nightly run |

---

## Roster Path (no cache)

```
GET /v1/roster/{rosterId}/aggregate?testGroupId={X}&schoolYear={Y}

1. Validate JWT; extract tenant_id, user_id, roles.
2. Embargo pre-check:
     SELECT embargo_until FROM trs.test_configs
       WHERE tenant_id = ? AND test_group_id = ?
     If embargo_until IS NOT NULL AND embargo_until > NOW():
       caller roles contain 'EMBARGO_VIEWER'?  NO  → return 404 Not Found.
     (Result cached in-process 60 s.)
3. Authorize: SELECT 1 FROM roster_assignments
     WHERE tenant_id=? AND user_id=? AND roster_id=?   → 403 if not found.
4. Resolve student list:
   SELECT student_id FROM roster_members_today
     WHERE tenant_id=? AND roster_id=?                 -- typically ≤30 rows
5. Probe ClickHouse health (200ms timeout on /ping).
6a. [Tier 2] ClickHouse healthy:
     Run roster queries Q1–Q4 with student_id IN (step 4 result).
6b. [Tier 3] ClickHouse unavailable:
     Run equivalent queries directly on Aurora student_opportunities.
7. Return 200; asOfTs = now(); servedFrom = 'clickhouse' | 'postgres_fallback'.
```

Roster scope is never cached — it is always computed live. Latency: <5ms (ClickHouse FINAL on ~30 rows).

---

## School Path

```
GET /v1/school/{schoolId}/aggregate?testGroupId={X}&schoolYear={Y}

1. Validate JWT; authorize school access.
2. Embargo pre-check:
     SELECT embargo_until FROM trs.test_configs
       WHERE tenant_id = ? AND test_group_id = ?
     If embargo_until IS NOT NULL AND embargo_until > NOW():
       caller roles contain 'EMBARGO_VIEWER'?  NO  → return 404 Not Found.
     (Result cached in-process 60 s.)
3. Check report_cache:
   SELECT payload, computed_at FROM trs.report_cache
     WHERE cache_key = 'school#{tenant_id}#{school_id}#{school_year}#{test_group_id}'
       AND expires_at > now()
   → HIT: return payload; servedFrom = 'cache'.
3. Probe ClickHouse health.
4a. [Tier 2] ClickHouse healthy:
     Run school aggregate queries (JOIN student_scope_today WHERE school_id=?).
     Write to report_cache ON CONFLICT DO UPDATE; expires_at = now() + '5 min'.
     Return result; servedFrom = 'clickhouse'. Latency: <30ms.
4b. [Tier 3] ClickHouse unavailable:
     Run equivalent aggregation on Aurora (JOIN school_roster_members on school_id).
     Write to report_cache with same TTL.
     Return result; servedFrom = 'postgres_fallback'.
```

---

## District Path

```
GET /v1/district/{districtId}/aggregate?testGroupId={X}&schoolYear={Y}

1. Validate JWT; authorize district access.
2. Embargo pre-check:
     SELECT embargo_until FROM trs.test_configs
       WHERE tenant_id = ? AND test_group_id = ?
     If embargo_until IS NOT NULL AND embargo_until > NOW():
       caller roles contain 'EMBARGO_VIEWER'?  NO  → return 404 Not Found.
     (Result cached in-process 60 s.)
3. Check report_cache:
   SELECT payload, computed_at FROM trs.report_cache
     WHERE cache_key = 'district#{tenant_id}#{district_id}#{school_year}#{test_group_id}'
       AND expires_at > now()
   → HIT: return payload; servedFrom = 'cache'. (~3ms)
3. Probe ClickHouse health.
4a. [Tier 2] ClickHouse healthy:
     Run district aggregate queries Q1–Q4:
       Q1: Overall aggregate (count, PL distribution, avg score)
       Q2: Per-school breakdown
       Q3: Standards aggregate (per-standard PL distribution)
       Q4: Demographic slice (gender × ethnicity × ELL)
     All queries JOIN trs.student_scope_today WHERE district_id=?;
     no Aurora round-trip for member IDs.
     Write result to Aurora:
       INSERT INTO trs.report_cache (...)
       ON CONFLICT (cache_key) DO UPDATE SET
         payload=EXCLUDED.payload, computed_at=now(),
         expires_at=now() + INTERVAL '5 minutes',
         computed_by='clickhouse';
     Return result; servedFrom = 'clickhouse'.
4b. [Tier 3] ClickHouse unavailable:
     Run equivalent aggregation on Aurora
       (JOIN school_roster_members WHERE district_id=?).
     Write to report_cache with same TTL; computed_by='postgres_live'.
     Return result; servedFrom = 'postgres_fallback'.
     Log CloudWatch metric: TRS/API/ClickHouseFallback (district_size dimension).
```

Expected latency: <5ms on cache hit; 50–500ms ClickHouse on miss (district-size dependent); 1–10s Aurora fallback on miss.

---

## State Path

```
GET /v1/state/{tenantId}/aggregate?testGroupId={X}&schoolYear={Y}

1. Validate JWT; authorize state-level access.
2. Embargo pre-check:
     SELECT embargo_until FROM trs.test_configs
       WHERE tenant_id = ? AND test_group_id = ?
     If embargo_until IS NOT NULL AND embargo_until > NOW():
       caller roles contain 'EMBARGO_VIEWER'?  NO  → return 404 Not Found.
     (Result cached in-process 60 s.)
3. Check report_cache:
   SELECT payload FROM trs.report_cache
     WHERE cache_key = 'state#{tenant_id}#state#{school_year}#{test_group_id}'
       AND expires_at > now()
   → HIT: return payload; servedFrom = 'cache'. (~3ms)
3. MISS (nightly job hasn't run yet, or first cold start):
3a. [Tier 2] ClickHouse healthy:
     Compute live (1–3s at TX scale).
     Write to report_cache; expires_at = date_trunc('day', now()) + INTERVAL '1 day'.
     Return result.
3b. [Tier 3] ClickHouse unavailable:
     Compute on Aurora (1–5s).
     Write to report_cache with same TTL.
     Return result; servedFrom = 'postgres_fallback'.
```

State aggregates are pre-populated nightly so the first user of the day always hits Tier 1. The live fallback is a safety net only.

---

## Test Discovery Endpoint

This is the **only endpoint through which a caller learns which `test_group_id`s exist** for a
tenant and school year. By filtering embargoed tests here for regular users, the rest of the
aggregate endpoints never receive requests for invisible tests under normal flow.

```
GET /v1/{tenantId}/tests?schoolYear={Y}

1. Validate JWT; extract tenant_id, roles.
2. Build query based on role:
   a. Regular user (no EMBARGO_VIEWER role):
        SELECT test_group_id, test_family, subject, grade
        FROM trs.test_configs
        WHERE tenant_id   = ?
          AND school_year = ?
          AND (embargo_until IS NULL OR embargo_until <= NOW())
        ORDER BY test_group_id;

   b. EMBARGO_VIEWER:
        SELECT test_group_id, test_family, subject, grade, embargo_until
        FROM trs.test_configs
        WHERE tenant_id   = ?
          AND school_year = ?
        ORDER BY test_group_id;
        -- embargo_until included so the viewer knows which tests are still under embargo
3. Return list; HTTP 200.
```

EmbargoViewer response includes an `isEmbargoed` field per entry so the UI can
display a visual indicator on unreleased tests:

```json
{
  "schoolYear": 2026,
  "tests": [
    { "testGroupId": "cp1-g5-ela-2026",  "subject": "ELA",  "grade": "Grade5", "isEmbargoed": false },
    { "testGroupId": "cp1-g5-math-2026", "subject": "Math", "grade": "Grade5", "isEmbargoed": true,
      "embargoUntil": "2026-04-01T00:00:00Z" }
  ]
}
```

---

## Nightly Aggregate Job (EventBridge Scheduler, 02:00 AM per tenant)

Pre-warms the state and school cache entries so the first user of each day always hits Tier 1. Runs as a separate Lambda.

```
For each (tenant_id, school_year, test_group_id) combination:

  1. Execute state aggregate query on ClickHouse (or Aurora if unavailable).
  2. INSERT INTO trs.report_cache ON CONFLICT DO UPDATE:
       expires_at = date_trunc('day', now()) + INTERVAL '1 day 3 hours'
       -- Expires at 03:00 AM next night (buffer for job runtime variance)

  3. For each school in the tenant: parallelise up to 20 concurrent ClickHouse queries.
     Write each school result to report_cache with expires_at = now() + INTERVAL '5 min'.
     (School cache uses short TTL even from the nightly job — on-demand freshness
      is more important than pre-warming for school scope.)

If ClickHouse is unavailable:
  Compute all aggregates on Aurora.
  Write to report_cache.
  Morning users are all served from cache — no one triggers a slow live fallback.
```

---

## Aurora Equivalent Queries (Tier 3 Fallback)

These are the Postgres SQL equivalents run when ClickHouse is unavailable. Correct but slower at large scale.

**District overall aggregate (Postgres fallback):**
```sql
SELECT
    count(*)                                        AS students_tested,
    count(*) FILTER (WHERE overall_perf_level = 1)  AS pl1,
    count(*) FILTER (WHERE overall_perf_level = 2)  AS pl2,
    count(*) FILTER (WHERE overall_perf_level = 3)  AS pl3,
    count(*) FILTER (WHERE overall_perf_level = 4)  AS pl4,
    avg(overall_scale_score)                        AS avg_scale_score
FROM trs.student_opportunities so
JOIN trs.school_roster_members rm
    ON rm.tenant_id = so.tenant_id AND rm.student_id = so.student_id AND rm.active = true
WHERE so.tenant_id             = :tenant_id
  AND so.school_year            = :school_year
  AND so.test_group_id          = :test_group_id
  AND rm.district_id            = :district_id
  AND so.is_aggregate_eligible  = true;
```

Expected latency: 200ms–10s depending on district size. Acceptable as an emergency fallback; the result is written to `report_cache` immediately so all subsequent requests for the same scope are fast.

---

## Tenant & Security Isolation

- Every query — ClickHouse or Aurora — is gated by `tenant_id` from the verified JWT. URL parameter `tenant_id` is validated against the JWT claim; mismatch returns 403.
- `student_scope_today` JOINs are always bounded by `tenant_id`.
- `report_cache` keys are prefixed with `tenant_id` — cross-tenant cache collisions are structurally impossible.

---

## Response Shape

```json
{
  "scope": "district",
  "scopeId": "district-123",
  "testGroupId": "checkpoint1-grade5-ela-2026",
  "schoolYear": 2026,
  "asOfTs": "2026-03-03T14:32:00.123Z",
  "servedFrom": "clickhouse",        // "cache" | "clickhouse" | "postgres_fallback"
  "aggregate": {
    "studentsTested": 4820,
    "avgScaleScore": 241.7,
    "perfLevelDistribution": {
      "pl1": 823, "pl2": 1102, "pl3": 1954, "pl4": 941
    },
    "perfLevelPercent": {
      "pl1": 17.1, "pl2": 22.9, "pl3": 40.5, "pl4": 19.5
    }
  },
  "bySchool": [ ... ],
  "byStandard": [ ... ],
  "byDemographic": [ ... ]
}
```

`servedFrom` is logged and emitted as a CloudWatch EMF metric to track cache hit rate and ClickHouse availability over time. Alert threshold: `postgres_fallback` rate > 5% over a 10-minute window triggers an on-call notification.
