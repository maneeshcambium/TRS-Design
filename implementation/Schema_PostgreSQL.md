# TRS — Aurora PostgreSQL Schema (DDL Reference)

Full DDL for all Aurora PostgreSQL tables, indexes, and materialized views.
See [TRS_Design_Document_v4.md](../TRS_Design_Document_v4.md) §6 for design rationale.

---

## Score Tables

### `trs.student_opportunities`

Two-level composite partitioning: Level 1 = `LIST(tenant_id)`, Level 2 = `LIST(school_year)`.

```sql
-- Application code ALWAYS queries the parent table only.
-- Postgres routes transparently to the correct leaf partition at plan time.
-- Never reference partition names (e.g. student_opportunities_tx_2026) in application code.

CREATE TABLE trs.student_opportunities (
    tenant_id              TEXT         NOT NULL,   -- L1 partition key
    school_year            SMALLINT     NOT NULL,   -- L2 partition key
    opp_key                UUID         NOT NULL,

    test_group_id          TEXT         NOT NULL,
    test_key               TEXT         NOT NULL,
    test_event             TEXT,
    student_id             INTEGER      NOT NULL,
    opp_status             TEXT         NOT NULL,
    condition_code         TEXT,
    tested_date            DATE,
    date_scored            TIMESTAMPTZ  NOT NULL,   -- version key; must be millisecond precision

    is_aggregate_eligible  BOOLEAN      NOT NULL DEFAULT FALSE,

    overall_scale_score    REAL,
    overall_raw_score      INTEGER,
    overall_lexile_score   INTEGER,
    overall_quantile_score INTEGER,
    overall_perf_level     SMALLINT,
    overall_standard_error REAL,

    ingested_at            TIMESTAMPTZ  NOT NULL DEFAULT NOW(),
    raw_s3_key             TEXT,

    -- PK must include both partition keys for unique constraint on partitioned table
    PRIMARY KEY (tenant_id, school_year, opp_key)
)
PARTITION BY LIST (tenant_id);

-- Per-tenant partition (example: Texas)
CREATE TABLE trs.student_opportunities_tx
    PARTITION OF trs.student_opportunities
    FOR VALUES IN ('tx')
    PARTITION BY LIST (school_year);

CREATE TABLE trs.student_opportunities_tx_2026
    PARTITION OF trs.student_opportunities_tx
    FOR VALUES IN (2026);
```

**Indexes:**
```sql
-- Primary query pattern: tenant + year + test + student_id list (roster queries)
CREATE INDEX idx_so_roster
    ON trs.student_opportunities (tenant_id, school_year, test_group_id, student_id)
    WHERE is_aggregate_eligible = TRUE;

-- Full per-student score list (includes ineligible)
CREATE INDEX idx_so_student
    ON trs.student_opportunities (tenant_id, school_year, test_group_id, student_id);
```

**Rescore UPSERT:**
```sql
INSERT INTO trs.student_opportunities
    (tenant_id, school_year, opp_key, test_group_id, test_key, test_event,
     student_id, opp_status, condition_code, tested_date, date_scored,
     is_aggregate_eligible, overall_scale_score, overall_raw_score,
     overall_perf_level, overall_standard_error, raw_s3_key)
VALUES ($1, $2, $3, $4, $5, $6, $7, $8, $9, $10, $11, $12, $13, $14, $15, $16, $17)
ON CONFLICT (tenant_id, school_year, opp_key) DO UPDATE SET
    date_scored            = EXCLUDED.date_scored,
    opp_status             = EXCLUDED.opp_status,
    condition_code         = EXCLUDED.condition_code,
    tested_date            = EXCLUDED.tested_date,
    is_aggregate_eligible  = EXCLUDED.is_aggregate_eligible,
    overall_scale_score    = EXCLUDED.overall_scale_score,
    overall_raw_score      = EXCLUDED.overall_raw_score,
    overall_perf_level     = EXCLUDED.overall_perf_level,
    overall_standard_error = EXCLUDED.overall_standard_error,
    ingested_at            = NOW()
WHERE EXCLUDED.date_scored > trs.student_opportunities.date_scored;
-- WHERE clause: stale resends are silently ignored; zero PG rows affected → no WAL UPDATE event → no CDC
```

**PeerDB configuration:** Run before enabling CDC replication:
```sql
ALTER TABLE trs.student_opportunities REPLICA IDENTITY FULL;
```

---

### `trs.student_component_scores`

```sql
CREATE TABLE trs.student_component_scores (
    tenant_id             TEXT         NOT NULL,
    school_year           SMALLINT     NOT NULL,
    opp_key               UUID         NOT NULL,
    component_type        TEXT         NOT NULL,  -- STANDARD | RC | WRITING_DIM
    component_id          TEXT         NOT NULL,

    test_group_id         TEXT         NOT NULL,
    student_id            INTEGER      NOT NULL,
    perf_level            SMALLINT,
    scale_score           REAL,
    standard_error        REAL,
    condition_code        TEXT,
    is_aggregate_eligible BOOLEAN      NOT NULL DEFAULT FALSE,
    date_scored           TIMESTAMPTZ  NOT NULL,

    PRIMARY KEY (tenant_id, school_year, opp_key, component_type, component_id)
)
PARTITION BY LIST (tenant_id);

CREATE INDEX idx_scs_query
    ON trs.student_component_scores
       (tenant_id, school_year, test_group_id, student_id, component_type)
    WHERE is_aggregate_eligible = TRUE;
```

```sql
ALTER TABLE trs.student_component_scores REPLICA IDENTITY FULL;
```

---

## Config & Reference Tables

> **Schema note — TestKey vs TestAlias relationship:**
> A `testalias_id` groups one or more delivery-variant `test_key`s (e.g., online, paper, braille
> all share the same alias). Embargo, metadata, and test attributes belong to the alias level and
> are stored once in `test_aliases`, not duplicated per key. Multiple aliases can be sequenced
> into a multi-opportunity group via `test_alias_groups` (opp1 → seq 1, opp2 → seq 2).

---

### `trs.test_aliases`

One row per `testalias_id`. Holds all config-level attributes: test family, embargo, release
state, and metadata. Multiple `test_key`s in `trs.test_keys` reference a single alias row.

```sql
CREATE TABLE trs.test_aliases (
    tenant_id           TEXT        NOT NULL,
    testalias_id        TEXT        NOT NULL,   -- e.g. checkpoint1-grade3-ela-2026-opp1

    test_family         TEXT        NOT NULL,   -- e.g. checkpoint1
    subject             TEXT        NOT NULL,   -- e.g. ELA
    grade               TEXT        NOT NULL,   -- e.g. grade3
    school_year         SMALLINT    NOT NULL,   -- e.g. 2026
    testalias_name      TEXT        NOT NULL,   -- display name ({TestFamily}-{Subject}-{Grade}-{SchoolYear})

    -- Embargo / release date: these are the same concept.
    -- NULL             = never embargoed; test results are visible to all roles.
    -- Future timestamp = embargoed (unreleased); auto-lifts when clock passes it.
    -- Past timestamp   = embargo has lifted; results are released.
    -- Derivations: is_released ≡ (embargo_until IS NULL OR embargo_until <= NOW())
    --              release_date ≡ embargo_until::DATE
    embargo_until       TIMESTAMPTZ NULL DEFAULT NULL,

    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (tenant_id, testalias_id)
);

-- Efficient lookup for the API embargo pre-check
CREATE INDEX idx_test_aliases_embargo_lookup
    ON trs.test_aliases (tenant_id, testalias_id)
    WHERE embargo_until IS NOT NULL;

-- Efficient scan to find all currently embargoed tests (used by ops tooling)
CREATE INDEX idx_test_aliases_embargo
    ON trs.test_aliases (tenant_id, embargo_until)
    WHERE embargo_until IS NOT NULL;
```

---

### `trs.test_keys`

One row per `test_key` (specific delivery variant: online, paper, braille, etc.).
Points to its parent alias. The API and CDC pipeline resolve a `test_key` to its
`testalias_id` via this table.

```sql
CREATE TABLE trs.test_keys (
    tenant_id       TEXT        NOT NULL,
    test_key        TEXT        NOT NULL,   -- e.g. checkpoint1-grade3-ela-2026-online-opp1
    testalias_id    TEXT        NOT NULL,   -- FK → test_aliases

    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),

    PRIMARY KEY (tenant_id, test_key),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);

-- Reverse lookup: find all test_keys belonging to an alias
CREATE INDEX idx_test_keys_alias
    ON trs.test_keys (tenant_id, testalias_id);
```

---

### `trs.test_alias_groups`

Orders aliases within a multi-opportunity group. Each row assigns a `testalias_id` a
sequence number within a named group (e.g., `checkpoint1-grade3-ela-2026` has
`opp1` at sequence 1 and `opp2` at sequence 2).

```sql
CREATE TABLE trs.test_alias_groups (
    tenant_id               TEXT        NOT NULL,
    testalias_group_name    TEXT        NOT NULL,   -- e.g. checkpoint1-grade3-ela-2026
    testalias_id            TEXT        NOT NULL,   -- FK → test_aliases
    sequence                SMALLINT    NOT NULL,   -- 1, 2, … (display / report order)
    sequence_label          TEXT        NOT NULL,   -- e.g. opp1, opp2

    PRIMARY KEY (tenant_id, testalias_group_name, sequence),
    UNIQUE      (tenant_id, testalias_id),          -- each alias belongs to at most one group position
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);

-- Lookup all aliases in a group, ordered for display
CREATE INDEX idx_test_alias_groups_alias
    ON trs.test_alias_groups (tenant_id, testalias_group_name, sequence);
```

---

### `trs.test_alias_standards`

One row per standard aligned to a `testalias_id`. The `standard_id` matches the
`component_id` stored in `trs.student_component_scores` when `component_type = 'STANDARD'`.

```sql
CREATE TABLE trs.test_alias_standards (
    tenant_id       TEXT    NOT NULL,
    testalias_id    TEXT    NOT NULL,   -- FK → test_aliases
    standard_id     TEXT    NOT NULL,   -- e.g. CCSS.ELA-LITERACY.RL.3.1
    description     TEXT    NOT NULL,

    PRIMARY KEY (tenant_id, testalias_id, standard_id),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);
```

---

### `trs.standard_performance_levels`

One row per performance level per standard. Descriptions are standard-specific even when
level titles (Below Basic / Basic / Proficient / Advanced) repeat across standards.

```sql
CREATE TABLE trs.standard_performance_levels (
    tenant_id       TEXT        NOT NULL,
    testalias_id    TEXT        NOT NULL,
    standard_id     TEXT        NOT NULL,
    level           SMALLINT    NOT NULL,   -- 1, 2, 3, 4  (matches component_perf_level in scores)
    title           TEXT        NOT NULL,   -- e.g. Below Basic, Basic, Proficient, Advanced
    description     TEXT        NOT NULL,
    min_score       REAL        NOT NULL,   -- inclusive lower bound; REAL to match overall_scale_score
    max_score       REAL        NOT NULL,   -- inclusive upper bound of the scale-score range for this level

    PRIMARY KEY (tenant_id, testalias_id, standard_id, level),
    FOREIGN KEY (tenant_id, testalias_id, standard_id)
        REFERENCES trs.test_alias_standards (tenant_id, testalias_id, standard_id)
);
```

---

### `trs.test_alias_measures`

One row per measure type per alias. Controls which score measures are displayed in reports
and provides labels and descriptions shown to users.

`min_score` / `max_score` are NULL for `performanceLevel` — its score ranges are defined
per-level in `trs.test_alias_perf_levels`.

```sql
CREATE TABLE trs.test_alias_measures (
    tenant_id       TEXT        NOT NULL,
    testalias_id    TEXT        NOT NULL,   -- FK → test_aliases
    measure_type    TEXT        NOT NULL,   -- scaleScore | rawScore | lexileScore | quantileScore
                                            -- | percentCorrect | percentile | performanceLevel
    show            BOOLEAN     NOT NULL DEFAULT TRUE,
    label           TEXT        NOT NULL,
    description     TEXT        NOT NULL,
    min_score       REAL        NULL,       -- NULL for performanceLevel; REAL to match overall_scale_score
    max_score       REAL        NULL,       -- NULL for performanceLevel

    PRIMARY KEY (tenant_id, testalias_id, measure_type),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);
```

---

### `trs.test_alias_perf_levels`

Overall test-level performance levels. Parallel to `trs.standard_performance_levels` but
keyed to the alias only — these describe the test's overall score bands, not per-standard.
`level` matches `overall_perf_level` in `trs.student_opportunities`.

```sql
CREATE TABLE trs.test_alias_perf_levels (
    tenant_id       TEXT        NOT NULL,
    testalias_id    TEXT        NOT NULL,   -- FK → test_aliases
    level           SMALLINT    NOT NULL,   -- 1, 2, 3, 4  (matches overall_perf_level in student_opportunities)
    title           TEXT        NOT NULL,   -- e.g. Below Basic, Basic, Proficient, Advanced
    description     TEXT        NOT NULL,
    min_score       REAL        NOT NULL,   -- inclusive lower bound; REAL to match overall_scale_score
    max_score       REAL        NOT NULL,   -- inclusive upper bound

    PRIMARY KEY (tenant_id, testalias_id, level),
    FOREIGN KEY (tenant_id, testalias_id)
        REFERENCES trs.test_aliases (tenant_id, testalias_id)
);
```

### `trs.embargo_roles`

```sql
CREATE TABLE trs.embargo_roles (
    tenant_id  TEXT  NOT NULL,
    role_name  TEXT  NOT NULL,  -- must exactly match a value in the JWT 'roles' array claim
    PRIMARY KEY (tenant_id, role_name)
);

-- Example data:
INSERT INTO trs.embargo_roles (tenant_id, role_name) VALUES
    ('tx', 'STATE_ASSESSMENT_DIRECTOR'),
    ('tx', 'TRS_INTERNAL_STAFF');

INSERT INTO trs.embargo_roles (tenant_id, role_name) VALUES
    ('va', 'EMBARGO_REVIEWER');
```

---

## Membership & Relationship Tables

### Normalized RTS Relationship Tables

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

```sql
-- Required for PeerDB CDC membership replication:
ALTER TABLE trs.school_student    REPLICA IDENTITY FULL;
ALTER TABLE trs.district_student  REPLICA IDENTITY FULL;
```

---

## Operational Tables

### `trs.score_ingest_log` — Idempotency Table

```sql
CREATE TABLE trs.score_ingest_log (
    s3_key       TEXT        NOT NULL,
    etag         TEXT        NOT NULL,
    processed_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    PRIMARY KEY (s3_key, etag)
);
-- Cleanup: pg_cron deletes rows older than 14 days nightly
```

### `trs.report_cache` — Pre-Computed Aggregate Store

Cache key format: `"{scope}#{tenant_id}#{scope_id}#{school_year}#{test_group_id}"`
Examples:
- `"school#tx#s-789#2026#cp1-g5-ela"`
- `"district#tx#d-456#2026#cp1-g5-ela"`
- `"state#tx#tx#2026#cp1-g5-ela"`

```sql
CREATE TABLE trs.report_cache (
    cache_key    TEXT        PRIMARY KEY,
    payload      JSONB       NOT NULL,  -- full aggregate result
    computed_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    expires_at   TIMESTAMPTZ NOT NULL,
    computed_by  TEXT        NOT NULL   -- 'clickhouse_gold' | 'postgres_mv' | 'postgres_live' | 'nightly_job'
);

CREATE INDEX idx_report_cache_expiry ON trs.report_cache (expires_at);
-- pg_cron cleanup: DELETE FROM trs.report_cache WHERE expires_at < now();
```

---

## Materialized Views (Fallback Pre-Aggregation)

### `trs.mv_school_overall`

```sql
CREATE MATERIALIZED VIEW trs.mv_school_overall AS
SELECT
    ss.tenant_id,
    ss.school_year,
    ss.test_group_id,
    ss_m.school_id,
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)     AS pl1,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)     AS pl2,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)     AS pl3,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)     AS pl4,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)     AS pl5,
    AVG(ss.overall_scale_score)                           AS avg_scale_score,
    AVG(ss.overall_standard_error)                        AS avg_standard_error,
    NOW()                                                 AS refreshed_at
FROM trs.student_opportunities ss
JOIN trs.school_student ss_m
    ON ss.tenant_id = ss_m.tenant_id AND ss.student_id = ss_m.student_id
WHERE ss.is_aggregate_eligible = TRUE AND ss_m.active = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.test_group_id, ss_m.school_id;

CREATE UNIQUE INDEX uq_mv_school_overall
    ON trs.mv_school_overall (tenant_id, school_year, test_group_id, school_id);
```

### `trs.mv_district_overall`

```sql
CREATE MATERIALIZED VIEW trs.mv_district_overall AS
SELECT
    ss.tenant_id,
    ss.school_year,
    ss.test_group_id,
    ds.district_id,
    COUNT(*)                                               AS students_tested,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 1)     AS pl1,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 2)     AS pl2,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 3)     AS pl3,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 4)     AS pl4,
    COUNT(*) FILTER (WHERE ss.overall_perf_level = 5)     AS pl5,
    AVG(ss.overall_scale_score)                           AS avg_scale_score,
    NOW()                                                 AS refreshed_at
FROM trs.student_opportunities ss
JOIN trs.district_student ds
    ON ss.tenant_id = ds.tenant_id AND ss.student_id = ds.student_id
WHERE ss.is_aggregate_eligible = TRUE AND ds.active = TRUE
GROUP BY ss.tenant_id, ss.school_year, ss.test_group_id, ds.district_id;

CREATE UNIQUE INDEX uq_mv_district_overall
    ON trs.mv_district_overall (tenant_id, school_year, test_group_id, district_id);
```

---

## Operational Prerequisites

### Aurora Configuration

```sql
-- WAL cap: prevents unbounded growth during PeerDB downtime
-- If this cap is hit, the replication slot is invalidated → PeerDB Initial Load required
ALTER SYSTEM SET max_slot_wal_keep_size = '10GB';
SELECT pg_reload_conf();
```

### REPLICA IDENTITY FULL (run once per replicated table before enabling PeerDB)

Without `REPLICA IDENTITY FULL`, PeerDB Before images on UPDATE/DELETE contain only PK columns —
all non-key fields are null, making the Sign-Pair Transformer's undo row useless.

```sql
ALTER TABLE trs.student_opportunities        REPLICA IDENTITY FULL;
ALTER TABLE trs.student_component_scores     REPLICA IDENTITY FULL;
ALTER TABLE trs.school_student               REPLICA IDENTITY FULL;
ALTER TABLE trs.district_student             REPLICA IDENTITY FULL;
```

---

*See [TRS_Design_Document_v4.md](../TRS_Design_Document_v4.md) for full design rationale.*
