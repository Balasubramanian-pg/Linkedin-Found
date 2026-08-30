# Difference Between ROW_NUMBER(), RANK(), and DENSE_RANK()

## Question
What is the difference between `ROW_NUMBER()`, `RANK()`, and `DENSE_RANK()` in SQL window functions?

---

## 1. Summary Comparison

- **`ROW_NUMBER()`:** Assigns a unique, consecutive integer to every row, breaking ties arbitrarily or by secondary order columns.
- **`RANK()`:** Assigns the same rank to identical values, but **skips** subsequent rank positions (e.g., `1, 2, 2, 4`).
- **`DENSE_RANK()`:** Assigns the same rank to identical values, but **does not skip** rank positions (e.g., `1, 2, 2, 3`).

---

## 2. Code Example & Output Comparison

```sql
SELECT 
    emp_name,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) AS row_num,
    RANK() OVER (ORDER BY salary DESC) AS rank_val,
    DENSE_RANK() OVER (ORDER BY salary DESC) AS dense_rank_val
FROM employees;
```

### Result Table:
| emp_name | salary | `ROW_NUMBER()` | `RANK()` | `DENSE_RANK()` | Explanation |
| :--- | :--- | :--- | :--- | :--- | :--- |
| Alice | $120,000 | 1 | 1 | 1 | Highest salary |
| Bob | $100,000 | 2 | 2 | 2 | Tie for 2nd |
| Charlie | $100,000 | 3 | 2 | 2 | Tie for 2nd |
| David | $90,000 | 4 | **4** (Skipped 3) | **3** (No skip) | Next distinct salary |
| Emma | $80,000 | 5 | 5 | 4 | |

---

## 3. When to Use Which?
- Use **`ROW_NUMBER()`** when you need strict pagination or deduplication (retaining exactly 1 row per key).
- Use **`RANK()`** in sports or competition leaderboards where ties push subsequent places down.
- Use **`DENSE_RANK()`** for finding Nth highest/lowest business metrics (e.g., 2nd highest distinct salary).
