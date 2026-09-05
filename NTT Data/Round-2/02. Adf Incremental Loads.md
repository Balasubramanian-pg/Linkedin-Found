# Handling Incremental Loads (Delta Extraction) in Azure Data Factory

## Question
How does Azure Data Factory (ADF) handle incremental loads (CDC / Watermarking)?

---

## 1. Core Incremental Load Strategies in ADF

Azure Data Factory supports several native approaches to ingest only new and modified records:

```
+-------------------------------------------------------------------------+
|                  ADF Incremental Load Patterns                          |
+-------------------+--------------------+--------------------------------+
| Watermark Pattern | Change Tracking/CDC| Partition / Time-slice Pattern |
| (High-Water Mark) | (Native Source CDC)| (File System / Folder Path)    |
+-------------------+--------------------+--------------------------------+
```

---

## 2. Approach 1: Watermark-Based Incremental Extraction (Most Common)

This approach tracks a monotonically increasing column (e.g., `LastModifiedDate` or auto-incrementing `ID`) using a dedicated metadata control/watermark table.

### Workflow Architecture:
```
[1. Lookup Old Watermark] ---> [2. Lookup Current Max Source Watermark]
                                              |
                                              v
[4. Update Control Table] <--- [3. Copy Data (WHERE ModifiedDate > Old AND <= New)]
```

### Step-by-Step Implementation:
1. **Create Watermark Control Table in Azure SQL:**
   ```sql
   CREATE TABLE WatermarkTable (
       TableName VARCHAR(100) PRIMARY KEY,
       WatermarkValue DATETIME2
   );
   INSERT INTO WatermarkTable VALUES ('CustomerOrders', '2026-01-01 00:00:00');
   ```

2. **ADF Pipeline Activities:**
   - **Activity 1 (Lookup - Old Watermark):** Reads stored `WatermarkValue` from `WatermarkTable`.
   - **Activity 2 (Lookup - New Watermark):** Executes `SELECT MAX(LastModifiedDate) AS NewWatermark FROM SourceDB.CustomerOrders;`.
   - **Activity 3 (Copy Data Activity):** Parameterizes the source SQL query:
     ```sql
     SELECT * 
     FROM CustomerOrders 
     WHERE LastModifiedDate > '@{activity('LookupOldWatermark').output.firstRow.WatermarkValue}'
       AND LastModifiedDate <= '@{activity('LookupNewWatermark').output.firstRow.NewWatermarkValue}'
     ```
   - **Activity 4 (Stored Procedure Activity):** Updates the `WatermarkTable` with `NewWatermarkValue` upon successful copy.

---

## 3. Approach 2: Azure SQL / Database Native Change Data Capture (CDC)

When physical deletes must be captured or source tables do not have reliable timestamp columns, native CDC is preferred.

- **Source:** Enable CDC on the database (`EXEC sys.sp_cdc_enable_table ...`).
- **ADF Native CDC / Change Tracking Feature:**
  - ADF's **Copy Activity** has built-in support for **Change Data Capture** on Azure SQL DB / SQL Server.
  - When enabled, ADF automatically manages checkpoints and LSN (Log Sequence Numbers) without requiring custom watermark tables.
  - Captures `INSERT`, `UPDATE`, and `DELETE` operations with metadata columns (`__$operation`).

---

## 4. Approach 3: Binary/File-Based Incremental Ingestion (ADLS Gen2 / Blob)

For file storage sources (e.g., files dumped periodically by external systems):

1. **Filter by `LastModifiedDate`:**
   - In the ADF Copy Activity Source tab, configure `Filter by last modified` using pipeline system variables:
     - **Start Time:** `@pipeline().parameters.windowStart`
     - **End Time:** `@pipeline().parameters.windowEnd`
2. **Dynamic File/Folder Partitioning:**
   - Ingest from parameter-driven paths: `raw/sales/@{formatDateTime(utcnow(), 'yyyy')}/@{formatDateTime(utcnow(), 'MM')}/`.

---

## 5. Summary Comparison

| Method | Captures Deletes? | Requires Schema Change? | Best For |
| :--- | :--- | :--- | :--- |
| **Watermark Column** | No (soft-deletes only if tracked) | Requires modified timestamp column | Generic databases, APIs, legacy sources |
| **Native CDC** | **Yes** | Requires CDC enabled on source | High-fidelity transaction auditing |
| **File Last Modified** | N/A | No | ADLS Gen2, Amazon S3, SFTP file drops |
