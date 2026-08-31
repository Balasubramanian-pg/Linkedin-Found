# The `_delta_log` Architecture: The Brain of Delta Lake

## Question
What is the `_delta_log/` directory, how does it store commits and checkpoints, and why is it essential for ACID compliance and Time Travel?

---

## 1. Structure of `_delta_log`

The `_delta_log` directory is an ordered, append-only transaction log containing JSON commits and Parquet checkpoints:

```
[ Table Root Directory: `/mnt/lake/tables/customers/` ]
├── _delta_log/
│   ├── 00000000000000000000.json           <-- Commit 0 (Schema, Initial Append)
│   ├── 00000000000000000001.json           <-- Commit 1 (MERGE: Add 2 files, Remove 1 file)
│   ├── ...
│   ├── 00000000000000000010.json           <-- Commit 10
│   └── 00000000000000000010.checkpoint.parquet  <-- Consolidated snapshot of Commits 0-10
├── part-00000-tid...snappy.parquet
└── part-00001-tid...snappy.parquet
```

---

## 2. Why `_delta_log` is Critical

1. **Single Source of Truth:** Dictates exactly which physical Parquet files constitute the current table snapshot.
2. **Enables Time Travel:** By replaying log entries up to Commit N, readers reconstruct historical table states:
   ```sql
   SELECT * FROM customers VERSION AS OF 5;
   ```
3. **Data Skipping Min/Max Statistics:** Every JSON commit stores min/max column values and null counts for every added file, allowing engines to skip reading irrelevant files.
4. **Checkpoint Compaction:** Every 10 commits, Delta writes a consolidated `.checkpoint.parquet` file so engines do not have to read hundreds of individual JSON files during startup.
