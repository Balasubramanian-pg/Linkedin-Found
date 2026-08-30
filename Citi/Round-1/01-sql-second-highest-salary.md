# SQL: Finding the Second Highest Salary

## Question
Write an SQL query to find the second highest salary from an `employees` table, handling duplicate salaries and NULL cases gracefully.

---

## 1. Approach 1: Window Function `DENSE_RANK()` (Recommended)

`DENSE_RANK()` assigns the same rank to identical salaries without skipping subsequent ranks:

```sql
WITH RankedSalaries AS (
    SELECT 
        emp_id,
        emp_name,
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT emp_id, emp_name, salary
FROM RankedSalaries
WHERE salary_rank = 2;
```

---

## 2. Approach 2: Using `MAX()` with Subquery (Universal ANSI SQL)

```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (
    SELECT MAX(salary) 
    FROM employees
);
```

---

## 3. Approach 3: Using `LIMIT` & `OFFSET` (MySQL, PostgreSQL, SQLite)

```sql
SELECT DISTINCT salary AS second_highest_salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

---

## 4. Second Highest Salary Per Department

```sql
WITH DeptRanks AS (
    SELECT 
        emp_id,
        emp_name,
        dept_id,
        salary,
        DENSE_RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT dept_id, emp_id, emp_name, salary
FROM DeptRanks
WHERE rnk = 2;
```
