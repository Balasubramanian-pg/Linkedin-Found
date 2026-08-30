# Zero-Downtime Schema Evolution on Multi-Billion Row Tables

## Question
How do you implement **zero-downtime schema evolution** (e.g., adding nullable columns, renaming fields, or modifying data types) on multi-billion row tables in Delta Lake / Databricks without locking tables or breaking active downstream queries?

---

## 1. The Challenges of Schema Evolution at Big Data Scale

In traditional relational databases, running `ALTER TABLE ADD COLUMN` on a table with 5 billion rows triggers an exclusive table-level lock, causes query timeouts, and may rewrite massive physical data files on disk.

In **Delta Lake**, schema updates are **pure metadata operations** that execute in milliseconds without modifying physical Parquet files.

```
Delta Schema Evolution:
[ Physical Parquet Files v1 ] ──> (Columns: id, name, amount)
                │
                ▼ (Execute: ALTER TABLE ADD COLUMN loyalty_tier STRING)
[ _delta_log/000025.json ]    ──> (Schema updated in metadata commit)
                │
[ Reader Query for loyalty_tier ]
  • Existing Parquet files (v1): Engine dynamically returns NULL for loyalty_tier
  • New Parquet files (v2): Engine reads written physical loyalty_tier values
```

---

## 2. Approach 1: In-Place Schema Evolution via Delta Lake `mergeSchema`

When new columns arrive in incoming streaming or batch DataFrames:

```python
# Automatic Schema Evolution during write/merge
(
    df_incoming_new_columns
    .write
    .format("delta")
    .mode("append")
    .option("mergeSchema", "true")  # Dynamically adds missing columns to target schema
    .saveAsTable("silver_transactions")
)
```

### In Delta `MERGE INTO`:
```sql
SET spark.databricks.delta.schema.autoMerge.enabled = true;

MERGE INTO silver_transactions AS target
USING staged_updates AS source
ON target.tx_id = source.tx_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

---

## 3. Approach 2: Explicit DDL Evolution (Instant Metadata Update)

Delta Lake supports instant, zero-lock metadata operations:

```sql
-- 1. Add new nullable column (O(1) execution time - zero physical file rewrites)
ALTER TABLE silver_transactions 
ADD COLUMNS (
    loyalty_tier STRING COMMENT 'Cardholder loyalty tier',
    rewards_earned DECIMAL(10, 2) COMMENT 'Points earned on transaction'
);

-- 2. Add nested column to existing struct
ALTER TABLE silver_transactions 
ADD COLUMNS (
    merchant.sub_category STRING
);
```

---

## 4. Advanced: Zero-Downtime Column Renaming and Type Widening

### A. Column Renaming with Column Mapping
In legacy Delta, renaming a column required full table rewriting. Delta Column Mapping decouples physical parquet column names from logical table column names:

```sql
-- Enable Column Mapping metadata mode
ALTER TABLE silver_transactions 
SET TBLPROPERTIES (
  'delta.columnMapping.mode' = 'name',
  'delta.minReaderVersion' = '2',
  'delta.minWriterVersion' = '5'
);

-- Instant zero-copy column rename
ALTER TABLE silver_transactions 
RENAME COLUMN tx_amt TO transaction_amount;
```

### B. Type Widening (Zero-Copy Automatic Promotion)
Allows widening numeric types without rewriting underlying parquet data:
- `BYTE` &rarr; `SHORT` &rarr; `INT` &rarr; `BIGINT`
- `FLOAT` &rarr; `DOUBLE`
- `DATE` &rarr; `TIMESTAMP`

```sql
ALTER TABLE silver_transactions 
ALTER COLUMN transaction_id TYPE BIGINT;
```

---

## 5. Schema Evolution Safety Rules Matrix

| Evolution Action | Delta Operation Type | Breaks Downstream Readers? | Action Required |
| :--- | :--- | :--- | :--- |
| **Add Nullable Column** | Instant Metadata Update | ❌ No | Add via `ALTER TABLE` or `mergeSchema` |
| **Rename Column** | Column Mapping (Instant) | ⚠️ Only if queries reference old name | Create compatibility SQL View with aliases |
| **Drop Column** | Metadata Soft-Drop | ⚠️ Only if queries reference dropped col | Ensure downstream deprecation window |
| **Narrow Data Type (e.g. BIGINT &rarr; INT)** | Dangerous (Potential Overflow) | ❌ Disallowed | Create new column with migration phase |
