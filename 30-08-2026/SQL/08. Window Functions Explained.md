# Understanding SQL Window Functions

## Question
What are SQL Window Functions, how do they differ from `GROUP BY`, and what are the main categories of window functions?

---

## 1. Window Functions vs `GROUP BY`

- **`GROUP BY`:** Collapses multiple rows into a single summary row per group.
- **Window Functions (`OVER ()`):** Compute aggregate or analytical metrics across a group of rows (window) while **retaining individual row identities**.

```
Input: 5 Rows ──[ GROUP BY ]──────────> Output: 2 Summary Rows
Input: 5 Rows ──[ OVER(PARTITION BY) ]──> Output: 5 Rows (with attached aggregate values)
```

---

## 2. Syntax Structure

```sql
FUNCTION_NAME() OVER (
    PARTITION BY partition_column   -- Grouping scope
    ORDER BY sort_column            -- Ordering within group
    ROWS/RANGE BETWEEN ...          -- Window frame definition
)
```

---

## 3. Major Categories of Window Functions

### A. Ranking Functions
- `ROW_NUMBER()`: Sequential row index.
- `RANK()`: Rank with gaps for ties.
- `DENSE_RANK()`: Rank without gaps for ties.
- `NTILE(n)`: Divides rows into N equal quartiles/buckets.

### B. Value / Offset Functions
- `LEAD(col, offset)`: Accesses subsequent row values.
- `LAG(col, offset)`: Accesses preceding row values (useful for Year-over-Year calculations).
- `FIRST_VALUE(col)` / `LAST_VALUE(col)`: Accesses first/last boundary value.

### C. Aggregate Window Functions
- `SUM()`, `AVG()`, `MIN()`, `MAX()`, `COUNT()` computed as running/moving totals.

---

## 4. Practical Real-World Example: Month-over-Month Growth

```sql
WITH MonthlyRevenue AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS order_month,
        SUM(amount) AS monthly_sales
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
)
SELECT
    order_month,
    monthly_sales,
    -- Previous month's sales
    LAG(monthly_sales, 1) OVER (ORDER BY order_month) AS prev_month_sales,
    -- Month-over-Month Growth Rate (%)
    ROUND(
        (monthly_sales - LAG(monthly_sales, 1) OVER (ORDER BY order_month)) * 100.0 / 
        LAG(monthly_sales, 1) OVER (ORDER BY order_month), 
        2
    ) AS mom_growth_pct,
    -- Running Total Year-to-Date
    SUM(monthly_sales) OVER (ORDER BY order_month) AS running_total_ytd
FROM MonthlyRevenue;
```
