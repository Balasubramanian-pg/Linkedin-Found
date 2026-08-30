# Top 2 Highest Salaries Per Department

## Question
Write SQL to find the top 2 highest salaries per department.

---

## Solution 1: Using `DENSE_RANK()` (Standard & Recommended)

`DENSE_RANK()` handles ties gracefully without skipping rank numbers (e.g., if two employees tie for 1st place, the next salary receives rank 2).

```sql
WITH RankedSalaries AS (
    SELECT
        emp_id,
        emp_name,
        dept_id,
        salary,
        DENSE_RANK() OVER (
            PARTITION BY dept_id 
            ORDER BY salary DESC
        ) AS salary_rank
    FROM 
        employees
)
SELECT 
    emp_id,
    emp_name,
    dept_id,
    salary
FROM 
    RankedSalaries
WHERE 
    salary_rank <= 2
ORDER BY 
    dept_id, 
    salary DESC;
```

---

## Solution 2: Using `ROW_NUMBER()` (Strict 2 Records per Department)

If the requirement strictly asks for at most 2 records per department regardless of salary ties:

```sql
WITH NumberedEmployees AS (
    SELECT
        emp_id,
        emp_name,
        dept_id,
        salary,
        ROW_NUMBER() OVER (
            PARTITION BY dept_id 
            ORDER BY salary DESC, emp_id ASC
        ) AS rn
    FROM 
        employees
)
SELECT 
    emp_id,
    emp_name,
    dept_id,
    salary
FROM 
    NumberedEmployees
WHERE 
    rn <= 2
ORDER BY 
    dept_id, 
    salary DESC;
```

---

## Key Window Functions Comparison

| Function | Tie Behavior | Next Rank After Tie (1, 1) | Best For |
| :--- | :--- | :--- | :--- |
| `ROW_NUMBER()` | Assigns arbitrary distinct numbers (1, 2) | N/A | Strict limit of N rows |
| `RANK()` | Same rank for ties (1, 1) | Skips to 3 | Top N with ranking gaps |
| `DENSE_RANK()` | Same rank for ties (1, 1) | Continues to 2 | Distinct top N values |

---

## Example Walkthrough

### Sample Data (`employees`)
| emp_id | emp_name | dept_id | salary |
| :--- | :--- | :--- | :--- |
| 101 | Alice | IT | 120,000 |
| 102 | Bob | IT | 120,000 |
| 103 | Charlie | IT | 100,000 |
| 104 | David | IT | 80,000 |
| 201 | Emma | HR | 95,000 |
| 202 | Frank | HR | 90,000 |
| 203 | Grace | HR | 75,000 |

### `DENSE_RANK()` Output (Returns distinct top 2 salary levels)
| emp_id | emp_name | dept_id | salary | salary_rank |
| :--- | :--- | :--- | :--- | :--- |
| 101 | Alice | IT | 120,000 | 1 |
| 102 | Bob | IT | 120,000 | 1 |
| 103 | Charlie | IT | 100,000 | 2 |
| 201 | Emma | HR | 95,000 | 1 |
| 202 | Frank | HR | 90,000 | 2 |
