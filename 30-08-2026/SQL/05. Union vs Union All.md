# Difference Between UNION vs UNION ALL in SQL

## Question
What is the difference between `UNION` and `UNION ALL` in SQL, and when should you choose one over the other?

---

## 1. Key Differences

| Feature | `UNION` | `UNION ALL` |
| :--- | :--- | :--- |
| **Duplicates** | Removes all duplicate rows from the combined result. | Retains **all** rows including exact duplicates. |
| **Sorting Overhead**| Triggers an expensive distinct/sorting operation in memory. | Zero sorting; simply appends dataset streams. |
| **Performance** | **Slower** due to deduplication cost. | **Significantly faster**. |
| **Memory Usage** | High (must track distinct rows across result sets). | Low (streamlined streaming append). |

---

## 2. SQL Syntax & Example

```sql
-- UNION (Removes duplicates)
SELECT customer_id, email FROM online_customers
UNION
SELECT customer_id, email FROM store_customers;

-- UNION ALL (Preserves duplicates, much faster)
SELECT transaction_id, amount FROM 2026_q1_sales
UNION ALL
SELECT transaction_id, amount FROM 2026_q2_sales;
```

---

## 3. Mandatory Requirements for Both:
1. Each `SELECT` query must have the **same number of columns**.
2. The columns must have **compatible data types** in the corresponding order.
3. Column names in the final result set are determined by the **first** `SELECT` query.

---

## 4. Best Practice
> Default to `UNION ALL` whenever you know datasets are mutually exclusive (e.g., merging separate quarterly archive tables) or when duplicate preservation is acceptable. Use `UNION` only when duplicate elimination is strictly required.
