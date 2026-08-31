# Scenario 04: Designing Pipelines for Late-Arriving Records

## Problem Statement
An upstream transactional system transmits events that occurred days or weeks ago (late-arriving data). How do you architect a Databricks streaming and batch pipeline that accurately incorporates historical updates without corrupting daily aggregates or rewriting petabytes of storage?

---

## 1. The Core Design Dilemma: Event Time vs Ingestion Time

```
Transaction Occurs: 2026-08-10 (Event Date)
Network Delay / Sync: Arrives on 2026-08-30 (Ingestion Date)
```

- **Mistake:** Partitioning the raw/bronze landing layer physically by `event_date`. This forces the pipeline to scan and write into hundreds of old historical folders every single run.
- **Best Practice:** Partition landing data physically by **`ingestion_date`**, while storing **`event_timestamp`** as a column for analytical logic.

---

## 2. Solution for Real-Time Streaming: Watermarking & State Store

Using Spark Structured Streaming, define a bounded watermark threshold:

```python
from pyspark.sql import functions as F

streaming_df = (
    spark.readStream
    .format("delta")
    .table("bronze_pos_swipes")
    # Allow late data up to 3 days behind max event time
    .withWatermark("event_timestamp", "3 days")
    .groupBy(
        F.window("event_timestamp", "1 day"),
        "store_id"
    )
    .agg(F.sum("amount").alias("daily_revenue"))
)

(
    streaming_df.writeStream
    .format("delta")
    .outputMode("update") # Update existing state buckets
    .option("checkpointLocation", "/mnt/lake/checkpoints/pos_aggregates")
    .table("silver_store_daily_revenue")
)
```

---

## 3. Solution for Batch Lakehouse: Targeted Delta Upserts (`MERGE INTO`)

When late batch files arrive, compute the affected date boundaries dynamically, and update only the matching partitions in the Gold Fact/Summary tables:

```python
# 1. Read incoming batch and find the earliest event date affected
incoming_df = spark.read.table("bronze_incremental_batch")
min_event_date = incoming_df.select(F.min("event_date")).collect()[0][0]

# 2. Stage aggregated metrics
staged_aggregates = (
    incoming_df
    .groupBy("event_date", "customer_id")
    .agg(F.sum("amount").alias("incremental_amount"))
)
staged_aggregates.createOrReplaceTempView("staged_late_data")

# 3. Targeted Partition Merge (Prunes partitions before min_event_date)
spark.sql(f"""
    MERGE INTO gold_customer_daily_summary AS target
    USING staged_late_data AS source
    ON target.event_date = source.event_date 
   AND target.customer_id = source.customer_id
   AND target.event_date >= '{min_event_date}' -- Partition Pruning Predicate!
    WHEN MATCHED THEN
      UPDATE SET 
        target.total_amount = target.total_amount + source.incremental_amount,
        target.last_modified_at = current_timestamp()
    WHEN NOT MATCHED THEN
      INSERT (event_date, customer_id, total_amount, last_modified_at)
      VALUES (source.event_date, source.customer_id, source.incremental_amount, current_timestamp());
""")
```

---

## 4. Handling Late-Arriving Dimensions (Early Arriving Facts)

When a transaction (Fact) arrives before the Customer account (Dimension) is created:

1. **Insert Inferred Dummy Dimension:** Create a placeholder record in `Dim_Customer` (`customer_id = 9812`, `name = 'PENDING_REGISTRATION'`, `is_inferred = TRUE`).
2. **Fact Ingestion:** Link the transaction to the inferred dimension key so revenue reporting is never dropped or delayed.
3. **Reconciliation:** When the true customer registration record arrives in a subsequent batch, update `Dim_Customer` attributes and flip `is_inferred = FALSE`.
