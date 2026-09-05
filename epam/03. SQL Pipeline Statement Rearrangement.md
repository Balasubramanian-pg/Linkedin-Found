# SQL: Rearranging Shuffled SQL Clauses to Build a Valid Query

## Problem Statement
You are given shuffled SQL clauses. Rearrange them in the correct syntactic and logical execution sequence to join customer orders with product categories, filter completed orders from 2026, aggregate revenue by category, apply a HAVING threshold, and sort results in descending order.

---

## 1. Shuffled SQL Clauses (Problem Input)

```sql
Clause A: HAVING SUM(o.order_amount) >= 50000.00
Clause B: ORDER BY total_category_revenue DESC
Clause C: SELECT p.category_name, COUNT(o.order_id) AS total_orders, SUM(o.order_amount) AS total_category_revenue
Clause D: FROM orders o
Clause E: WHERE o.order_status = 'DELIVERED' AND o.order_date >= '2026-01-01'
Clause F: INNER JOIN products p ON o.product_id = p.product_id
Clause G: GROUP BY p.category_name
```

---

## 2. Correct Syntactic Clause Order

```
[ C. SELECT Column Projections & Aggregates ]
                  │
                  ▼
[ D. FROM Base Orders Table ]
                  │
                  ▼
[ F. INNER JOIN Products on Key ]
                  │
                  ▼
[ E. WHERE Row-Level Filters ]
                  │
                  ▼
[ G. GROUP BY Category ]
                  │
                  ▼
[ A. HAVING Aggregated Threshold ]
                  │
                  ▼
[ B. ORDER BY Final Sort ]
```

**Correct Syntactic Sequence:** `C` &rarr; `D` &rarr; `F` &rarr; `E` &rarr; `G` &rarr; `A` &rarr; `B`

---

## 3. Complete Valid SQL Query

```sql
SELECT 
    p.category_name,
    COUNT(o.order_id) AS total_orders,
    SUM(o.order_amount) AS total_category_revenue
FROM 
    orders o
INNER JOIN 
    products p ON o.product_id = p.product_id
WHERE 
    o.order_status = 'DELIVERED' 
    AND o.order_date >= '2026-01-01'
GROUP BY 
    p.category_name
HAVING 
    SUM(o.order_amount) >= 50000.00
ORDER BY 
    total_category_revenue DESC;
```

---

## 4. Logical Processing Order of the Engine

| Step | Clause Executed | Action Taken by SQL Optimizer |
| :--- | :--- | :--- |
| **1** | `FROM & JOIN` (`D, F`) | Loads datasets into memory and matches join keys `product_id`. |
| **2** | `WHERE` (`E`) | Filters individual rows before grouping (`order_status = 'DELIVERED'`). |
| **3** | `GROUP BY` (`G`) | Groups remaining rows into category buckets. |
| **4** | `HAVING` (`A`) | Discards grouped buckets where total revenue is below \$50,000. |
| **5** | `SELECT` (`C`) | Computes final projections and assigns column aliases. |
| **6** | `ORDER BY` (`B`) | Sorts output rows in descending order of revenue. |
