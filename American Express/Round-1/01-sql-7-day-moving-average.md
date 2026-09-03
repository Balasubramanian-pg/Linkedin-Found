# Calculating 7-Day Moving Average on High-Volume Fact Tables

## Question
How do you calculate a 7-day moving average (and moving sum) on a high-volume financial transaction fact table using SQL window functions, ensuring correct handling of calendar gaps?

## 1. The Challenge of Calendar Gaps in High-Volume Tables

> [!WARNING]
> Using `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` only works if **every single day has at least one transaction**. If a customer has no transactions on weekends or holidays, a 6-row offset will calculate a moving average across 6 distinct active days (which might span 2 to 3 calendar weeks), producing inaccurate financial metrics!

To calculate a true **calendar 7-day moving average**, use either:
1. `RANGE BETWEEN INTERVAL '6' DAY PRECEDING AND CURRENT ROW` (Supported in PostgreSQL, BigQuery, Snowflake, Databricks SQL).
2. A **Date Spine (Calendar Master)** left-joined with daily aggregates (Universal ANSI SQL standard).

## 2. Approach 1: Pre-Aggregated Daily Fact Table with Date Spine (ANSI SQL Standard)

For high-volume transaction tables with billions of rows, always pre-aggregate to the daily level per entity before computing window aggregates to prevent memory exhaustion and skew.

```sql
WITH DailyCustomerSales AS (
    -- Step 1: Pre-aggregate millions of raw transactions to 1 row per cardholder per day
    SELECT
        card_id,
        CAST(transaction_timestamp AS DATE) AS transaction_date,
        SUM(transaction_amount) AS daily_amount,
        COUNT(transaction_id) AS daily_tx_count
    FROM 
        fact_card_transactions
    WHERE 
        transaction_timestamp >= '2026-01-01'
    GROUP BY 
        card_id, 
        CAST(transaction_timestamp AS DATE)
),
CalendarRange AS (
    -- Step 2: Generate contiguous calendar dates to eliminate date gaps
    SELECT DISTINCT 
        d.calendar_date,
        c.card_id
    FROM dim_date d
    CROSS JOIN (SELECT DISTINCT card_id FROM DailyCustomerSales) c
    WHERE d.calendar_date BETWEEN '2026-01-01' AND '2026-08-30'
),
DenseDailySales AS (
    -- Step 3: Densify data so missing transaction days have 0.00 amount
    SELECT
        cr.card_id,
        cr.calendar_date,
        COALESCE(dcs.daily_amount, 0.00) AS daily_amount,
        COALESCE(dcs.daily_tx_count, 0) AS daily_tx_count
    FROM CalendarRange cr
    LEFT JOIN DailyCustomerSales dcs
        ON cr.card_id = dcs.card_id 
       AND cr.calendar_date = dcs.transaction_date
)
-- Step 4: Compute exact 7-Day Moving Metrics
SELECT
    card_id,
    calendar_date,
    daily_amount,
    -- 7-Day Moving Average (Current Day + 6 Preceding Days)
    AVG(daily_amount) OVER (
        PARTITION BY card_id
        ORDER BY calendar_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d,
    -- 7-Day Moving Sum (Spend Velocity)
    SUM(daily_amount) OVER (
        PARTITION BY card_id
        ORDER BY calendar_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_sum_7d,
    -- 7-Day Moving Transaction Count (Fraud Signal)
    SUM(daily_tx_count) OVER (
        PARTITION BY card_id
        ORDER BY calendar_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_tx_count_7d
FROM DenseDailySales
ORDER BY card_id, calendar_date;
```

## 3. Approach 2: Using `RANGE` Windowing (Databricks / Snowflake / Postgres)

If your SQL engine natively supports datetime range intervals:

```sql
WITH DailyAggregates AS (
    SELECT
        card_id,
        CAST(transaction_timestamp AS DATE) AS tx_date,
        SUM(transaction_amount) AS daily_amount
    FROM fact_card_transactions
    GROUP BY card_id, CAST(transaction_timestamp AS DATE)
)
SELECT
    card_id,
    tx_date,
    daily_amount,
    AVG(daily_amount) OVER (
        PARTITION BY card_id
        ORDER BY CAST(tx_date AS TIMESTAMP)
        RANGE BETWEEN INTERVAL 6 DAYS PRECEDING AND CURRENT ROW
    ) AS moving_avg_7d
FROM DailyAggregates;
```

## 4. Query Plan & Performance Tuning on TB-Scale Tables
1. **Pre-Aggregation (Pushdown):** Aggregating before windowing reduces the working row count by 100x to 1000x.
2. **Partitioning & Clustering:** Ensure the underlying fact table is partitioned by `transaction_date` and clustered/Z-Ordered by `card_id`.
3. **Avoid Unbounded Frames:** Always explicitly declare `ROWS BETWEEN ...` or `RANGE BETWEEN ...` to prevent SQL engines from defaulting to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, which incurs heavy spooling.
