# Finding the 2nd Highest Salary in SQL: 3 Methods Compared

Finding the second highest salary is a common SQL interview question and a useful pattern for ranking and top-N problems. The correct solution depends on what “second highest” means:

- **Second highest distinct salary**: If two employees earn the same top salary, the next lower salary is second highest.
- **Second row when sorted by salary**: Duplicate top salaries occupy multiple positions.

Most business questions mean the **second highest distinct salary**. The three common methods below handle that definition, with different trade-offs.

---

## Sample data

```sql
CREATE TABLE employees (
    id      INT,
    name    VARCHAR(100),
    salary  DECIMAL(10, 2)
);

INSERT INTO employees (id, name, salary) VALUES
    (1, 'Alice', 100000),
    (2, 'Bob',   100000),
    (3, 'Carol',  90000),
    (4, 'Dave',   80000),
    (5, 'Eve',     NULL);
```

The second highest distinct salary is **90000**. Alice and Bob tie for highest, so the next lower salary is Carol’s.

---

## Method 1: Subquery with `MAX()`

Find the maximum salary that is strictly less than the overall maximum salary.

```sql
SELECT MAX(salary) AS second_highest_salary
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

### How it works

The inner query returns the highest salary. The outer query returns the maximum salary among rows below that value. Because the comparison is strict (`<`), all rows tied at the highest salary are excluded, and the next lower distinct salary is returned.

### Behavior

- Returns one row with a scalar value.
- Handles duplicate top salaries correctly.
- Returns `NULL` if there is no second highest salary.
- Ignores `NULL` salaries naturally because `MAX()` ignores nulls and `salary < ...` excludes nulls.
- Portable across almost all SQL databases.

### Performance

With an index on `salary`, both the inner `MAX()` and the outer range scan can use the index efficiently. Without an index, the table may be scanned twice. For a single lookup, this is usually fast.

### Limitation

This method returns only the salary value, not the employee row. To return the employee name, you need an additional join or a different method.

---

## Method 2: `ORDER BY` with `LIMIT` / `OFFSET`

Sort distinct salaries descending and skip the first row.

```sql
SELECT DISTINCT salary AS second_highest_salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

### Dialect variations

```sql
-- SQL Server
SELECT DISTINCT salary AS second_highest_salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY;

-- Oracle 12c+
SELECT DISTINCT salary AS second_highest_salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
OFFSET 1 ROWS FETCH NEXT 1 ROWS ONLY;

-- MySQL, PostgreSQL, SQLite
SELECT DISTINCT salary AS second_highest_salary
FROM employees
WHERE salary IS NOT NULL
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

### How it works

`DISTINCT` collapses duplicate salaries. `ORDER BY salary DESC` sorts them from highest to lowest. `LIMIT 1 OFFSET 1` skips the highest salary and returns the next one.

### Behavior

- Returns zero rows if there is no second highest salary.
- Handles duplicate top salaries correctly **only because of `DISTINCT`**.
- Without `DISTINCT`, a tie at the top causes `OFFSET 1` to return the same top salary again.
- Syntax is not fully portable; `LIMIT` and `OFFSET` differ by database.
- Filtering `salary IS NOT NULL` avoids null-ordering surprises. In some databases, `DESC` places nulls first by default.

### Performance

With an index on `salary`, this can be very efficient because the database can walk the index in descending order and stop after two distinct values. `DISTINCT` may still require a sort or unique operation. `OFFSET 1` is cheap; large offsets are more expensive, but that is not a concern here.

### Limitation

This method returns only the salary value. It is best for ad hoc queries or dialect-specific code where portability is not required.

---

## Method 3: `DENSE_RANK()` window function

Rank salaries densely, then filter for rank 2.

```sql
WITH ranked AS (
    SELECT
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT salary AS second_highest_salary
FROM ranked
WHERE salary_rank = 2;
```

If you need one scalar row even when multiple employees share the second highest salary:

```sql
WITH ranked AS (
    SELECT
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT DISTINCT salary AS second_highest_salary
FROM ranked
WHERE salary_rank = 2;
```

### How it works

`DENSE_RANK()` assigns rank 1 to the highest salary, rank 2 to the next distinct salary, and so on. Ties receive the same rank, but no rank values are skipped. Filtering `salary_rank = 2` returns the second highest distinct salary.

### Why `DENSE_RANK()` and not `ROW_NUMBER()` or `RANK()`

- `ROW_NUMBER()` numbers rows sequentially. If two employees tie for highest salary, `ROW_NUMBER()` gives them 1 and 2, so rank 2 would return the same top salary again.
- `RANK()` skips ranks after ties. If two employees tie for highest, both get rank 1, and the next salary gets rank 3. There is no rank 2.
- `DENSE_RANK()` gives the correct “second distinct salary” semantics.

### Behavior

- Returns the second highest distinct salary.
- Returns multiple rows if multiple employees share the second highest salary, unless you add `DISTINCT` or aggregate.
- Returns zero rows if there is no second highest salary.
- Can easily return full employee details by selecting additional columns in the CTE.

```sql
WITH ranked AS (
    SELECT
        id,
        name,
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT id, name, salary
FROM ranked
WHERE salary_rank = 2;
```

### Performance

Window functions generally require a sort of the data by `salary`. On large tables, this can be more expensive than the index-friendly subquery or `LIMIT` approach. However, `DENSE_RANK()` is the most flexible method when you need ranking, ties, or additional columns.

---

## Comparison table

| Method | Duplicate top salaries | Ties at second place | No second highest | Portability | Returns employee row easily | Relative performance |
|---|---|---|---|---|---|---|
| `MAX()` subquery | Correct | Returns one scalar value | Returns `NULL` | High | No, needs join | Fast with index |
| `DISTINCT` + `LIMIT/OFFSET` | Correct only with `DISTINCT` | Returns one distinct salary | Returns zero rows | Dialect-specific | No | Fast with index |
| `DENSE_RANK()` | Correct | Returns multiple rows unless `DISTINCT` | Returns zero rows | High in modern databases | Yes | Sort/window overhead |

---

## Common mistakes

### Forgetting `DISTINCT` with `LIMIT/OFFSET`

```sql
-- Wrong when the top salary is duplicated
SELECT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 1;
```

If Alice and Bob both earn 100000, this returns 100000 again, not 90000.

### Using `ROW_NUMBER()` instead of `DENSE_RANK()`

```sql
-- Wrong for "second highest distinct salary" when ties exist
WITH ranked AS (
    SELECT salary,
           ROW_NUMBER() OVER (ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT salary FROM ranked WHERE rn = 2;
```

With duplicate top salaries, this returns the same top salary.

### Using `RANK()` and filtering for rank 2

```sql
-- Returns nothing when the top salary is tied
WITH ranked AS (
    SELECT salary,
           RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
)
SELECT salary FROM ranked WHERE rnk = 2;
```

If two employees tie for highest, the next salary has rank 3, so no row has rank 2.

### Ignoring `NULL` salaries

`ORDER BY salary DESC` may place nulls first in some databases. Add `WHERE salary IS NOT NULL` when using `LIMIT/OFFSET` or window functions.

### Using `NOT IN` with a subquery that can return `NULL`

```sql
-- Risky if the subquery can produce NULL
SELECT MAX(salary)
FROM employees
WHERE salary NOT IN (SELECT MAX(salary) FROM employees);
```

If the subquery returns `NULL`, `NOT IN` can evaluate to unknown and return no rows. The strict comparison `salary < (SELECT MAX(salary) ...)` is safer.

---

## Which method should you use?

- **Use the `MAX()` subquery** when you need a simple, portable scalar answer and only the salary value matters.
- **Use `DISTINCT` with `LIMIT/OFFSET`** for quick ad hoc queries in a known database dialect, especially when an index on `salary` exists.
- **Use `DENSE_RANK()`** when you need deterministic ranking, want to return employee details, or need to generalize to the Nth highest salary.

For an Nth highest salary, the window-function pattern generalizes cleanly:

```sql
WITH ranked AS (
    SELECT
        salary,
        DENSE_RANK() OVER (ORDER BY salary DESC) AS salary_rank
    FROM employees
    WHERE salary IS NOT NULL
)
SELECT DISTINCT salary
FROM ranked
WHERE salary_rank = 2;  -- change 2 to N
```

The `MAX()` subquery does not generalize easily beyond the second highest, and the `LIMIT/OFFSET` method requires changing the offset. `DENSE_RANK()` is therefore the most maintainable choice for ranking problems in production SQL.
