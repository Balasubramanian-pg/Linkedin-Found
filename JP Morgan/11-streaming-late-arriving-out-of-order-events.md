# Handling Late-Arriving and Out-of-Order Events in Streaming Pipelines

## Question
How do you handle out-of-order and late-arriving trade events in a distributed streaming pipeline (Apache Flink / Spark Structured Streaming) without causing memory exhaustion or inaccurate financial balances?

---

## 1. Why Events Arrive Out-of-Order

```
Event Generation Order (Event Time):
  Trade 1 (10:00:01) ──> Trade 2 (10:00:02) ──> Trade 3 (10:00:03)

Arrival Order at Consumer (Processing Time):
  Trade 2 (10:00:02) ──> Trade 1 (10:00:01 - Arrived Late!) ──> Trade 3 (10:00:03)
```
- **Causes:** Network jitter, mobile offline caching, multi-partition Kafka processing speeds.

---

## 2. Solution: Watermarking & State Bounding (Spark Structured Streaming)

A **Watermark** defines the point in time where the engine assumes no more late data will arrive for a given time window.

```python
from pyspark.sql import functions as F

trade_stream = (
    spark.readStream
    .format("kafka")
    .option("subscribe", "market_trades")
    .load()
    # Parse payload
    .select(F.from_json(F.col("value").cast("string"), trade_schema).alias("data"))
    .select("data.*")
    # Define 10-minute late data tolerance based on event_timestamp
    .withWatermark("event_timestamp", "10 minutes")
    .groupBy(
        F.window("event_timestamp", "5 minutes", "1 minute"),
        "symbol"
    )
    .agg(
        F.sum("volume").alias("total_volume"),
        F.avg("price").alias("vwap_price")
    )
)
```

---

## 3. Handling Extremely Late Data (Past Watermark Horizon)

If an event arrives **after** the watermark has already closed (e.g., 2 hours late):

1. **Watermark Engine Drops from Streaming State:** This protects executor state memory from unbounded growth.
2. **Side Output / Dead Letter Table:** Route dropped records into a `late_arriving_trades_lake` Delta table.
3. **Reconciliation Batch Job:** Run an hourly/daily batch merge that incorporates late records into historical accounting ledgers.
