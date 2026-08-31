# Scenario 02: Deduplication & Idempotent Processing in Databricks

## Problem Statement
An upstream source system frequently sends duplicate records (identical business keys with same or updated payloads) due to network retries. How do you implement robust deduplication and ensure strictly idempotent processing in Databricks?

---

## 1. Multi-Stage Deduplication Architecture

```
[ Dirty Source Batch ]
           │
           ▼ (Step 1: Intra-Batch Deduplication)
[ Clean In-Memory Micro-Batch (1 row per business key) ]
           │
           ▼ (Step 2: Inter-Table Idempotent Delta MERGE INTO)
[ Curated Delta Lake Table (Zero Duplicates & Deterministic State) ]
```

---

## 2. Step 1: In-Batch Deduplication (PySpark / SQL)

Before merging data into the target table, resolve intra-batch duplicates by ranking records on the event/modification timestamp:

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

def deduplicate_microbatch(df_source, business_key: str, timestamp_col: str):
    """
    Retains the latest record per business key within the active batch.
    """
    window_spec = Window.partitionBy(business_key).orderBy(
        F.col(timestamp_col).desc(),
        F.col("ingestion_time").desc()
    )
    
    return (
        df_source
        .withColumn("row_num", F.row_number().over(window_spec))
        .filter(F.col("row_num") == 1)
        .drop("row_num")
    )
```

---

## 3. Step 2: Idempotent Storage Merge (Delta Lake `MERGE INTO`)

In Delta Lake, merging guarantees **exactly-once write semantics** even if the pipeline fails and restarts:

```sql
MERGE INTO silver_cardholder_accounts AS target
USING staged_clean_batch AS source
ON target.account_id = source.account_id
WHEN MATCHED AND source.updated_at > target.updated_at THEN
  UPDATE SET 
    target.card_status = source.card_status,
    target.credit_limit = source.credit_limit,
    target.billing_address = source.billing_address,
    target.updated_at = source.updated_at,
    target._last_processed_at = current_timestamp()
WHEN NOT MATCHED THEN
  INSERT (account_id, card_status, credit_limit, billing_address, updated_at, _last_processed_at)
  VALUES (source.account_id, source.card_status, source.credit_limit, source.billing_address, source.updated_at, current_timestamp());
```

---

## 4. Step 3: Streaming Deduplication (`dropDuplicates` with Watermarking)

For real-time streaming jobs using Spark Structured Streaming:

```python
streaming_df = (
    spark.readStream
    .format("delta")
    .table("bronze_events")
    # Define state watermark to bound memory usage
    .withWatermark("event_time", "2 hours")
    # Drop duplicates within the 2-hour sliding state horizon
    .dropDuplicates(["transaction_id", "event_time"])
)

(
    streaming_df.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", "/mnt/lake/checkpoints/streaming_dedup")
    .table("silver_events_deduped")
)
```

---

## 5. Ensuring Idempotency in Batch Reprocessing

- **Strict Rule:** Avoid `mode("append")` without deduplication logic.
- **For Batch Partition Overwrites:** Use `replaceWhere` to make partition rewrites deterministic:
  ```python
  (
      df_clean.write
      .format("delta")
      .mode("overwrite")
      .option("replaceWhere", "transaction_date = '2026-08-30'")
      .saveAsTable("silver_events")
  )
  ```
