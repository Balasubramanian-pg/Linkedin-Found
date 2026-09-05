# Comprehensive Guide to Optimizing Slow-Running Spark Jobs

## Question
How do you systematically identify bottlenecks and optimize a slow-running Apache Spark / Databricks job?

---

## 1. Step-by-Step Diagnostic Framework (Spark UI Analysis)

Before changing code or cluster settings, inspect the **Spark UI**:

```
[ Check Spark UI ]
       |
       +---> Check "Event Timeline" (Are executors idle? Garbage Collection overhead?)
       |
       +---> Check "Stages & Tasks Summary" (Max task duration vs Median task duration -> Data Skew)
       |
       +---> Check "Spill (Memory / Disk)" (Insufficient executor memory / high shuffle volume)
       |
       +---> Check "SQL / Plan Visualization" (SortMergeJoin vs BroadcastHashJoin, full scans)
```

---

## 2. Key Optimization Strategies

### A. Shuffle & Join Optimizations
1. **Broadcast Hash Join (BHJ) for Small-to-Large Table Joins:**
   - Eliminates expensive network shuffle by broadcasting dimension tables (< 10MB default, or up to ~1GB with tuning) to all worker nodes.
   ```python
   from pyspark.sql.functions import broadcast

   # Force broadcast on dimension table
   result_df = large_fact_df.join(broadcast(small_dim_df), "customer_id")
   ```
2. **Enable Adaptive Query Execution (AQE):**
   - AQE dynamically re-optimizes query physical plans at runtime based on completed stage statistics.
   ```python
   spark.conf.set("spark.sql.adaptive.enabled", "true")
   spark.conf.set("spark.sql.adaptive.coalescePartitions.enabled", "true")
   spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
   ```

---

### B. Partition Tuning & Shuffle Partitions
1. **Tuning `spark.sql.shuffle.partitions`:**
   - Default is `200`. For large workloads (hundreds of GBs), 200 partitions cause tasks to spill to disk. For small workloads (< 10GB), 200 partitions create excessive scheduling overhead.
   - **Rule of Thumb:** Aim for ~100MB to 200MB of uncompressed data per partition.
   ```python
   spark.conf.set("spark.sql.shuffle.partitions", "1200")
   ```
2. **Coalesce vs Repartition:**
   - `df.coalesce(n)`: Reduces partition count **without a full shuffle** (merges adjacent partitions on same nodes).
   - `df.repartition(n, "key")`: Increases or balances partition distribution with a **full network shuffle**.

---

### C. Eliminate Memory Spills & Garbage Collection (GC) Issues
- **Disk / Memory Spill:** Occurs when a single partition exceeds executor working memory, forcing Spark to serialize data to disk.
- **Remedies:**
  - Increase partition count (`repartition`).
  - Increase executor memory (`spark.executor.memory`) and memory fraction (`spark.memory.fraction`).
  - Avoid Python UDFs (which trigger Py4J IPC serialization overhead); use native PySpark SQL functions or Pandas UDFs (Arrow-based vectorization).

---

### D. File I/O & Storage Layer Optimization (Delta Lake)
1. **Predicate Pushdown & Partition Pruning:**
   - Ensure filter conditions are applied directly on partition columns (`WHERE year = 2026 AND month = 8`) to avoid reading non-matching files.
2. **Delta Lake Compaction (`OPTIMIZE` & `Z-ORDER`):**
   ```sql
   OPTIMIZE silver_transactions ZORDER BY (merchant_id, transaction_date);
   ```
3. **Avoid Small File Problem:**
   - Turn on Auto-Compaction and Optimized Writes in Databricks:
   ```python
   spark.conf.set("spark.databricks.delta.optimizeWrite.enabled", "true")
   spark.conf.set("spark.databricks.delta.autoCompact.enabled", "true")
   ```

---

### E. Caching and Persisting Judiciously
- Only `.cache()` or `.persist()` a DataFrame if it is reused multiple times across separate downstream actions.
- Always unpersist (`df.unpersist()`) once downstream jobs complete to free cluster memory.

---

## 3. Spark Performance Optimization Summary Checklist

| Bottleneck Observed in Spark UI | Root Cause | Actionable Solution |
| :--- | :--- | :--- |
| Single task running for hours while others finish in seconds | **Data Skew** | Salting, AQE skew join, Broadcast join |
| High GC Time (> 15% of stage runtime) | Memory pressure / large objects | Increase executor memory, reduce partition sizes, use off-heap memory |
| Spill (Disk) in ShuffleExchange stage | Partitions too large for memory | Increase `spark.sql.shuffle.partitions` |
| Millions of tiny KB-sized files | Inefficient write partitioning | Run `OPTIMIZE`, enable `autoCompact` |
| Driver Out of Memory (OOM) | Calling `.collect()` on large DF | Use `.take(n)`, write output directly to storage |
