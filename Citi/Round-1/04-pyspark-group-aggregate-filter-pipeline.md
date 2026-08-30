# PySpark Transformation Pipeline: Grouping, Aggregation & Filtering

## Question
Write a production PySpark transformation pipeline that reads a transaction dataset, applies business filtering, computes multi-metric aggregations across account categories, and filters aggregated results.

---

## 1. Production PySpark Pipeline Code

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.types import DecimalType


def process_portfolio_metrics():
    spark = (
        SparkSession.builder
        .appName("Citi_Portfolio_Aggregations")
        .config("spark.sql.adaptive.enabled", "true")
        .getOrCreate()
    )

    # 1. Read input dataset
    df_raw = spark.read.parquet("abfss://curated@citiadls.dfs.core.windows.net/fact_trades/")

    # 2. Step 1: Pre-aggregation filtering (Predicate Pushdown)
    df_filtered = (
        df_raw
        .filter(
            (F.col("trade_status") == "SETTLED") &
            (F.col("trade_date") >= "2026-01-01") &
            (F.col("notional_usd").isNotNull()) &
            (F.col("notional_usd") > 0)
        )
    )

    # 3. Step 2: Multi-metric Grouping & Aggregation
    df_aggregated = (
        df_filtered
        .groupBy("desk_id", "asset_class", "region")
        .agg(
            F.count("trade_id").alias("total_trades"),
            F.sum("notional_usd").cast(DecimalType(18, 2)).alias("total_notional_usd"),
            F.avg("notional_usd").cast(DecimalType(18, 2)).alias("avg_trade_size"),
            F.max("notional_usd").alias("largest_single_trade"),
            F.countDistinct("counterparty_id").alias("unique_counterparties")
        )
    )

    # 4. Step 3: Post-aggregation filtering (Equivalent to SQL HAVING)
    df_result = (
        df_aggregated
        .filter(
            (F.col("total_trades") >= 50) & 
            (F.col("total_notional_usd") >= 10000000.00) # $10M+ desks
        )
        .orderBy(F.col("total_notional_usd").desc())
    )

    # 5. Write to partitioned curated table
    (
        df_result
        .write
        .mode("overwrite")
        .partitionBy("region", "asset_class")
        .format("delta")
        .saveAsTable("gold_desk_performance_summary")
    )


if __name__ == "__main__":
    process_portfolio_metrics()
```
