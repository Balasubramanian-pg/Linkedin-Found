# SQL: Finding Consecutive Numbers Using LAG and LEAD

## Question
Write an SQL query to find all numbers that appear at least 3 consecutive times in a logs/transactions table using `LAG()` and `LEAD()` window functions.

---

## 1. Solution: Using `LAG()` and `LEAD()`

```sql
WITH ConsecutiveAnalysis AS (
    SELECT
        id,
        num,
        -- Previous number in sequence
        LAG(num, 1) OVER (ORDER BY id) AS prev_num,
        -- Subsequent number in sequence
        LEAD(num, 1) OVER (ORDER BY id) AS next_num
    FROM 
        logs
)
SELECT DISTINCT 
    num AS consecutive_num
FROM 
    ConsecutiveAnalysis
WHERE 
    num = prev_num 
    AND num = next_num;
```

---

## 2. Solution: Using Dual `LAG()` Lookback

If evaluating strictly against preceding rows:

```sql
WITH PriorValues AS (
    SELECT
        id,
        num,
        LAG(num, 1) OVER (ORDER BY id) AS prev_1,
        LAG(num, 2) OVER (ORDER BY id) AS prev_2
    FROM logs
)
SELECT DISTINCT num AS consecutive_num
FROM PriorValues
WHERE num = prev_1 AND num = prev_2;
```

---

## 3. Generalized Island-and-Gaps Technique (for N Consecutive Rows)

When finding sequences of arbitrary length (N >= 5 consecutive occurrences):

```sql
WITH IslandGroups AS (
    SELECT
        id,
        num,
        id - ROW_NUMBER() OVER (PARTITION BY num ORDER BY id) AS island_id
    FROM logs
)
SELECT 
    num,
    COUNT(*) AS consecutive_count
FROM IslandGroups
GROUP BY num, island_id
HAVING COUNT(*) >= 3;
```
