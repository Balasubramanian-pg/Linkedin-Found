# Difference Between RANK(), DENSE_RANK(), and ROW_NUMBER()

## Question
Explain the differences between `RANK()`, `DENSE_RANK()`, and `ROW_NUMBER()` in SQL with practical examples and tie-breaking behavior.

---

## 1. Comparison Matrix

| Function | Tie Handling | Rank Sequence for Values `[100, 100, 80]` | Primary Use Case |
| :--- | :--- | :--- | :--- |
| **`ROW_NUMBER()`** | Assigns unique consecutive integers arbitrarily. | `1, 2, 3` | Strict pagination, deduplication (`rn = 1`). |
| **`RANK()`** | Same rank for ties; **skips** next rank numbers. | `1, 1, 3` (3rd place skips 2). | Leaderboards, Olympic-style medal ranking. |
| **`DENSE_RANK()`** | Same rank for ties; **no skips** in sequence. | `1, 1, 2` | Finding Nth highest distinct business metrics. |

---

## 2. Code Example

```sql
SELECT 
    emp_name,
    dept_id,
    salary,
    ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS row_num,
    RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rank_val,
    DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS dense_rank_val
FROM employees;
```

### Result Sample:
| emp_name | salary | `ROW_NUMBER()` | `RANK()` | `DENSE_RANK()` |
| :--- | :--- | :--- | :--- | :--- |
| John (Lead) | $150,000 | 1 | 1 | 1 |
| Alice (Senior) | $120,000 | 2 | 2 | 2 |
| Bob (Senior) | $120,000 | 3 | 2 | 2 |
| Charlie (Mid) | $100,000 | 4 | **4** (Skipped 3) | **3** (No skip) |
