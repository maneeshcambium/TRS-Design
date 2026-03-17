# EmbargoService — Implementation Reference

Implements the in-process embargo gate called by the API Lambda before every aggregate request.

---

## Overview

`EmbargoService` performs two lookups from the TRS database and caches both results in-process:

| Value | Source table | Cache TTL |
|-------|-------------|-----------|
| `embargo_until` for the requested `test_group_id` | `trs.test_configs` | 60 s per `(tenant_id, test_group_id)` |
| Permitted embargo role names for the tenant | `trs.embargo_roles` | 5 min per `tenant_id` |

Role names in `trs.embargo_roles` must exactly match values in the caller's JWT `roles` array
claim (case-insensitive comparison). If at least one caller role appears in the tenant's
`embargo_roles` set, the caller may see embargoed tests; otherwise `AssertVisibleAsync` throws
`EmbargoException`, which the Lambda maps to **HTTP 404 Not Found**.

404 is used instead of 403 to avoid confirming to a regular user that the test exists but is
restricted. From a non-permitted caller's perspective the test simply does not exist.

---

## C# Implementation

```csharp
/// <summary>
/// Checks embargo status for a test_group_id and throws if the caller cannot see it.
///
/// Two values are read from the TRS database and cached in-process:
///   1. embargo_until  (from test_configs)  — cached 60 s per test_group_id
///   2. embargo_roles  (from embargo_roles)  — cached 5 min per tenant
///      (longer TTL: role config changes rarely; ops-driven reconfig tolerable within 5 min)
///
/// Role resolution: callerRoles is extracted directly from the validated JWT claims.
/// RH SSO is the sole authority for role assignment; TRS only stores which role names
/// are permitted to view embargoed tests, not what roles a given user holds.
/// </summary>
public class EmbargoService
{
    private readonly ITestConfigRepository _repo;
    private readonly IEmbargoRolesRepository _rolesRepo;
    private readonly IMemoryCache _cache;
    private const int EmbargoStatusTtlSeconds  = 60;
    private const int EmbargoRolesTtlSeconds   = 300;  // 5 minutes

    /// <summary>
    /// Throws <see cref="EmbargoException"/> (mapped to HTTP 404) when the test is
    /// currently embargoed and none of the caller's JWT roles appear in the tenant's
    /// embargo_roles configuration.
    /// callerRoles must be sourced from validated JWT claims, not from any local store.
    /// </summary>
    public async Task AssertVisibleAsync(
        string tenantId, string testGroupId, IReadOnlyList<string> callerRoles)
    {
        // 1. Check whether the test is currently embargoed (60 s cache)
        var statusKey = $"embargo:status:{tenantId}:{testGroupId}";
        if (!_cache.TryGetValue(statusKey, out DateTimeOffset? embargoUntil))
        {
            embargoUntil = await _repo.GetEmbargoUntilAsync(tenantId, testGroupId);
            _cache.Set(statusKey, embargoUntil, TimeSpan.FromSeconds(EmbargoStatusTtlSeconds));
        }

        if (!embargoUntil.HasValue || embargoUntil.Value <= DateTimeOffset.UtcNow)
            return;  // not embargoed — fast path

        // 2. Test is embargoed — load the tenant's permitted embargo roles (5 min cache)
        var rolesKey = $"embargo:roles:{tenantId}";
        if (!_cache.TryGetValue(rolesKey, out HashSet<string>? allowedRoles))
        {
            var rows = await _rolesRepo.GetEmbargoRolesAsync(tenantId);
            allowedRoles = new HashSet<string>(rows, StringComparer.OrdinalIgnoreCase);
            _cache.Set(rolesKey, allowedRoles, TimeSpan.FromSeconds(EmbargoRolesTtlSeconds));
        }

        // 3. Caller is permitted if any JWT role matches the tenant's embargo roles
        if (!callerRoles.Any(r => allowedRoles!.Contains(r)))
            throw new EmbargoException(testGroupId);
    }
}
```

---

## Repository Queries

### Embargo status for a specific `test_group_id`

Called by `ITestConfigRepository.GetEmbargoUntilAsync(tenantId, testGroupId)`.

```sql
SELECT embargo_until
FROM trs.test_configs
WHERE tenant_id     = $1
  AND test_group_id = $2
LIMIT 1;
```

Returns `NULL` if the test is not embargoed or not found. The cache stores the raw nullable
value — a `NULL` result is also cached for 60 s to prevent cache-miss hammering on valid
non-embargoed tests.

### Permitted embargo role names for a tenant

Called by `IEmbargoRolesRepository.GetEmbargoRolesAsync(tenantId)`.

```sql
SELECT role_name
FROM trs.embargo_roles
WHERE tenant_id = $1;
```

Returns zero rows if the tenant has no embargo-permitted roles configured (all callers treated
as regular users for that tenant).

---

## Test Discovery Endpoint

`GET /v1/{tenantId}/tests?schoolYear={Y}` is the only place a caller learns which
`test_group_id`s exist. Embargo filtering here prevents regular users from ever acquiring a
`test_group_id` to probe via the aggregate endpoints.

### Step 1 — Resolve caller's embargo permission

```sql
-- $3 = caller's JWT roles array (TEXT[])
SELECT EXISTS (
    SELECT 1 FROM trs.embargo_roles
    WHERE tenant_id = $1
      AND role_name = ANY($3)
) AS can_view_embargo;
```

### Step 2a — Regular caller (`can_view_embargo = false`)

```sql
SELECT DISTINCT test_group_id, test_family, subject, grade
FROM trs.test_configs
WHERE tenant_id   = $1
  AND school_year = $2
  AND (embargo_until IS NULL OR embargo_until <= NOW())
ORDER BY test_group_id;
```

### Step 2b — Embargo-permitted caller (`can_view_embargo = true`)

```sql
SELECT DISTINCT test_group_id, test_family, subject, grade, embargo_until
FROM trs.test_configs
WHERE tenant_id   = $1
  AND school_year = $2
ORDER BY test_group_id;
-- embargo_until included so the UI can label which tests are still under embargo
```

**Response shape for embargo-permitted callers** — includes `isEmbargoed` per entry so the UI
can display a visual indicator on unreleased tests:

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

## Cache Invalidation Notes

| Event | Cache impact |
|-------|-------------|
| Embargo applied to a live test | `embargo:status:{tenant}:{testGroupId}` must be manually evicted (or wait up to 60 s) |
| Embargo lifted (`embargo_until` set to past/NULL) | Auto-expires within 60 s — no manual action needed |
| `embargo_roles` row added or removed | Auto-expires within 5 min — no Lambda redeployment needed |
