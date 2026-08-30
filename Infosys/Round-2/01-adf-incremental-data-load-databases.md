# Designing ADF Pipelines for Incremental Data Loads Between Databases

## Question
How do you design an enterprise Azure Data Factory (ADF) pipeline to incrementally sync data between a source database and a target analytical data store using high-watermark patterns?

---

## 1. Watermark Pipeline Architecture

```
[ 1. Lookup Old Watermark ] ──> Query control table for last committed timestamp
               │
               ▼
[ 2. Lookup New Max Watermark ] ──> Query source database for `MAX(LastModifiedDate)`
               │
               ▼
[ 3. Copy Data Activity ] ──> Extract `WHERE LastModifiedDate > Old AND <= New`
               │
               ▼
[ 4. Stored Procedure Activity ] ──> Update control table with new watermark on success
```

---

## 2. ADF Control Table Setup in Azure SQL

```sql
CREATE TABLE WatermarkControl (
    TableName VARCHAR(100) PRIMARY KEY,
    WatermarkColumn VARCHAR(100),
    LastProcessedWatermark DATETIME2
);

INSERT INTO WatermarkControl VALUES ('SalesOrders', 'LastModifiedDate', '2026-01-01 00:00:00');
```

---

## 3. Dynamic Source Query in ADF Copy Activity

```sql
SELECT * 
FROM SourceDB.SalesOrders 
WHERE LastModifiedDate > '@{activity('LookupOldWatermark').output.firstRow.LastProcessedWatermark}'
  AND LastModifiedDate <= '@{activity('LookupNewWatermark').output.firstRow.NewMaxWatermark}'
```
