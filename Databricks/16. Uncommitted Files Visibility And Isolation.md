# Uncommitted Files & Pipeline Failures: Visibility & Isolation

## Question
A Databricks pipeline writes 50 GB of Parquet files to ADLS Gen2, but fails due to a network error before completing its transaction. Are those written files visible to downstream readers? How are they cleaned up?

---

## 1. Are Uncommitted Files Visible to Readers?
**NO, absolutely not.**

In Delta Lake, readers do **not** read physical files directly by listing directories. Instead, readers inspect the **Delta Transaction Log (`_delta_log/`)** to construct the valid table snapshot.
- Because the pipeline failed before writing the commit JSON file (`_delta_log/000015.json`), the newly written Parquet files are never referenced in the active table state.
- Readers continue reading Snapshot `v14` as if the failed write never happened.

---

## 2. How are Orphan Uncommitted Files Cleaned Up?

```sql
-- Physically deletes uncommitted orphan files older than the retention threshold
VACUUM gold_table RETAIN 168 HOURS;
```
- During `VACUUM`, the Delta engine cross-references physical storage files against active valid transaction log pointers.
- Any unreferenced orphan files older than the safety threshold (default 7 days) are permanently deleted.
