# Incremental Hourly Processing Systems: Watermarking & Backfill Strategies

## Question
How do you build a resilient, incremental hourly batch processing system that handles late-arriving events with watermarking and executes safe historical backfills?

---

## 1. Architecture of Incremental Hourly Processing

```
Hourly Schedule Trigger (e.g., Airflow / ADF every hour on the hour):
Window Execution: [ T_start = 2026-08-30 14:00:00, T_end = 2026-08-30 15:00:00 ]

[ Ingestion Source ] ──> Filter by Ingestion Timestamp [T_start, T_end]
                                │
                                ▼
                       [ Extract Max & Min Event Timestamps ]
                                │
                                ▼
                       [ Watermark Lookback Buffer (e.g., 3 Hours) ]
                                │
                                ▼
                       [ Atomic Delta Merge into Target Dimension/Fact ]
                                │
                                ▼
                       [ Update High-Watermark Metadata Store ]
```

```mermaid
graph TD
    A[Hourly Schedule Trigger] -->|Airflow / ADF every hour| B[Window Execution]
    
    subgraph Window Configuration [T_start = 2026-08-30 14:00:00, T_end = 2026-08-30 15:00:00]
        B --> C[Ingestion Source]
        C -->|Filter by Ingestion Timestamp| D{Ingestion Time within [T_start, T_end]?}
        D -->|Yes| E[Extract Max & Min Event Timestamps]
        D -->|No| F[Skip / Log]
    end

    subgraph Watermark & Merge Logic
        E --> G[Watermark Lookback Buffer]
        G -->|e.g., 3 Hours| H[Calculate Watermark Window]
        H --> I[Read Late-Arriving Data within Buffer]
        I --> J[Atomic Delta MERGE into Target Dimension/Fact]
    end

    subgraph Metadata Store
        J --> K[Update High-Watermark Metadata Store]
        K --> L[(High-Watermark Table)]
        L -->|Next run reference| B
    end

    J --> M[Target Tables]
    M --> N[(Dimension Tables)]
    M --> O[(Fact Tables)]
```

## 2. Watermarking and Lookback Windows

In an hourly pipeline running at 15:00 for the `[14:00 - 15:00]` ingestion window:
- Most records have `event_time` between `14:00 - 15:00`.
- A fraction of records have `event_time` from `12:30` (delayed network sync).
- **Rule:** Never partition downstream tables purely by `event_time` during writes without a controlled lookback merge.

```python
from pyspark.sql import functions as F

def process_hourly_slice(spark, execution_date_hour: str, lookback_hours: int = 3):
    """
    execution_date_hour: e.g., '2026-08-30T14:00:00'
    """
    # 1. Calculate boundaries
    window_end = F.to_timestamp(F.lit(execution_date_hour))
    window_start = window_end - F.expr(f"INTERVAL {lookback_hours} HOURS")
    
    # 2. Read new data arrived in the last hour
    incremental_df = (
        spark.read.table("bronze_card_swipes")
        .filter((F.col("ingestion_time") > window_start) & (F.col("ingestion_time") <= window_end))
    )
    
    # 3. Aggregate hourly metrics
    hourly_metrics = (
        incremental_df
        .groupBy(
            F.window("event_time", "1 hour"),
            "merchant_category",
            "region"
        )
        .agg(
            F.sum("amount").alias("hourly_spend"),
            F.count("tx_id").alias("tx_count")
        )
        .select(
            F.col("window.start").alias("hour_bucket"),
            "merchant_category",
            "region",
            "hourly_spend",
            "tx_count"
        )
    )
    
    # 4. Merge incrementally into Gold Hourly Table
    hourly_metrics.createOrReplaceTempView("staged_hourly_metrics")
    
    spark.sql("""
        MERGE INTO gold_hourly_spend_summary AS target
        USING staged_hourly_metrics AS source
        ON target.hour_bucket = source.hour_bucket
       AND target.merchant_category = source.merchant_category
       AND target.region = source.region
        WHEN MATCHED THEN
          UPDATE SET 
            target.hourly_spend = target.hourly_spend + source.hourly_spend,
            target.tx_count = target.tx_count + source.tx_count,
            target.last_updated = current_timestamp()
        WHEN NOT MATCHED THEN
          INSERT (hour_bucket, merchant_category, region, hourly_spend, tx_count, last_updated)
          VALUES (source.hour_bucket, source.merchant_category, source.region, source.hourly_spend, source.tx_count, current_timestamp())
    """)
```

---

## 3. Backfill Strategy (Handling Upstream Schema Changes or Historical Bug Fixes)

When code logic changes and you need to recompute 6 months of historical data (e.g., 4,300 hours):

### Backfill Best Practices:
1. **Never Run Backfills in One Massive Spark Job:**
   - Attempting to overwrite 6 months of data in one single action causes driver OOM, massive shuffle spills, and locks target tables for hours.
2. **Chunked Dynamic Date Range Slicing:**
   - Execute backfills day-by-day or week-by-week using parameterized Airflow DAG runs (`catchup=True` with `max_active_runs=4`).
3. **Partition Overwrite Mode (`replaceWhere`):**
   - Atomically replace only the target partition without touching other active partitions:
   ```python
   (
       df_backfill_day
       .write
       .format("delta")
       .mode("overwrite")
       .option("replaceWhere", "hour_bucket >= '2026-03-01' AND hour_bucket < '2026-03-02'")
       .saveAsTable("gold_hourly_spend_summary")
   )
   ```
