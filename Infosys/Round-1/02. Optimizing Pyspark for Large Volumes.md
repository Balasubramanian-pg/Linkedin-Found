# Optimizing PySpark Jobs for Large Data Volumes

## Question
How do you optimize Apache Spark / PySpark jobs to process multi-terabyte datasets efficiently without running out of memory?

---

## 1. Key Optimization Pillars

```
+-----------------------------------------------------------------------------------------------+
|                             PySpark Big Data Optimization Pillars                             |
+-----------------------+-----------------------+-----------------------+-----------------------+
|  Join Optimization   |  Partition Sizing     |  Memory & Storage     |  Code Optimization    |
| • Broadcast Join      | • AQE Coalescing      | • Avoid Spills to Disk| • Avoid Python UDFs   |
| • Key Salting         | • 100-200MB Partitions| • Optimize Writes     | • Vectorized Pandas   |
+-----------------------+-----------------------+-----------------------+-----------------------+
```

---

## 2. Practical Tuning Strategies

### 1. Enable Adaptive Query Execution (AQE)
In Spark 3.0+, AQE re-optimizes query plans at runtime based on completed stage statistics:
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

### 2. Broadcast Hash Joins for Dimension Tables
Eliminate expensive network shuffles by broadcasting small lookup tables (< 1 GB):
```python
from pyspark.sql.functions import broadcast

df_joined = df_large_fact.join(broadcast(df_small_dim), "store_id")
```

### 3. Eliminate Data Skew with Key Salting
When join keys are heavily imbalanced, add a random salt (0 to N) to distribute skewed keys across multiple partitions.

### 4. Replace Slow Python UDFs with Native PySpark / Pandas UDFs
Native PySpark expressions run inside the JVM using optimized Tungsten bytecode, avoiding slow Py4J IPC serialization.

### 5. Delta Lake Compaction & Z-Ordering
```sql
OPTIMIZE fact_transactions ZORDER BY (customer_id, transaction_date);
```
