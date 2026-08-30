# Difference Between DELETE, TRUNCATE, and DROP in SQL

## Question
What are the differences between `DELETE`, `TRUNCATE`, and `DROP` commands in SQL?

---

## 1. Comparison Matrix

| Feature | `DELETE` | `TRUNCATE` | `DROP` |
| :--- | :--- | :--- | :--- |
| **Command Category** | **DML** (Data Manipulation) | **DDL** (Data Definition) | **DDL** (Data Definition) |
| **Operation** | Deletes specific or all rows. | Deletes all rows by deallocating data pages. | Completely destroys table structure and data. |
| **`WHERE` Clause Support** | ✅ Yes (`WHERE id = 5`). | ❌ No (always removes all rows). | ❌ No. |
| **Transaction Logging** | Row-by-row logging (Slow). | Minimal page deallocation logging (Fast). | Minimal schema drop logging. |
| **Performance** | Slow for large tables. | **Extremely fast**. | Instantaneous. |
| **Identity Reset** | Does **not** reset auto-increment counter. | **Resets** auto-increment/identity to seed (1). | Table object deleted entirely. |
| **Rollback Capability** | Fully rollbackable inside transactions. | Rollbackable in most RDBMS (SQL Server/Postgres) if in transaction. | Rollbackable in Postgres/SQL Server if in transaction; not in MySQL. |
| **Triggers** | Fires `DELETE` triggers. | Does **not** fire triggers. | Does not fire triggers. |

---

## 2. Syntax Examples

```sql
-- 1. DELETE: Remove specific records
DELETE FROM customers WHERE status = 'INACTIVE';

-- 2. TRUNCATE: Empty entire table quickly while keeping structure
TRUNCATE TABLE staging_orders;

-- 3. DROP: Remove table structure and data permanently from catalog
DROP TABLE old_archive_orders;
```
