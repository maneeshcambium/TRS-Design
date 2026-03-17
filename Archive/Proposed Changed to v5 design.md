The student\_component\_scores table contains granular data (standards, reporting categories, writing dimensions) that scales significantly higher than overall scores—up to **1.65 billion rows per year** at Texas scale. Because of this volume, the materialization strategy shifts toward heavy reliance on **ClickHouse for school/district aggregates** while keeping **Postgres for roster-level seeking**.

### **1\. ClickHouse Medallion Layer (Standards & Categories)**

In ClickHouse, component scores follow a similar "Wide Column" Gold pattern to maintain rescore correctness via the Sign-Pair Transformer.

#### **Bronze & Silver Layers**

The CDC pipeline replicates raw component updates into a **Bronze** table. The **Silver** layer (optional) deduplicates events to provide the "latest" version of a specific standard score for a student.

#### **Gold Layer: Standard-Level Aggregates**

Instead of one massive table, use a Gold table partitioned by component\_type (e.g., STANDARD, RC) to keep scans efficient.

SQL

CREATE TABLE trs.component\_aggregates\_gold (  
    tenant\_id       String,  
    school\_year     UInt16,  
    school\_id       String,  
    test\_group\_id   String,  
    component\_type  LowCardinality(String), \-- 'STANDARD' | 'RC' | 'WRITING\_DIM'  
    component\_id    String,  
      
    \-- Superset Measures  
    avg\_scale\_score AggregateFunction(avg, Float32),  
    pl\_1\_count      AggregateFunction(sum, Int64),  
    \-- ... pl\_2 to pl\_5 (standards usually have fewer PLs) ...  
    student\_count   AggregateFunction(uniq, Int32)  
)  
ENGINE \= AggregatingMergeTree()  
ORDER BY (tenant\_id, school\_year, school\_id, test\_group\_id, component\_type, component\_id);

### ---

**2\. Postgres Materialized View (Tier 3 Fallback)**

Given the 1.6 billion row volume, building a single Materialized View for all standards across all years is **not feasible** in Postgres. To remain pragmatic and avoid over-engineering, follow these constraints:

* **Current Year Only:** Only create a Materialized View for the active school year's components.  
* **Partition Pruning:** The MV must explicitly join student\_component\_scores with the student\_opportunities parent table using tenant\_id and school\_year to force pruning.  
* **Scoped by Type:** Consider separate MVs for STANDARD vs. RC if refresh times exceed the 15-minute target.

SQL

CREATE MATERIALIZED VIEW trs.mv\_school\_standards\_current AS  
SELECT  
    scs.tenant\_id,  
    scs.school\_year,  
    scs.test\_group\_id,  
    ss\_m.school\_id,  
    scs.component\_id,  
    COUNT(\*) AS students\_tested,  
    AVG(scs.scale\_score) AS avg\_scale\_score,  
    COUNT(\*) FILTER (WHERE scs.perf\_level \= 1) AS pl\_1,  
    \-- ...  
FROM trs.student\_component\_scores scs  
JOIN trs.school\_student ss\_m   
    ON scs.tenant\_id \= ss\_m.tenant\_id AND scs.student\_id \= ss\_m.student\_id  
WHERE scs.school\_year \= 2026 \-- Partition Pruning  
  AND scs.component\_type \= 'STANDARD' \-- Scoped for performance  
  AND scs.is\_aggregate\_eligible \= TRUE  
GROUP BY scs.tenant\_id, scs.school\_year, scs.test\_group\_id, ss\_m.school\_id, scs.component\_id;

### ---

**3\. Strategic Data Flow for Component Scores**

Component aggregates follow the **Three-Tier Fallback** but with stricter "Live Query" guardrails:

1. **Tier 1: report\_cache** — Stores the full JSON of all standards/categories for a school.  
2. **Tier 2: ClickHouse Gold** — The primary source for all school/district standard aggregates.  
3. **Tier 3: Postgres**  
   * **Current Year:** Hits the mv\_school\_standards\_current.  
   * **Historical:** **No Archive Table.** If ClickHouse is down and historical standard data is needed, compute it live from student\_component\_scores and store it in report\_cache with a long TTL.  
   * **Guardrail:** Because raw component scans are expensive, Tier 3b (Live) should only be allowed for **School Scope** (roster size). District-wide live standard aggregates on Postgres should be disabled or limited to prevent Aurora exhaustion.

### **Summary of Materialization**

| Scope | ClickHouse Strategy | Postgres Strategy |
| :---- | :---- | :---- |
| **Roster** | Direct Seek (Silver/Bronze) | **Primary Path:** Index seek on idx\_scs\_query. |
| **School** | Gold AggregatingMergeTree | Current-Year Materialized View. |
| **District** | Gold AggregatingMergeTree | **Disabled** or Live-to-Cache only (too large for MV). |

| Database | Responsibility | Key Tables |
| :---- | :---- | :---- |
| **ConfigsDb** | **Metadata, tenant registry, and test definitions.** | **tenants, test\_configs, embargo\_roles.** |
| **ScoresDb** | **Transactional score data, membership mirrors, and caching.** | **student\_opportunities, student\_component\_scores, school\_student, district\_student, report\_cache.** |

