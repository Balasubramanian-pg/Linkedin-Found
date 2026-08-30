# Comprehensive Guide to All SQL JOIN Types

## Question
Explain all types of SQL JOINs (`INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, `CROSS`, and `SELF JOIN`) with syntax and Venn diagrams.

---

## 1. Visual Overview of JOIN Types

```
  [ Table A ]          [ Table B ]
      ┌─────┬───────┬─────┐
      │     │ INNER │     │   --> INNER JOIN: Intersection only
      │LEFT │ JOIN  │RIGHT│   --> LEFT JOIN: All A + matched B
      │ ONLY│       │ ONLY│   --> RIGHT JOIN: All B + matched A
      └─────┴───────┴─────┘   --> FULL OUTER JOIN: All A + All B
```

---

## 2. Detailed Breakdown of Each JOIN

### 1. `INNER JOIN`
Returns rows that have matching values in **both** tables.
```sql
SELECT c.name, o.order_id, o.amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;
```

### 2. `LEFT JOIN` (LEFT OUTER JOIN)
Returns all rows from the **left** table and matched rows from the right table. If no match, right columns return `NULL`.
```sql
SELECT c.name, o.order_id
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```

### 3. `RIGHT JOIN` (RIGHT OUTER JOIN)
Returns all rows from the **right** table and matched rows from the left table.
```sql
SELECT c.name, o.order_id
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;
```

### 4. `FULL OUTER JOIN`
Returns all rows when there is a match in either left or right table. Missing values are filled with `NULL`.
```sql
SELECT c.name, o.order_id
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id;
```

### 5. `CROSS JOIN` (Cartesian Product)
Returns the Cartesian product of both tables ($M \times N$ rows). Does not use an `ON` clause.
```sql
SELECT p.product_name, s.store_location
FROM products p
CROSS JOIN stores s;
```

### 6. `SELF JOIN`
A regular join in which a table is joined with itself (e.g., finding employee managers).
```sql
SELECT 
    e.emp_name AS employee, 
    m.emp_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.emp_id;
```

---

## 3. Anti-Join Pattern (Left Excluding Join)
To find records in Table A that have **no match** in Table B:
```sql
SELECT c.customer_id, c.name
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```
