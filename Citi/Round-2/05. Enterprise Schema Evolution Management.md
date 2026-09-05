# Enterprise Schema Evolution Management in Data Pipelines

## Question
How do you manage schema evolution and schema drift across enterprise financial pipelines without breaking downstream dependencies?

---

## 1. Schema Evolution Strategies in Delta Lake

### A. Automatic Schema Evolution (`mergeSchema`)
```python
# Automatically adds newly detected columns to target Delta table
(
    df_incoming
    .write
    .format("delta")
    .mode("append")
    .option("mergeSchema", "true")
    .saveAsTable("silver_transactions")
)
```

### B. Delta Column Mapping (Instant Metadata Column Rename)
```sql
ALTER TABLE silver_transactions SET TBLPROPERTIES (
  'delta.columnMapping.mode' = 'name',
  'delta.minReaderVersion' = '2',
  'delta.minWriterVersion' = '5'
);

-- Renames column in metadata in milliseconds without rewriting Parquet files
ALTER TABLE silver_transactions RENAME COLUMN tx_amt TO transaction_amount;
```

---

## 2. Governance Best Practices for Schema Changes
1. **Schema Contracts with Avro / Protobuf:** Enforce strict backward compatibility on Kafka topics.
2. **Compatibility Views:** When deprecating or renaming columns, maintain backward-compatible SQL Views with aliased column names for 90 days.
3. **Automated Schema Drift Alerts:** Trigger Microsoft Teams / Slack alerts whenever a new column is introduced.
