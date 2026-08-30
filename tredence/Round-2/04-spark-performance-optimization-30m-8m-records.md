# Optimizing Spark Joins Between 30 Million & 8 Million Records

## Question
You have a table with **30 Million rows (~15 GB)** joining with another table of **8 Million rows (~3 GB)**. The job runs slowly with high shuffle I/O. How do you optimize this join?

---

## 1. Analysis of the Scenario
- Neither table is small enough for standard default Broadcast Join (< 10 MB).
- Both tables are medium-large, triggering a standard **Sort Merge Join (SMJ)** with 2x full network shuffles.

---

## 2. Optimization Strategy 1: Increase Broadcast Threshold (If Memory Permits)
If executor nodes have 32+ GB RAM, increase the broadcast threshold to broadcast the 8M (3 GB) table directly:

```python
from pyspark.sql.functions import broadcast

# Option A: Explicit Broadcast Hint
df_joined = df_30m.join(broadcast(df_8m), on="join_key", how="inner")

# Option B: Increase session auto-broadcast threshold
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "4294967296") # 4 GB
```

---

## 3. Optimization Strategy 2: Pre-Bucket / Co-Locate Tables (Bucket Joins)
If these tables are joined repeatedly in daily pipelines, bucket both tables by `join_key` into the **exact same number of buckets** (e.g. 64 buckets):

```sql
-- Table 1: 30M Rows
CREATE TABLE bucketed_table_30m
USING DELTA
CLUSTERED BY (join_key) INTO 64 BUCKETS;

-- Table 2: 8M Rows
CREATE TABLE bucketed_table_8m
USING DELTA
CLUSTERED BY (join_key) INTO 64 BUCKETS;
```
> **Impact:** Eliminates 100% of network shuffle at join time! Spark pairs Bucket 0 with Bucket 0 locally on each node.

---

## 4. Optimization Strategy 3: Tune Adaptive Query Execution (AQE)
```python
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
spark.conf.set("spark.sql.shuffle.partitions", "300")
```
