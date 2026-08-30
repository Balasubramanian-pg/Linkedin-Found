# How to Remove Duplicate Records in SQL

## Question
How do you identify and remove duplicate records from an SQL table?

---

## 1. Identifying Duplicate Rows

```sql
SELECT 
    customer_id, 
    email, 
    COUNT(*) AS occurrence_count
FROM customers
GROUP BY customer_id, email
HAVING COUNT(*) > 1;
```

---

## 2. Method 1: Using CTE with `ROW_NUMBER()` (Universal Standard)

```sql
WITH RankedDuplicates AS (
    SELECT 
        customer_id,
        email,
        updated_at,
        ROW_NUMBER() OVER (
            PARTITION BY email 
            ORDER BY updated_at DESC, customer_id DESC
        ) AS rn
    FROM customers
)
DELETE FROM RankedDuplicates
WHERE rn > 1;
```
> Retains the latest row (`rn = 1`) and deletes all older duplicates (`rn > 1`).

---

## 3. Method 2: Self-Join Deletion (MySQL / PostgreSQL)

```sql
DELETE c1
FROM customers c1
INNER JOIN customers c2 
    ON c1.email = c2.email 
   AND c1.customer_id < c2.customer_id; -- Retains highest ID
```

---

## 4. Method 3: Recreating Clean Table (Fastest for Huge Tables)

```sql
-- 1. Insert distinct records into temporary table
CREATE TABLE customers_temp AS
SELECT DISTINCT * FROM customers;

-- 2. Truncate original table
TRUNCATE TABLE customers;

-- 3. Restore clean records
INSERT INTO customers
SELECT * FROM customers_temp;

-- 4. Clean up temporary table
DROP TABLE customers_temp;
```
