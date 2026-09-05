# SQL Query Execution Order

## Question
Explain the logical query processing order (execution order) of an SQL query.

---

## 1. Written Order vs Logical Execution Order

While an SQL query is written in a specific syntax order, the database engine processes clauses in a completely different logical sequence.

```
Syntactic (Written) Order:         Logical Execution Order:
1. SELECT                         1. FROM & JOIN (Determine dataset & Cartesian product)
2. DISTINCT                       2. ON (Apply join filter conditions)
3. FROM & JOIN                    3. WHERE (Filter individual rows)
4. WHERE                          4. GROUP BY (Aggregate rows into groups)
5. GROUP BY                       5. HAVING (Filter aggregated groups)
6. HAVING                         6. SELECT (Evaluate expressions & column list)
7. WINDOW FUNCTIONS               7. DISTINCT (Eliminate duplicate output rows)
8. UNION / UNION ALL              8. UNION / INTERSECT / EXCEPT (Set operations)
9. ORDER BY                       9. ORDER BY (Sort final result set)
10. LIMIT / OFFSET                10. LIMIT / TOP / OFFSET (Pagination / row truncation)
```

---

## 2. Why Execution Order Matters (Interview Pitfalls)

### A. Column Aliases in `WHERE` Clauses Fail
```sql
-- ❌ Fails with error: 'annual_salary' does not exist
SELECT emp_name, salary * 12 AS annual_salary
FROM employees
WHERE annual_salary > 100000;
```
> **Reason:** `WHERE` (Step 3) executes **before** `SELECT` (Step 6), so the engine has not yet defined the alias `annual_salary`.

### B. Aggregates in `WHERE` Clauses Fail
```sql
-- ❌ Fails: Cannot use aggregate function in WHERE
SELECT dept_id, AVG(salary)
FROM employees
WHERE AVG(salary) > 50000
GROUP BY dept_id;
```
> **Reason:** Aggregation occurs during `GROUP BY` (Step 4). Use `HAVING` (Step 5) to filter aggregate results.
