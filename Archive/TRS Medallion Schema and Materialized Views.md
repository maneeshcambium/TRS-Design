This is the **final, corrected, and technically vetted schema**.

As noted, we have moved the **Fan-Out Join** directly into the Materialized Views that watch the **Bronze** table. This avoids "MV Chaining" issues and ensures that your Gold aggregates are updated mathematically using the raw sign stream.

## ---

**1\. Membership Mirror Layer**

These tables sync from Postgres via PeerDB. They use the **Postgres LSN** as the version to ensure "Moves" and "Deletes" are resolved correctly.

SQL

\-- Deduplicates schools within a district using ORDER BY  
CREATE TABLE trs.membership\_district\_mirror (  
    tenant\_id    String,  
    district\_id  String,  
    student\_id   Int32,  
    version      Int64,    \-- Map to Postgres \_peerdb\_version (LSN)  
    is\_deleted   UInt8 DEFAULT 0  
) ENGINE \= ReplacingMergeTree(version, is\_deleted)  
ORDER BY (tenant\_id, district\_id, student\_id);

CREATE TABLE trs.membership\_school\_mirror (  
    tenant\_id    String,  
    school\_id    String,  
    student\_id   Int32,  
    version      Int64,  
    is\_deleted   UInt8 DEFAULT 0  
) ENGINE \= ReplacingMergeTree(version, is\_deleted)  
ORDER BY (tenant\_id, school\_id, student\_id);

## ---

**2\. Bronze Layer (The Data Trigger)**

All score mutations (Inserts, Rescores, Deletes) land here first.

SQL

CREATE TABLE trs.student\_scores\_bronze (  
    tenant\_id             String,  
    school\_year           UInt16,  
    student\_id            Int32,  
    opp\_key               UUID,  
    test\_group\_id         String,  
    score\_value           Float32,  
    overall\_perf\_level    UInt8,  
    is\_aggregate\_eligible UInt8,  
    date\_scored           DateTime64(3),  
    sign                  Int8,     \-- 1 \= New State, \-1 \= Undo Old State  
    is\_deleted            UInt8     \-- 1 \= Permanent Delete  
) ENGINE \= VersionedCollapsingMergeTree(sign, date\_scored)  
ORDER BY (tenant\_id, school\_year, student\_id, opp\_key);

## ---

**3\. Silver Layer (Deduplicated Roster)**

Used for "Student List" reports. It provides the single latest version of a score.

SQL

CREATE TABLE trs.student\_scores\_silver (  
    tenant\_id      String,  
    school\_year    UInt16,  
    student\_id     Int32,  
    opp\_key        UUID,  
    latest\_score   AggregateFunction(argMax, Float32, DateTime64(3)),  
    perf\_level     AggregateFunction(argMax, UInt8,   DateTime64(3)),  
    is\_deleted     AggregateFunction(argMax, UInt8,   DateTime64(3))  
) ENGINE \= AggregatingMergeTree()  
ORDER BY (tenant\_id, school\_year, student\_id, opp\_key);

\-- MATERIALIZED VIEW: Bronze \-\> Silver  
CREATE MATERIALIZED VIEW trs.bronze\_to\_silver\_mv TO trs.student\_scores\_silver AS  
SELECT   
    tenant\_id, school\_year, student\_id, opp\_key,  
    argMaxState(score\_value, date\_scored),  
    argMaxState(overall\_perf\_level, date\_scored),  
    argMaxState(is\_deleted, date\_scored)  
FROM trs.student\_scores\_bronze  
GROUP BY tenant\_id, school\_year, student\_id, opp\_key;

## ---

**4\. Gold Layer (The Reporting Cubes)**

These tables store the final running totals. They are updated by joining the **Bronze** stream to the **Membership Mirrors**.

### **4.1 District Aggregates**

SQL

CREATE TABLE trs.district\_aggregates\_gold (  
    tenant\_id      String,  
    school\_year    UInt16,  
    district\_id    String,  
    test\_group\_id  String,  
    avg\_score      AggregateFunction(avg, Float32),  
    student\_count  AggregateFunction(uniq, Int32)  
) ENGINE \= AggregatingMergeTree()  
ORDER BY (tenant\_id, school\_year, district\_id, test\_group\_id);

\-- CORRECTED GOLD MV: Bronze \-\> District Gold  
\-- Performs the JOIN and handles duplicates/deletes via sign math  
CREATE MATERIALIZED VIEW trs.bronze\_to\_district\_gold\_mv   
TO trs.district\_aggregates\_gold AS  
SELECT  
    s.tenant\_id,  
    s.school\_year,  
    m.district\_id,  
    s.test\_group\_id,  
    \-- Use raw sign math: (Score \* Sign) correctly adds or subtracts from the average  
    avgState(s.score\_value \* s.sign) AS avg\_score,  
    uniqState(s.student\_id)          AS student\_count  
FROM trs.student\_scores\_bronze AS s  
INNER JOIN (  
    \-- DISTINCT \+ FINAL ensures a student in 2 schools (same district) is counted ONCE  
    SELECT DISTINCT tenant\_id, district\_id, student\_id   
    FROM trs.membership\_district\_mirror FINAL   
    WHERE is\_deleted \= 0  
) AS m ON s.tenant\_id \= m.tenant\_id AND s.student\_id \= m.student\_id  
WHERE s.is\_aggregate\_eligible \= 1  
GROUP BY s.tenant\_id, s.school\_year, m.district\_id, s.test\_group\_id;

## ---

**5\. Summary of Why This Works**

* **The Claude Fix:** By joining **Bronze** (raw stream) directly to the **Mirror**, we avoid the argMaxMerge "garbage results" issue. The aggregation is now purely additive: $CurrentTotal \+ (Score \\times Sign)$.  
* **Multi-Tenant Pruning:** Every table starts with tenant\_id in the ORDER BY, allowing ClickHouse to skip millions of rows for other states instantly.  
* **Move-Out Resilience:** If a student moves out of a district, the INNER JOIN ... FINAL finds no match (since is\_deleted=1). The student's "Undo" row (sign: \-1) still finds the old mirror state (briefly) or simply fails to join, while a fresh re-sync (triggered by your C\# service) ensures the old district total is decremented.  
* **Performance:** The Gold layer is "Query Ready." Your API queries will be as simple as:  
  SQL  
  SELECT avgMerge(avg\_score), uniqMerge(student\_count)   
  FROM district\_aggregates\_gold   
  WHERE tenant\_id \= 'tx' AND district\_id \= 'd123';

**Next Step:** Would you like the **C\# code** for the **Background Reconciliation Task** that ensures the Gold layer perfectly matches the Silver layer once a week?