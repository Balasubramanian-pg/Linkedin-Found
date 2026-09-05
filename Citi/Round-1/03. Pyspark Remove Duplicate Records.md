# Removing Duplicate Records in a PySpark DataFrame

## Question
How do you identify and remove duplicate records from a PySpark DataFrame in batch and streaming workflows?

---

## 1. Method 1: `dropDuplicates()` (Exact or Subset Columns)

```python
from pyspark.sql import SparkSession

spark = SparkSession.builder.appName("Citi_Deduplication").getOrCreate()

# 1. Remove exact duplicate rows across all columns
df_unique = df.dropDuplicates()

# 2. Remove duplicates based on a subset of business keys
df_unique_trades = df.dropDuplicates(subset=["trade_id", "account_id"])
```

---

## 2. Method 2: Retaining the Latest Record via Window Functions

When duplicate records represent state updates and only the most recent version must be kept:

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

window_spec = Window.partitionBy("trade_id").orderBy(
    F.col("event_timestamp").desc(),
    F.col("ingestion_id").desc()
)

df_latest_trades = (
    df
    .withColumn("rn", F.row_number().over(window_spec))
    .filter(F.col("rn") == 1)
    .drop("rn")
)
```

---

## 3. Method 3: Streaming Deduplication with Watermarking

In Spark Structured Streaming, deduplication requires a state watermark to prevent unbounded memory growth:

```python
streaming_df = (
    spark.readStream
    .format("delta")
    .table("bronze_trades")
    .withWatermark("trade_timestamp", "2 hours")
    .dropDuplicates(["trade_id", "trade_timestamp"])
)
```
