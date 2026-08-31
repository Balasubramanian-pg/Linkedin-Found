# Triaging Query Degradation on a 30 TB Delta Table (5s to 2 mins)

## Question
A mission-critical 30 TB Delta table used to serve queries in 5 seconds. Over the past 6 months, query response time has degraded to 2 minutes. How do you investigate and restore sub-second performance?

---

## 1. Investigation Triage Steps

```
[ 30 TB Query Degradation: 5s ──> 2m ]
                     │
                     ▼
[ 1. Check Delta Log History & File Counts (`DESCRIBE DETAIL`) ]
       ├── Thousands of small uncompacted files? ──> Run `OPTIMIZE`
       └── Millions of old tombstoned files?     ──> Run `VACUUM`
                     │
                     ▼
[ 2. Check Data Skipping Column Statistics ]
       └── Is the query filtering on columns past the 32nd position?
                     │
                     ▼
[ 3. Evaluate Partitioning Strategy ]
       └── Replace fragmented physical partitions with Liquid Clustering
```

---

## 2. Restoration Actions

### Action 1: Increase Indexed Column Statistics Limit
By default, Delta only collects min/max stats for the first 32 columns:
```sql
ALTER TABLE fact_large_table SET TBLPROPERTIES (
  'delta.dataSkippingNumIndexedCols' = '64'
);
```

### Action 2: Run Full Compaction & Z-Order / Clustering
```sql
OPTIMIZE fact_large_table ZORDER BY (query_filter_col1, join_key2);
VACUUM fact_large_table RETAIN 168 HOURS;
```

### Action 3: Upgrade to Photon Vectorized Engine on Databricks SQL
Enable Databricks Serverless SQL with Photon for 4x faster scans across multi-terabyte datasets.
