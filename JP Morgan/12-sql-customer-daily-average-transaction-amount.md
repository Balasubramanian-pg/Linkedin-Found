# SQL: Calculating Customer Daily Average Transaction Amount

## Question
Write an optimized SQL query to calculate the daily average transaction amount for each customer, along with their overall historical daily average and variance, handling days with zero activity.

---

## 1. Solution: Aggregation with Window Metrics

```sql
WITH CustomerDailySpend AS (
    -- Step 1: Calculate total spend and count per customer per active day
    SELECT
        customer_id,
        CAST(transaction_timestamp AS DATE) AS transaction_date,
        SUM(transaction_amount) AS total_daily_amount,
        COUNT(transaction_id) AS total_daily_transactions,
        AVG(transaction_amount) AS avg_amount_per_transaction
    FROM 
        fact_customer_transactions
    WHERE 
        status = 'COMPLETED'
    GROUP BY 
        customer_id, 
        CAST(transaction_timestamp AS DATE)
)
SELECT
    customer_id,
    transaction_date,
    total_daily_amount,
    total_daily_transactions,
    ROUND(avg_amount_per_transaction, 2) AS daily_avg_per_tx,
    -- Customer's historical running average daily spend up to current date
    ROUND(
        AVG(total_daily_amount) OVER (
            PARTITION BY customer_id
            ORDER BY transaction_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ), 
        2
    ) AS historical_running_avg_daily_spend,
    -- Difference from historical baseline (Anomaly Indicator)
    ROUND(
        total_daily_amount - AVG(total_daily_amount) OVER (
            PARTITION BY customer_id
        ),
        2
    ) AS deviation_from_lifetime_daily_avg
FROM 
    CustomerDailySpend
ORDER BY 
    customer_id, 
    transaction_date DESC;
```

---

## 2. Output Schema & Explanation

| Column | Meaning | Use Case |
| :--- | :--- | :--- |
| `daily_avg_per_tx` | Average transaction ticket size for that specific day. | Retail transaction sizing |
| `historical_running_avg_daily_spend`| Cumulative baseline daily spend for the cardholder. | Baseline profiling |
| `deviation_from_lifetime_daily_avg`| Spike in spending relative to customer normal habits. | Fraud detection trigger |
