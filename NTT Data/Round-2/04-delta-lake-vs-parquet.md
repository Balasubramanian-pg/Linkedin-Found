# Delta Lake vs Apache Parquet

## Question
What is Delta Lake and why is it better than raw Parquet?

---

## 1. What is Delta Lake?

**Delta Lake** is an open-source storage layer that brings reliability, performance, and **ACID transactions** to data lakes.

Delta Lake uses **versioned Apache Parquet files** under the hood combined with a strictly sequenced, file-based transaction log called the **Delta Log (`_delta_log/`)**.

```
[ Delta Table Directory ]
├── _delta_log/
│   ├── 00000000000000000000.json   <-- Commit 0 (Schema, added files)
│   ├── 00000000000000000001.json   <-- Commit 1 (Appended/removed files)
│   └── 00000000000000000010.checkpoint.parquet
├── part-00000-tid...snappy.parquet
└── part-00001-tid...snappy.parquet
```

---

## 2. Parquet vs Delta Lake Comparison Matrix

| Capability | Raw Apache Parquet | Delta Lake (Parquet + Delta Log) |
| :--- | :--- | :--- |
| **ACID Transactions** | ❌ No (Readers see partial/corrupt writes during active writes). | ✅ **Full ACID** via Serializability / Optimistic Concurrency Control. |
| **DML Operations (`UPDATE`, `DELETE`, `MERGE`)** | ❌ Cannot modify individual rows; entire partitions/files must be rewritten manually. | ✅ **Native `MERGE INTO` (Upsert)**, `UPDATE`, and `DELETE` with automatic file rewriting. |
| **Time Travel / Audit History** | ❌ No built-in versioning. | ✅ **Full Time Travel** (`VERSION AS OF` or `TIMESTAMP AS OF`) for rollbacks and audits. |
| **Schema Enforcement & Evolution** | ❌ Silent column mismatches or query failures on dirty schemas. | ✅ Prevents bad data writes (**Schema Enforcement**) and supports managed **Schema Evolution**. |
| **Data Compaction & Indexing** | ❌ Manual partition management; prone to small-file problem. | ✅ Built-in `OPTIMIZE`, **Z-ORDER clustering**, and `VACUUM` commands. |
| **Streaming & Batch Convergence**| ❌ Difficult to read/write concurrently in real-time. | ✅ Unified batch and streaming source/sink (Structured Streaming). |

---

## 3. Key Advantages of Delta Lake

### A. Full ACID Transactions
In standard Parquet, if a Spark write job fails halfway, orphan files remain in storage, corrupting downstream reader queries. In Delta Lake:
- Transactions are committed atomically in the JSON transaction log.
- Readers always read a consistent snapshot of the table even while writes/updates are occurring simultaneously.

### B. Efficient Upserts (`MERGE INTO`)
Delta Lake makes CDC and Slowly Changing Dimension (SCD) updates simple and performant:

```sql
MERGE INTO silver_customers AS target
USING bronze_customer_updates AS source
ON target.customer_id = source.customer_id
WHEN MATCHED AND source.is_deleted = true THEN
  DELETE
WHEN MATCHED THEN
  UPDATE SET 
    target.email = source.email,
    target.address = source.address,
    target.updated_at = current_timestamp()
WHEN NOT MATCHED THEN
  INSERT (customer_id, email, address, created_at, updated_at)
  VALUES (source.customer_id, source.email, source.address, current_timestamp(), current_timestamp());
```

### C. Time Travel (Rollback & Reproducibility)
```sql
-- Query data as it existed at a prior version or timestamp
SELECT * FROM silver_customers VERSION AS OF 14;
SELECT * FROM silver_customers TIMESTAMP AS OF '2026-08-25 10:00:00';

-- Restore table after an accidental deletion/corruption
RESTORE TABLE silver_customers TO VERSION AS OF 13;
```

### D. File Compaction (`OPTIMIZE` & Z-ORDER)
Solves the "small file problem" caused by high-frequency streaming writes:

```sql
-- Compact small files into ideal ~1GB parquet files
OPTIMIZE silver_customers
ZORDER BY (customer_id, region);
```
> `ZORDER` organizes data along multi-dimensional space-filling curves, enabling Delta engines to skip non-relevant data files (**Data Skipping**) via column min/max statistics.
