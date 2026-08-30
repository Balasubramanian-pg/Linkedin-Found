# Delta Lake: ACID Transactions and Schema Evolution

## Question
How does Delta Lake guarantee ACID transactions over cloud storage, and how does it support zero-downtime schema evolution?

---

## 1. How Delta Lake Implements ACID Transactions

Delta Lake combines raw Apache Parquet data files with an ordered, append-only **Delta Transaction Log (`_delta_log/*.json`)**.

```
[ Delta Lake Directory Structure ]
├── _delta_log/
│   ├── 00000000000000000000.json   <-- Commit 0 (Table created)
│   ├── 00000000000000000001.json   <-- Commit 1 (Append Parquet files)
│   └── 00000000000000000010.checkpoint.parquet
├── part-00000-tid...snappy.parquet
└── part-00001-tid...snappy.parquet
```

- **Atomicity:** Commits are written atomically as single JSON files. If a write fails, no JSON log entry is written, and readers never see partial data.
- **Consistency:** Readers always query a consistent point-in-time table snapshot.
- **Isolation (Optimistic Concurrency Control - OCC):** Multiple concurrent writers verify that no conflicting files were modified before committing.

---

## 2. Schema Evolution vs Schema Enforcement

- **Schema Enforcement (Default):** Rejects any write that does not match the target table schema, preventing dirty data corruption.
- **Schema Evolution (`mergeSchema`):** Allows incoming new columns to be automatically appended to the target schema:
  ```python
  df.write.format("delta").mode("append").option("mergeSchema", "true").saveAsTable("silver_table")
  ```
