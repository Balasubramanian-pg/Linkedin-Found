# Troubleshooting Underutilized Spark Executors (500 Allocated, Only 20 Busy)

## Question
Your Spark cluster has 500 executor cores allocated, but the Spark UI reveals only 20 tasks are running while 480 cores sit completely idle. What causes this low parallelism, and how do you fix it?

---

## 1. Common Root Causes & Diagnostics

```
[ 500 Cores Allocated vs 20 Active Tasks ]
                     │
         ┌───────────┼───────────┐
         ▼           ▼           ▼
   [ Low Partition ] [ Single File ] [ Inefficient Coalesce ]
      Count (< 20)      Input (Gzip)    (`coalesce(1)` in pipeline)
```

### Cause 1: Partition Count is Smaller Than Core Count
- **Symptom:** Spark creates exactly **1 task per partition**. If a dataset has only 20 partitions, Spark can only run 20 tasks concurrently, leaving the remaining 480 cores idle.
- **Fix:** Explicitly repartition data to match or exceed cluster core count (2x to 3x cores):
  ```python
  df_parallel = df.repartition(1000)
  ```

---

### Cause 2: Non-Splittable Compressed Source Files (e.g., `.gz` / `.tar`)
- **Symptom:** Ingesting a single 100 GB Gzip-compressed file. Because standard Gzip is non-splittable, a single CPU core must decompress the entire file from start to finish.
- **Fix:** Ingest splittable formats (Parquet, Snappy, Bzip2) or decompress into multi-part files.

---

### Cause 3: Accidental Early `coalesce(1)`
- **Fix:** Move `coalesce()` to the very last line immediately before writing.
