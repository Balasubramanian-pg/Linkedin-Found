# PySpark: Country-Wise 3rd Highest Order Amount

## Question
Write a PySpark query to find the 3rd highest order amount for each country, handling ties and countries with fewer than 3 orders properly.

---

## 1. PySpark Implementation

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window


def get_3rd_highest_order_per_country():
    spark = SparkSession.builder.appName("Tredence_Country_Orders").getOrCreate()

    # Define Window Specification
    window_spec = Window.partitionBy("country").orderBy(F.col("order_amount").desc())

    # Ingest Orders Table
    df_orders = spark.read.table("fact_orders")

    # Filter for Rank 3 using DENSE_RANK()
    df_third_highest = (
        df_orders
        .withColumn("order_rank", F.dense_rank().over(window_spec))
        .filter(F.col("order_rank") == 3)
        .select(
            "country",
            "order_id",
            "customer_id",
            "order_amount",
            "order_date"
        )
        .orderBy("country")
    )

    df_third_highest.show()
```
