Data deduplication is a critical step in building reliable PySpark pipelines. Duplicates can inflate metrics, break joins, and significantly increase storage and processing costs. PySpark offers three primary mechanisms for deduplication, each with distinct behaviors, performance characteristics, and use cases. Choosing the right tool depends on whether you need full-row uniqueness, business-key uniqueness, or deterministic control over which record survives.

---

## Understanding the types of duplicates

Before selecting a method, it helps to classify what you are dealing with:

- **Exact duplicates**: Rows where every column value matches. These typically originate from accidental double-writes or idempotency failures in ingestion.
- **Partial duplicates**: Rows that share the same key columns (e.g., `user_id`) but differ in other attributes such as `updated_at` or `name`. These require a decision about which version is authoritative.

The three deduplication approaches map directly to these scenarios.

---

## `distinct()`: full-row deduplication

`distinct()` removes rows that are identical across all columns. It is the simplest form of deduplication and is appropriate when you need to guarantee that every row in the output is unique across the entire schema.

### How it works

`distinct()` hashes all N columns of every row and keeps one row per unique hash. On wide tables, this is an expensive operation because Spark must compare every column value across all rows. The surviving row is arbitrary when duplicates exist—`distinct()` does not give you control over which instance is retained.

### When to use it

- You need full-row uniqueness, not uniqueness on a business key.
- The dataset is relatively narrow, or performance is not the primary concern.
- Duplicates are exact copies, such as repeated identical records from a source system.

```python
# Full-row deduplication
df_distinct = df.distinct()
```

`distinct()` triggers a shuffle that redistributes every row by all columns, which can be costly on wide DataFrames. In benchmark tests, `distinct()` has been measured at approximately 0.013 seconds on a small sample, slightly slower than `dropDuplicates()` because it hashes all columns.

---

## `dropDuplicates()`: key-based deduplication

`dropDuplicates()` is the more flexible and commonly recommended method. It allows you to specify a subset of columns that define uniqueness, meaning you can deduplicate on a business key (e.g., `customer_id`) while retaining all other columns from one of the matching rows.

### Behavior and parameters

- **`subset`**: An optional list of column names. When omitted, `dropDuplicates()` behaves identically to `distinct()` and removes full-row duplicates.
- **`keep`**: In the PySpark DataFrame API, the `keep` parameter is not exposed as a keyword argument in the same way as pandas; the DataFrame API retains the **first** occurrence of each key by default, but this first occurrence is not guaranteed to be deterministic across runs. The surviving row is **arbitrary** when multiple rows share the same subset values.

### When to use it

- A business key defines uniqueness, and you want to compare only the relevant columns rather than all N columns.
- You need a faster, more memory-efficient alternative to `distinct()` on wide tables.
- The exact row retained does not matter, or the dataset is naturally ordered such that the "first" row is acceptable.

```python
# Deduplicate on a business key
df_deduped = df.dropDuplicates(["customer_id"])

# Deduplicate on a composite key
df_deduped = df.dropDuplicates(["customer_id", "order_id"])
```

### Performance advantage

The primary performance benefit of `dropDuplicates(subset)` is that Spark hashes only the columns in the subset, not all columns. On a 40-column table where uniqueness is defined by 3 columns, `dropDuplicates(["col1", "col2", "col3"])` avoids hashing the other 37 columns. Benchmark tests show `dropDuplicates()` at approximately **0.01 seconds** versus `distinct()` at **0.013 seconds** on the same small dataset.

### The determinism problem

When `dropDuplicates(subset)` is used, the row that survives is non-deterministic. Adding `orderBy()` before `dropDuplicates()` does **not** guarantee determinism, because Spark’s internal partitioning and physical plan can reorder rows after the sort. This is a known limitation (SPARK-26767, SPARK-25420). If you need to control which record survives, you must use a window function.

---

## Window functions with `row_number()`: deterministic deduplication

Window-based deduplication is the only method that gives you **deterministic control** over which row survives. You define a window partitioned by the deduplication key and ordered by a tiebreaker column, assign a row number within each partition, and filter to keep only the desired rank.

### The standard pattern

The canonical pattern uses `row_number()` (not `rank()` or `dense_rank()`) because `row_number()` guarantees a unique sequential number even when ties exist in the ordering column.

```python
from pyspark.sql import Window
from pyspark.sql.functions import row_number, col

window_spec = Window.partitionBy("customer_id").orderBy(col("updated_at").desc())

df_deduped = (
    df.withColumn("rn", row_number().over(window_spec))
      .filter(col("rn") == 1)
      .drop("rn")
)
```

This example keeps the **most recently updated** record for each `customer_id`. You can reverse the ordering (`asc()`) to keep the earliest record, or order by any other tiebreaker column that reflects business priority.

### Why `row_number()` rather than `rank()`

If two rows tie on the ordering column, `rank()` assigns the same rank to both, and filtering on `rank() == 1` would retain both duplicates. `row_number()` breaks ties arbitrarily but uniquely, ensuring exactly one row per partition survives.

### When to use it

- You need to keep the **latest**, **earliest**, or **highest-priority** record per key.
- Your deduplication logic must be deterministic and reproducible across runs.
- Partial duplicates exist with differing non-key attributes (e.g., late-arriving updates to the same entity).

### Performance characteristics

Window-based deduplication also requires a shuffle, because rows with the same partition key must be co-located. However, the shuffle is on the partition key (the deduplication key), not on all columns, making it comparable in shuffle volume to `dropDuplicates(subset)`. The additional sort within each partition introduces some overhead, but this is usually outweighed by the determinism benefit for production pipelines.

Some practitioners recommend window functions over `dropDuplicates()` for this reason, especially when key distribution is understood and data can be repartitioned sensibly beforehand.

---

## Comparison at a glance

| Method | What defines a duplicate | Deterministic row retention | Columns hashed / shuffled | Relative performance |
|---|---|---|---|---|
| `distinct()` | All columns | Not applicable (identical rows only) | All N columns | Slowest on wide tables |
| `dropDuplicates(subset)` | Specified key columns | No—surviving row is arbitrary | Only subset columns | Fast; best for most key-based dedup |
| `Window + row_number()` | Specified key columns | Yes—controlled by `orderBy` | Partition key columns + sort | Moderate; shuffle + sort overhead |

---

## Best practices for production pipelines

### Choose the method based on the definition of a duplicate

The first question is always: **what actually makes two rows duplicates?** If the answer is "every column must match," use `distinct()` or `dropDuplicates()` without a subset. If the answer is "the same `customer_id`," use `dropDuplicates(["customer_id"])` for speed, or a window function when determinism matters.

### Minimize the data being compared

Select only the columns you need **before** deduplication whenever possible. Deduplication compares and shuffles data, so carrying unnecessary columns through the operation increases memory pressure and network traffic. Filter out irrelevant records before the expensive dedup step as well.

### Repartition before deduplication

If you know the distribution of the deduplication key, repartition the DataFrame by that key before calling `dropDuplicates()` or the window function. This co-locates matching keys on the same partition and can reduce the volume of data shuffled during the deduplication stage.

```python
df = df.repartition("customer_id")
df_deduped = df.dropDuplicates(["customer_id"])
```

### Cache the deduplicated result if it is reused

If the deduplicated DataFrame will feed multiple downstream operations, cache or persist it to avoid recomputing the shuffle and deduplication logic each time it is accessed.

### Avoid running deduplication repeatedly

Deduplication is an expensive operation. If the same deduplicated dataset is needed in several places, compute it once and reuse the cached result rather than chaining deduplication calls across multiple branches of the pipeline.

### Monitor shuffle and stage duration

Use the Spark UI to inspect shuffle read/write volumes and stage durations around deduplication steps. A deduplication stage that dominates the job runtime is a signal to review the key definition, repartitioning strategy, or whether a window function with a narrower shuffle is more appropriate.

### Consider `groupBy` + aggregation only when you need aggregated values

An alternative pattern is to `groupBy` the key columns and aggregate the remaining columns (e.g., `F.first()` or `F.max()`). This gives you control over the aggregation but is generally **the slowest** approach due to the extra computation involved—benchmarks show `groupBy() + agg()` at roughly **0.12 seconds** versus **0.01 seconds** for `dropDuplicates()` on the same data. Use it when you need aggregated results as part of deduplication, not as a pure dedup mechanism.

---

## Summary

For most deduplication tasks, **`dropDuplicates()` with a column subset** is the recommended default: it is fast, explicit, and avoids unnecessary comparisons across wide tables. Use **`distinct()`** only when full-row uniqueness is genuinely required and the table is not excessively wide. When the pipeline demands **deterministic retention**—keeping the latest or highest-priority record per key—use a **window function with `row_number()`**, partitioned by the deduplication key and ordered by a meaningful tiebreaker. In all cases, select columns early, repartition by the dedup key when the distribution is known, and cache the deduplicated result if it will be reused.
