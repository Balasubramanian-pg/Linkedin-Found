# SQL Deduplication Strategies: Retaining the Most Recent Record per Business Key

## Question
What are the different SQL strategies to deduplicate records and retain only the latest active record per business key (e.g., `cardholder_id` or `transaction_id`), and how do they perform on multi-billion row tables?

## 1. Strategy 1: Window Function `ROW_NUMBER()` (Universal & Most Versatile)

Assigns an sequential rank partitioned by the business key, ordered by the modification/event timestamp descending.

```sql
WITH RankedRecords AS (
    SELECT
        card_id,
        cardholder_name,
        card_status,
        credit_limit,
        updated_at,
        ROW_NUMBER() OVER (
            PARTITION BY card_id 
            ORDER BY updated_at DESC, ingestion_id DESC
        ) AS rn
    FROM 
        stg_card_updates
)
SELECT 
    card_id,
    cardholder_name,
    card_status,
    credit_limit,
    updated_at
FROM 
    RankedRecords
WHERE 
    rn = 1;
```

- **Pros:** Handles multi-column tie-breakers cleanly (`ingestion_id DESC`). Returns all columns without extra joins.
- **Cons:** Triggers an expensive distributed sort across all partition keys.

## 2. Strategy 2: `QUALIFY` Clause (Snowflake, Databricks, BigQuery, Teradata)

Syntactic sugar for `ROW_NUMBER()` that filters window functions directly without requiring an explicit CTE or subquery.

```sql
SELECT 
    card_id,
    cardholder_name,
    card_status,
    credit_limit,
    updated_at
FROM stg_card_updates
QUALIFY ROW_NUMBER() OVER (
    PARTITION BY card_id 
    ORDER BY updated_at DESC, ingestion_id DESC
) = 1;
```

## 3. Strategy 3: Group By `MAX(updated_at)` with Self-Join

Finds the latest timestamp per key, then joins back to retrieve remaining attribute columns.

```sql
WITH LatestTimestamps AS (
    SELECT 
        card_id,
        MAX(updated_at) AS max_updated_at
    FROM stg_card_updates
    GROUP BY card_id
)
SELECT 
    s.card_id,
    s.cardholder_name,
    s.card_status,
    s.credit_limit,
    s.updated_at
FROM stg_card_updates s
INNER JOIN LatestTimestamps lt
    ON s.card_id = lt.card_id 
   AND s.updated_at = lt.max_updated_at;
```

> [!WARNING]
> If a single `card_id` has duplicate records with the exact same `max_updated_at`, this approach will return **duplicate rows**. Use `ROW_NUMBER()` or add unique row ID tiebreakers if exact 1-row guarantee is needed.

## 4. Strategy 4: In-Place Table Deduplication (Physical Cleanup in DWH/Lakehouse)

### A. Delta Lake / PostgreSQL / SQL Server CTE Deletion
```sql
-- Delta Lake In-Place Merge Deduplication
MERGE INTO target_card_dim AS target
USING (
    SELECT * FROM stg_card_updates
    QUALIFY ROW_NUMBER() OVER (PARTITION BY card_id ORDER BY updated_at DESC) = 1
) AS source
ON target.card_id = source.card_id
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED THEN INSERT *;
```

### B. SQL Server / PostgreSQL Deletion via CTE:
```sql
WITH DuplicateRows AS (
    SELECT 
        card_id,
        ROW_NUMBER() OVER (PARTITION BY card_id ORDER BY updated_at DESC) AS rn
    FROM persistent_stage_table
)
DELETE FROM DuplicateRows WHERE rn > 1;
```

## 5. Performance Comparison Matrix

| Strategy | Performance on 100M+ Rows | Handles Duplicate Timestamps? | Engine Support |
| :--- | :--- | :--- | :--- |
| `ROW_NUMBER() = 1` | High (Fast with proper indexing / shuffle) | ✅ Yes (deterministic with tie-breaker) | Universal ANSI SQL |
| `QUALIFY ROW_NUMBER()` | Highest (Optimized plan pushdown) | ✅ Yes | Snowflake, Databricks, BigQuery |
| `GROUP BY + INNER JOIN` | Medium (Requires 2 scans of large table) | ❌ No (duplicates on identical max timestamp) | Universal ANSI SQL |
| `Delta MERGE INTO` | Best for Incremental Lakehouse Ingestion | ✅ Yes | Databricks, Apache Iceberg |
