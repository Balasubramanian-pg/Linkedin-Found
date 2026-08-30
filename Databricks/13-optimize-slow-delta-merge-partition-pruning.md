# Optimizing Delta MERGE: Fixing 2 TB Scans for 20 GB Changes

## Question
Your daily Delta `MERGE INTO` operation scans and processes an entire 2 TB table every day, even though only 20 GB of records changed. How do you fix this to scan only the modified partitions?

---

## 1. Root Cause Analysis
Without a **partition pruning predicate** in the `ON` clause, the Delta engine cannot determine which physical files contain matching keys. It is forced to perform a full table scan of all 2 TB of Parquet data.

---

## 2. Solution: Add Partition Pruning in the Join Condition

### A. Suboptimal Merge (Full 2 TB Scan):
```sql
-- ❌ Scans all files across all years and months
MERGE INTO gold_transactions AS target
USING staged_updates AS source
ON target.transaction_id = source.transaction_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

### B. Optimized Merge with Dynamic Partition Pruning:
```sql
-- ✅ Scans ONLY partitions touched by the incoming 20 GB batch
MERGE INTO gold_transactions AS target
USING staged_updates AS source
ON target.transaction_id = source.transaction_id
   AND target.transaction_date = source.transaction_date -- Partition Pruning Predicate!
   AND target.transaction_date >= (SELECT MIN(transaction_date) FROM staged_updates)
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

---

## 3. Additional Optimizations
1. **Liquid Clustering / Z-ORDER:** Cluster the target table by `(transaction_date, transaction_id)`.
2. **Compact Target Files (`OPTIMIZE`):** Merge performance degrades exponentially when matching against thousands of tiny uncompacted files.
