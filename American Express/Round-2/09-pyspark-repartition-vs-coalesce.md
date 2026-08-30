# Repartition vs Coalesce in Apache Spark / PySpark

## Question
Explain the fundamental differences between `repartition()` and `coalesce()` in Apache Spark, their impact on network shuffle, and when to use each in production pipelines.

---

## 1. High-Level Comparison Matrix

| Feature | `df.repartition(n)` | `df.coalesce(n)` |
| :--- | :--- | :--- |
| **Network Shuffle** | **Full Network Shuffle** (Redistributes data across all cluster nodes). | **No Full Shuffle** (Merges adjacent partitions locally on existing nodes). |
| **Partition Count Change** | Can **increase** or **decrease** partition count. | Can **only decrease** partition count efficiently. |
| **Data Distribution** | Generates uniform, balanced partitions (eliminates skew). | May produce uneven partition sizes if source partitions were skewed. |
| **Custom Column Partitioning**| Yes (`df.repartition(n, "col")` - Hash partitioning). | No (cannot partition by column expression). |
| **Performance Overhead** | High I/O and network serialization cost. | Minimal overhead (avoids network transfer). |

---

## 2. Visualizing Partition Movement

```
df.coalesce(2) [From 4 partitions down to 2 - Local Node Merging]:
  Node 1: [ Part 1 ] + [ Part 2 ] ──> [ Merged Part A ]  (Zero network transfer)
  Node 2: [ Part 3 ] + [ Part 4 ] ──> [ Merged Part B ]  (Zero network transfer)

df.repartition(2) [From 4 partitions - Full Hash Shuffle across Network]:
  Node 1: [ Part 1 ] ──┬── Hash Partition 0 ──> [ New Part 0 ] (Node 1)
          [ Part 2 ] ──┤
  Node 2: [ Part 3 ] ──┼── Hash Partition 1 ──> [ New Part 1 ] (Node 2)
          [ Part 4 ] ──┘
```

---

## 3. Production Scenarios: When to Use Which?

### A. When to use `coalesce()`:
1. **Target File Consolidation before Writing to Storage:**
   - After heavy filtering operations where 200 partitions only contain 5MB of total data, writing directly creates 200 tiny files.
   - Use `df.coalesce(1)` or `df.coalesce(4)` right before `.write.parquet()` to output clean, compact files without triggering an expensive shuffle.
   ```python
   # Filter 90% of rows, then coalesce before saving
   df_filtered = df_large.filter(F.col("is_fraud") == True)
   df_filtered.coalesce(4).write.mode("overwrite").parquet("abfss://lake/fraud_cases/")
   ```

---

### B. When to use `repartition()`:
1. **Increasing Parallelism for Massive Compute:**
   - Ingesting a single huge 100GB compressed file creates 1 massive partition. The entire cluster sits idle while 1 core processes it.
   - Call `df.repartition(400)` to parallelize work across all cluster cores.
2. **Eliminating Severe Data Skew:**
   - Repartitioning on a composite key with a uniform hash distributes load evenly.
3. **Partitioning by Business Key for Optimized Downstream Joins:**
   ```python
   # Hash partition by merchant_id to co-locate records prior to multiple heavy joins
   df_repartitioned = df_transactions.repartition(200, "merchant_id")
   ```

---

## 4. Common Pitfalls & Anti-Patterns

> [!CAUTION]
> Calling `df.coalesce(1)` too early in a transformation pipeline forces Spark to execute all upstream filter and map transformations on a **single CPU core**, collapsing entire cluster parallelism! Always call `coalesce()` as the final step immediately before writing.
