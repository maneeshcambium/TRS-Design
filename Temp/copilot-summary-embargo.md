# Embargo Feature — Conversation Summary

**Session date:** March 4, 2026
**Scope:** TRS Design Document v4 + implementation specs

---

## Feature Goal

Support an **embargo** concept where a test's scores are fully ingested but hidden from regular
users until a future release date. Certain privileged users may view embargoed tests early based
on their SSO roles.

---

## Design Decisions (chronological)

### 1. Embargo Granularity
Embargo is applied at the **`test_group_id` level**, not per-test-key or per-student.

### 2. Visibility Rules
- **Regular user** — cannot see an embargoed test at all (not even aggregates). Test is completely invisible.
- **Embargo-permitted user** — sees all tests, including embargoed ones with a UI indicator.
- **Ingestion** — completely unaffected. Score Processor never reads `embargo_until`. Scores flow through the full pipeline identically regardless of embargo status.

### 3. 404 vs 403
Embargoed test requests return **404 Not Found**, not 403 Forbidden.
Rationale: 403 would confirm the test exists. 404 treats the test as non-existent from a regular user's perspective, preventing information leakage about unreleased assessments.

### 4. No `trs.users` Table
Red Hat SSO is the sole identity provider. TRS does not maintain a local users table.
JWT claims carry `sub`, `tenant_id`, and `roles` (string array). The Lambda authorizer validates
the JWT signature against the RH SSO JWKS endpoint before the request reaches the API Lambda.

### 5. Per-Tenant Embargo Roles Table
Instead of hardcoding a role name (e.g., `EMBARGO_VIEWER`), which would break across tenants
with different SSO role naming conventions, TRS stores the permitted role names in a config table:

```sql
CREATE TABLE trs.embargo_roles (
    tenant_id  TEXT  NOT NULL,
    role_name  TEXT  NOT NULL,  -- must exactly match a value in the JWT 'roles' array claim
    PRIMARY KEY (tenant_id, role_name)
);
```

Role names are managed via direct DB update or an admin API. No Lambda redeployment needed.

---

## Schema Changes

### `trs.test_configs` — new column
```sql
embargo_until   TIMESTAMPTZ NULL DEFAULT NULL
```
- `NULL` → not embargoed
- Future timestamp → embargoed; auto-lifts when clock passes it
- Past timestamp → embargo already lifted

Two new indexes added:
- `idx_test_configs_lookup` on `(tenant_id, test_group_id)` — API pre-check
- `idx_test_configs_embargo` on `(tenant_id, embargo_until) WHERE embargo_until IS NOT NULL` — ops tooling

### `trs.embargo_roles` — new table
Per-tenant mapping of SSO role names that are permitted to view embargoed tests.

---

## API Changes

### Embargo Pre-Check (all 4 aggregate paths)
Runs **before** the three-tier fallback chain on every request:
1. Query `test_configs` for `embargo_until` — cached in-process 60 s
2. Query `embargo_roles` for tenant's permitted role names — cached in-process 5 min
3. If embargoed AND caller's JWT roles ∩ tenant embargo_roles = empty → return 404

### Test Discovery Endpoint (new)
`GET /v1/{tenantId}/tests?schoolYear={Y}`

The **only** place callers learn which `test_group_id`s exist. Filters embargoed entries for
non-permitted callers so they never acquire a `test_group_id` to probe.
- Regular caller → excludes rows where `embargo_until > NOW()`
- Embargo-permitted caller → includes all rows + `embargo_until` field + `isEmbargoed` flag

---

## Implementation Files

| File | Contents |
|------|----------|
| [TRS_Design_Document_v4.md](../TRS_Design_Document_v4.md) | §6.3 DDL, §9.1 fallback diagram, §9.2 routing table, §9.3 C# school aggregate, §9.4 health check, §9.5 EmbargoService summary, §11.2 cache invalidation, §2.2 and §18 auth rows |
| [implementation/EmbargoService.md](../implementation/EmbargoService.md) | Full C# `EmbargoService` class, repository SQL queries, test discovery SQL (Steps 1/2a/2b), response shape, cache invalidation notes |
| [implementation/API_Lambda.md](../implementation/API_Lambda.md) | Auth header, per-path embargo pre-check steps (Roster/School/District/State), test discovery endpoint section |

---

## Outstanding Items

- `implementation/API_Lambda.md` — the 4 per-path embargo pre-check steps and the test
  discovery section were added when `EMBARGO_VIEWER` was still hardcoded. These references
  should be updated to reflect the `trs.embargo_roles` table-based approach (two-part pre-check
  with 5-min roles cache).
