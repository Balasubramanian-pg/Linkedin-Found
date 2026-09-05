# Data Skew in Apache Spark: Causes, Detection, and Solutions

## Question
What is **Data Skew**, how do you detect it in Spark UI, and what practical techniques do you use to resolve it?

---

## 1. What is Data Skew?

**Data Skew** occurs when data is distributed unevenly across partitions due to high-cardinality imbalance on the partition/join key (e.g., millions of `NULL` keys, a default value like `"UNKNOWN"`, or a viral merchant/celebrity ID).

```
Even Distribution:
  Partition 1: [██████████] 100MB (Finished in 10s)
  Partition 2: [██████████] 100MB (Finished in 10s)
  Partition 3: [██████████] 100MB (Finished in 10s)

Skewed Distribution:
  Partition 1: [█] 10MB (Finished in 2s)
  Partition 2: [█] 10MB (Finished in 2s)
  Partition 3: [██████████████████████████████████████████████████] 5GB (Stuck for 45 mins / OOM)
```

In Spark, a stage only finishes when its **slowest task** completes. A single skewed partition can freeze an entire multi-node cluster.

---

## 2. How to Detect Data Skew

In the **Spark UI** &rarr; **Stage Details**:
- Look at the **Task Deserialization / Duration / Shuffle Read** metrics table.
- **Classic Signature:**
  - `Min Task Time:` 3 seconds
  - `Median (50th percentile):` 5 seconds
  - `Max Task Time:` 48 minutes
  - `Max Shuffle Read Size:` 8.5 GB vs `75th percentile:` 40 MB

---

## 3. Techniques to Mitigate Data Skew

### Approach 1: Salting (The Gold Standard for Join Skew)
Salting adds a random uniform integer (salt) to the join key of the skewed table, and replicates the matching key across the lookup table to distribute skewed keys across multiple partitions.

```python
from pyspark.sql import functions as F

# Step 1: Add a salt key (0 to 4) to the large skewed table
SALT_FACTOR = 5
df_large_salted = df_large.withColumn("salt", F.concat(F.col("join_key"), F.lit("_"), F.floor(F.rand() * SALT_FACTOR)))

# Step 2: Explode / Replicate the lookup table across all salt values
df_small_exploded = df_small.withColumn("salt_array", F.array([F.lit(i) for i in range(SALT_FACTOR)])) \
                            .withColumn("salt_suffix", F.explode("salt_array")) \
                            .withColumn("salt", F.concat(F.col("join_key"), F.lit("_"), F.col("salt_suffix")))

# Step 3: Join on the salted key
df_result = df_large_salted.join(df_small_exploded, on="salt", how="inner").drop("salt", "salt_array", "salt_suffix")
```

---

### Approach 2: Enable Adaptive Query Execution (AQE) Skew Join
In modern Spark (Spark 3.0+ / Databricks), AQE automatically detects skewed partitions at runtime and splits them into smaller sub-partitions.

```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionFactor", "5")
spark.conf.set("spark.sql.adaptive.skewJoin.skewedPartitionThresholdInBytes", "256MB")
```

---

### Approach 3: Broadcast Hash Join
If one of the joining tables is small (dimension table), broadcast it to avoid the shuffle phase entirely:

```python
from pyspark.sql.functions import broadcast

df_joined = df_large.join(broadcast(df_dim), "customer_id")
```

---

### Approach 4: Isolating and Handling Null / Default Keys
If skew is caused by millions of `NULL` or dummy values (e.g., `customer_id = -1`):

```python
# Separate valid and invalid/null keys
df_valid = df_transactions.filter(F.col("customer_id").isNotNull())
df_nulls = df_transactions.filter(F.col("customer_id").isNull())

# Join only valid keys
df_joined_valid = df_valid.join(df_customers, "customer_id", "inner")

# Union back the null rows without joining
df_final = df_joined_valid.unionByName(
    df_nulls.withColumn("customer_name", F.lit(None))
)
```

---

## 4. Summary Matrix

| Skew Scenario | Recommended Solution |
| :--- | :--- |
| Huge Table joined with Small Table (< 1GB) | **Broadcast Hash Join** |
| Huge Table joined with Huge Table (Heavy Key Skew) | **Salting Technique** |
| Runtime Partition Imbalance on Spark 3.x+ | **AQE Skew Join (`spark.sql.adaptive.skewJoin.enabled`)** |
| High count of `NULL` / Default Values | **Filter, Process & Union** |
