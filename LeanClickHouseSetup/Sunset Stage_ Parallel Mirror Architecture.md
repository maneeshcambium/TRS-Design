To handle the transition cleanly, you should use a **Parallel Mirror Architecture**.

In PeerDB (and most CDC tools), a **"Mirror"** is a specific replication job. By creating separate mirrors for each school year, you gain the ability to manage their lifecycles (start, stop, and delete) independently without affecting the active year's data flow.

### **1\. The Multi-Mirror Architecture**

Instead of one massive pipe streaming all data from ScoresDb to a single ClickHouse bucket, you define **Year-Specific Mirrors** that act as filters.

#### **Mirror A: "The Trailing Year" (e.g., 2025\)**

* **Source:** ScoresDb (Postgres)  
* **Filter:** SELECT \* FROM trs.student\_opportunities WHERE school\_year \= 2025  
* **Destination:** ClickHouse Instance **\#1** (The "2025 Cluster")  
* **Role:** Captures late rescores and trailing data for the previous year.

#### **Mirror B: "The Current Year" (e.g., 2026\)**

* **Source:** ScoresDb (Postgres)  
* **Filter:** SELECT \* FROM trs.student\_opportunities WHERE school\_year \= 2026  
* **Destination:** ClickHouse Instance **\#2** (The "2026 Cluster")  
* **Role:** Handles the high-volume real-time ingestion for the active test window.

### ---

**2\. Why Two ClickHouse Instances?**

While you *could* put both years in one ClickHouse server, having **two separate instances** (one per year) is the key to the "Sunset Stage":

1. **Zero-Interference Maintenance:** You can perform the "Final Snapshot" of the 2025 data and then **shut down the entire 2025 ClickHouse EC2 instance**. This instantly stops the billing for that year's analytical compute.  
2. **Clean "Active" Set:** The 2026 ClickHouse instance stays "lean." It doesn't have 5 years of historical indices or data bloating its memory, keeping query performance at sub-30ms.  
3. **Independent Scaling:** During the busy Spring testing window for 2026, you can scale up the 2026 ClickHouse instance without wasting money on the 2025 instance, which is seeing very low traffic.

### ---

**3\. The "Sunset" Process (Step-by-Step)**

When December 2026 arrives and you are ready to retire the 2025 infrastructure:

* **Step 1 (Snapshot):** The **Aggregate Refresh Lambda** performs a final read of the 2025 ClickHouse instance and writes those values into the school\_overall\_aggregates table in Postgres.  
* **Step 2 (Redirect):** Update the year\_registry in **ConfigsDb** to set school\_year 2025 status to Archived. The API now knows to ignore ClickHouse for 2025\.  
* **Step 3 (Disable Mirror):** You log into the PeerDB UI or CLI and **stop Mirror A**. PeerDB releases the replication slot for that filter.  
* **Step 4 (Delete Infrastructure):** You terminate the **ClickHouse Instance \#1** EC2 node. The 2025 data still lives in the Postgres student\_opportunities partition (for rosters) and the snapshot table (for aggregates), but you are no longer paying for the ClickHouse "engine".

### **4\. Visualizing the Overlap**

Think of it like a **Relay Race**:

* **August:** Runner 1 (2025) is still running but slowing down. Runner 2 (2026) starts their lap.  
* **August – December:** Both runners are on the track (Parallel CDC/Parallel ClickHouse).  
* **December:** Runner 1 (2025) hands the "baton" (the final aggregate data) to the permanent Postgres Snapshot Table and leaves the track (Sunset).

### ---

**Summary Table: PeerDB Mirror Strategy**

| Year | Mirror Status | Destination | Strategy |
| :---- | :---- | :---- | :---- |
| **2026** | **Running** | CH Instance 2026 | Real-time CDC. |
| **2025** | **Running (Grace Period)** | CH Instance 2025 | Capture late rescores. |
| **2024** | **Disabled** | None | Data is in Postgres Snapshot Table. |

This architecture ensures that your "Current Year" is always running on a clean, high-performance, and isolated infrastructure.

**Would you like me to update the v5 document to explicitly include this "Parallel Mirror" and "Instance Isolation" logic?**