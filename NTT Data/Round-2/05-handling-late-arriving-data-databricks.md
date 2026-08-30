# Handling Late-Arriving Data in Databricks

## Question
How do you handle late-arriving data (out-of-order records) in Databricks (Streaming & Batch)?

---

## 1. Understanding Late-Arriving Data

**Late-arriving data** occurs when an event happens at time $T_{event}$ (e.g., in an IoT sensor or mobile app offline mode), but reaches the cloud processing pipeline at a much later processing time $T_{processing}$ (hours or days later due to network latency, system outages, or batch uploads).

---

## 2. Strategy 1: Spark Structured Streaming with Watermarking

Spark Structured Streaming handles late data by maintaining a sliding state window and pruning old state once the **Watermark** passes.

```
Event Time Horizon:
[---------------- State Maintained in Memory / RocksDB ----------------]
<---- Dropped (Too Late) ---- | <--- Processed & Aggregated ---> | [Current Max Event Time]
                             ^
                      Watermark Threshold
                      (Max Event Time - Delay Tolerance)
```

### PySpark Structured Streaming Example:
```python
from pyspark.sql import functions as F

streaming_df = (
    spark.readStream
    .format("delta")
    .table("bronze_iot_telemetry")
    # Define 1-hour delay threshold on event_timestamp
    .withWatermark("event_timestamp", "1 hour")
    .groupBy(
        F.window("event_timestamp", "10 minutes", "5 minutes"),
        "device_id"
    )
    .agg(F.avg("temperature").alias("avg_temp"))
)

query = (
    streaming_df.writeStream
    .format("delta")
    .outputMode("append") # Or 'update' depending on downstream consumers
    .option("checkpointLocation", "/mnt/checkpoints/iot_stream")
    .table("silver_iot_telemetry_aggregated")
)
```
- **How it works:** Any event with `event_timestamp < (max_event_timestamp_seen - 1 hour)` is deemed too late and discarded from state aggregations, preventing infinite memory consumption.

---

## 3. Strategy 2: Delta Lake Upserts (`MERGE INTO`) in Batch Pipelines

For batch ETL pipelines, late-arriving dimensional or transactional records must update existing historical aggregates or target dimension tables without reprocessing the entire table history.

```sql
MERGE INTO gold_fact_sales AS target
USING late_batch_updates AS source
ON target.order_id = source.order_id 
   AND target.order_date = source.order_date
WHEN MATCHED AND source.event_timestamp > target.event_timestamp THEN
  UPDATE SET 
    target.order_amount = source.order_amount,
    target.order_status = source.order_status,
    target.event_timestamp = source.event_timestamp,
    target.is_late_record = true,
    target.last_updated_at = current_timestamp()
WHEN NOT MATCHED THEN
  INSERT (order_id, order_date, order_amount, order_status, event_timestamp, is_late_record, last_updated_at)
  VALUES (source.order_id, source.order_date, source.order_amount, source.order_status, source.event_timestamp, true, current_timestamp());
```

---

## 4. Strategy 3: Partitioning by Ingestion Date vs Event Date

A critical architectural pitfall in data lakes is partitioning by event date when data arrives weeks late, as this forces the engine to scan and rewrite hundreds of old historical partition folders.

### Recommended Dual-Timestamp Pattern:
- **`event_timestamp`:** Represents the actual business timestamp used for business logic and analytical reporting.
- **`ingestion_date`:** Represents the physical partition key on ADLS Gen2 (`/year=2026/month=08/day=30/`).
- **Benefit:** Ingestion write jobs write exclusively to today's partition folder regardless of how old the event timestamp is. Downstream transformations then reconcile historical tables via targeted indexed/Z-Ordered merges.

---

## 5. Strategy 4: Handling Late-Arriving Dimensions (Inferred Dimensions)

In a Star Schema, a transaction/fact row may arrive before the corresponding customer dimension record exists:

1. **Create a Placeholder / Inferred Dimension Row:**
   - Insert an inferred dummy record in `Dim_Customer` (`customer_sk = -1` or with `is_inferred = TRUE`).
   - The Fact table references this dimension key so sales numbers are not lost or blocked.
2. **Reconcile When Dimension Data Arrives:**
   - When the actual customer record arrives later, update the inferred record with true customer attributes.
