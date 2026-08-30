# SQL: Finding the Nth Highest Salary (Multiple Approaches)

## Question
Write SQL queries to find the Nth highest salary in an employee table using multiple different SQL approaches.

---

## 1. Approach 1: Using `DENSE_RANK()` (Recommended & Universal)

```sql
WITH RankedSalaries AS (
    SELECT 
        emp_id,
        emp_name,
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS rank_num
    FROM employees
    WHERE salary IS NOT NULL
)
-- Substitute N with desired rank (e.g. N = 3 for 3rd highest)
SELECT emp_id, emp_name, salary
FROM RankedSalaries
WHERE rank_num = 3;
```

---

## 2. Approach 2: Correlated Subquery (Universal ANSI SQL)

```sql
-- Finds the salary where exactly (N-1) distinct salaries are greater
SELECT DISTINCT e1.salary
FROM employees e1
WHERE (
    SELECT COUNT(DISTINCT e2.salary)
    FROM employees e2
    WHERE e2.salary > e1.salary
) = (3 - 1); -- For N=3
```

---

## 3. Approach 3: Using `LIMIT` & `OFFSET` (MySQL / PostgreSQL / SQLite)

```sql
SELECT DISTINCT salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 1 OFFSET (N - 1); -- For 3rd highest: LIMIT 1 OFFSET 2
```

---

## 4. Approach 4: Nested `MAX()` Subqueries (For 2nd or 3rd Highest)

```sql
-- 3rd Highest Salary
SELECT MAX(salary) AS third_highest
FROM employees
WHERE salary < (
    SELECT MAX(salary) 
    FROM employees 
    WHERE salary < (SELECT MAX(salary) FROM employees)
);
```
