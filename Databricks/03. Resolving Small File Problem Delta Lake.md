# Scenario 03: Resolving the Small-File Problem in Delta Lake

## Problem Statement
High-frequency streaming writes or small-batch append pipelines have generated millions of tiny (few KB / MB) Parquet files in a Delta table. Query read times and metadata listing operations have degraded significantly. How do you resolve this?

---

## 1. Why Small Files Degrade Lakehouse Performance

1. **Storage Metadata Overhead:** Distributed query engines (Spark, Trino, Photon) spend significant time listing file metadata from ADLS Gen2 instead of processing actual data.
2. **Task Scheduling Bottleneck:** Spark creates 1 task per file/block. Reading 1,000,000 tiny files spawns 1,000,000 JVM tasks, exhausting CPU cycles on driver scheduling and garbage collection.
3. **Inefficient Columnar Compression:** Parquet dictionary encoding and Snappy compression achieve poor compression ratios on small byte chunks.

---

## 2. Immediate Remediation: File Compaction (`OPTIMIZE` & `Z-ORDER`)

The `OPTIMIZE` command compacts small files into optimal **~1 GB** Parquet files:

```sql
-- 1. Compact entire table or specific date partition
OPTIMIZE silver_transactions
WHERE transaction_date >= '2026-08-01';

-- 2. Multi-dimensional clustering (Z-ORDER) on high-cardinality filter columns
OPTIMIZE silver_transactions
ZORDER BY (customer_id, merchant_id);
```

### Automated Background Maintenance (Vacuuming Old Files):
Compaction creates new consolidated Parquet files without immediately deleting the old small files (to preserve Delta Time Travel). Schedule `VACUUM` to physically delete expired tombstoned files:

```sql
-- Prune files older than default 7 days (168 hours)
VACUUM silver_transactions RETAIN 168 HOURS;
```

---

## 3. Preventive Architecture: Proactive Compaction Settings

Configure table properties and Spark session configs so small files are prevented during active writes:

### A. Auto-Compaction & Optimized Writes
```sql
ALTER TABLE silver_transactions SET TBLPROPERTIES (
   'delta.autoOptimize.optimizeWrite' = 'true',
   'delta.autoOptimize.autoCompact' = 'true'
);
```
- **Optimized Writes:** Spark dynamically coalesces partitions to aim for 128 MB write sizes.
- **Auto-Compact:** After an append or merge completes, Spark automatically checks and compacts sub-partitions that contain too many small files.

---

### B. Target File Size Tuning (for Databricks SQL / BI)
If the table is primarily queried by low-latency BI dashboards (Power BI / Databricks SQL), tune target file size to 128 MB or 256 MB:

```sql
ALTER TABLE silver_transactions SET TBLPROPERTIES (
   'delta.targetFileSize' = '268435456' -- 256 MB
);
```

---

## 4. Modern Databricks Alternative: Liquid Clustering

In Databricks Runtime 13.3+, **Liquid Clustering** replaces static partitioning and Z-Ordering, eliminating partition tuning overhead:

```sql
CREATE TABLE silver_transactions (
    transaction_id STRING,
    customer_id STRING,
    transaction_date DATE,
    amount DECIMAL(10, 2)
)
USING DELTA
CLUSTER BY (transaction_date, customer_id);

-- Run periodic clustering
OPTIMIZE silver_transactions;
```
