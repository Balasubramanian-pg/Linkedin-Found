# Difference Between WHERE and HAVING in SQL

## Question
What is the difference between `WHERE` and `HAVING` clauses in SQL?

---

## 1. Comparison Matrix

| Feature | `WHERE` Clause | `HAVING` Clause |
| :--- | :--- | :--- |
| **Execution Point** | Before `GROUP BY` and aggregation. | After `GROUP BY` and aggregation. |
| **Filtering Level** | Filters **individual rows** before grouping. | Filters **grouped/aggregated summary rows**. |
| **Aggregate Functions**| ❌ Cannot use aggregates (`SUM`, `AVG`, `COUNT`). | ✅ Specifically designed for aggregates (`COUNT(*) > 5`). |
| **Applicability** | Can be used with or without `GROUP BY`. | Typically requires `GROUP BY` (or table-level aggregates). |
| **Performance** | **Higher** (reduces row count before expensive grouping). | Slower if used for row-level filters that could have been in `WHERE`. |

---

## 2. Code Example

```sql
SELECT 
    dept_id,
    COUNT(emp_id) AS total_employees,
    AVG(salary) AS avg_salary
FROM employees
-- WHERE filters individual rows BEFORE grouping
WHERE status = 'ACTIVE' AND hire_date >= '2020-01-01'
GROUP BY dept_id
-- HAVING filters aggregated groups AFTER grouping
HAVING COUNT(emp_id) >= 5 AND AVG(salary) > 75000;
```

---

## 3. Best Practice Rule
> Always filter individual rows as early as possible using `WHERE` so that the database engine groups fewer rows in memory, using `HAVING` strictly for post-aggregation threshold filtering.
