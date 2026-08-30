# Trade Processing Pipeline: Identifying and Eliminating Duplicates

## Question
A high-volume trade-processing pipeline is producing duplicate records downstream. How do you identify where duplicates are introduced, classify duplicate types, and systematically eliminate them with zero trade loss?

---

## 1. Step 1: Classify Duplicate Types

1. **Exact Bitwise Duplicates:** Same `trade_id`, identical timestamps, matching execution price/shares. Caused by network retries or producer re-sends.
2. **Semantic / State Mutation Duplicates:** Same `trade_id` with differing status (`NEW` &rarr; `FILLED` &rarr; `SETTLED`). Downstream joins incorrectly treating state mutations as independent trade records.
3. **Execution ID Collisions:** Different trades accidentally assigned matching IDs by upstream gateways.

---

## 2. Step 2: Diagnostic SQL Query to Uncover Duplicate Patterns

```sql
SELECT 
    trade_id,
    COUNT(*) AS occurrence_count,
    COUNT(DISTINCT execution_timestamp) AS distinct_timestamps,
    COUNT(DISTINCT price) AS distinct_prices,
    MIN(ingestion_time) AS first_seen,
    MAX(ingestion_time) AS last_seen
FROM trade_clearing_stage
GROUP BY trade_id
HAVING COUNT(*) > 1
ORDER BY occurrence_count DESC;
```

---

## 3. Step 3: PySpark / Delta Lake Elimination Architecture

```
[ Trade Ingestion Stream ]
             │
             ▼ (Stage 1: Redis Distributed Lock / Bloom Filter)
   [ Drop Duplicate Message Payloads ]
             │
             ▼ (Stage 2: Microbatch Window Deduplication)
   [ Rank by `execution_timestamp` and `version` DESC ]
             │
             ▼ (Stage 3: Atomic Delta Lake Upsert)
   [ MERGE INTO target_trade_ledger ]
```

### PySpark Implementation:
```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

def clean_trade_batch(df_raw):
    # Window to select highest version and latest event timestamp
    window_spec = Window.partitionBy("trade_id").orderBy(
        F.col("version").desc(),
        F.col("event_timestamp").desc()
    )
    
    return (
        df_raw
        .withColumn("rank", F.row_number().over(window_spec))
        .filter(F.col("rank") == 1)
        .drop("rank")
    )
```

### Target Table Atomic Merge:
```sql
MERGE INTO curated_trade_ledger AS target
USING cleaned_trade_batch AS source
ON target.trade_id = source.trade_id
WHEN MATCHED AND source.version > target.version THEN
  UPDATE SET *
WHEN NOT MATCHED THEN
  INSERT *;
```
