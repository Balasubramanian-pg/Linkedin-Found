# Scenario 08: Pipeline Restartability & Fault-Tolerant Idempotency

## Problem Statement
A long-running production Databricks ETL job fails halfway through processing due to a transient cluster crash or network timeout. When the job is re-triggered, how do you ensure it resumes cleanly without creating duplicate records or leaving orphan staging files?

---

## 1. The Core Principles of Resilient Data Pipeline Design

```
+-------------------------------------------------------------------------------+
|                       Pipeline Restartability Pillars                         |
+-------------------+-------------------+-------------------+-------------------+
|  ACID Storage     | Streaming State   | Deterministic     | Staged Partition  |
|  (Delta Lake)     | Checkpoints       | MERGE Upserts     | Dynamic Overwrites|
+-------------------+-------------------+-------------------+-------------------+
```

---

## 2. Solution 1: Leverage Delta Lake ACID Transactions

Unlike legacy Parquet/Hadoop (where failed jobs left dirty uncommitted files on disk), **Delta Lake writes are atomic**:
- If a write task or executor dies halfway, the transaction is **never committed** to `_delta_log/*.json`.
- Readers continue reading the previous valid snapshot.
- Uncommitted temporary Parquet files are automatically ignored and subsequently cleaned up during scheduled `VACUUM` runs.

---

## 3. Solution 2: Streaming Checkpoints for Auto-Resume

In Spark Structured Streaming / Auto Loader pipelines, checkpoints persist the exact byte offset and commit index:

```python
(
    df.writeStream
    .format("delta")
    .option("checkpointLocation", "abfss://checkpoints@datalake.dfs.core.windows.net/job_offset_01")
    .table("silver_table")
)
```
- **On Restart:** The engine inspects the checkpoint WAL, detects the exact microbatch that failed, re-reads only those offsets, and re-executes the transaction.

---

## 4. Solution 3: Partition-Level Atomic Overwrites (`replaceWhere`)

For scheduled batch ETL pipelines that recalculate daily partition slices:

```python
# ❌ Anti-pattern: Non-idempotent append
# df.write.mode("append").saveAsTable("fact_sales") # Running twice duplicates rows!

# ✅ Idempotent Pattern: Atomic Partition Replacement
(
    df_daily_sales
    .write
    .format("delta")
    .mode("overwrite")
    .option("replaceWhere", "business_date = '2026-08-30'")
    .saveAsTable("fact_sales")
)
```
> If the job fails halfway, the target partition remains untouched. If it succeeds, the entire target partition is swapped atomically.

---

## 5. Solution 4: Control / Watermark Table with Transaction State

For multi-stage orchestration (e.g., Databricks Workflows / ADF):

```sql
CREATE TABLE etl_batch_control (
    pipeline_name STRING,
    batch_id STRING,
    window_start TIMESTAMP,
    window_end TIMESTAMP,
    status STRING, -- 'STARTED', 'COMPLETED', 'FAILED'
    updated_at TIMESTAMP
);
```

### Execution Protocol:
1. **Start:** Check if `batch_id` is marked `COMPLETED`. If yes, skip to next batch. If `FAILED` or not found, set status to `STARTED`.
2. **Execute:** Run transformations inside Delta transaction.
3. **Commit:** Update status to `COMPLETED`.
