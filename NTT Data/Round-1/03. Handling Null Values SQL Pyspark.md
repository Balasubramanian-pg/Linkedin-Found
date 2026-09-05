# Handling NULL Values in SQL and PySpark

## Question
How do you handle `NULL` values in SQL and PySpark?

---

## 1. Handling NULL Values in SQL

In relational databases, `NULL` represents missing or unknown data. Standard equality operators (`= NULL`) do not work; special functions and conditions must be used.

### Common SQL Techniques:

#### A. Checking for NULLs
```sql
SELECT * FROM employees WHERE manager_id IS NULL;
SELECT * FROM employees WHERE commission_pct IS NOT NULL;
```

#### B. Replacing NULLs with Fallback Values
- **`COALESCE(val1, val2, ...)`** (ANSI SQL Standard): Returns the first non-null expression.
  ```sql
  SELECT emp_name, COALESCE(phone_number, mobile_number, 'No Contact') AS contact
  FROM employees;
  ```
- **`IFNULL(col, default_val)`** (MySQL) / **`NVL(col, default_val)`** (Oracle) / **`ISNULL(col, default_val)`** (SQL Server):
  ```sql
  SELECT emp_name, ISNULL(bonus, 0) AS bonus FROM employees;
  ```

#### C. Conditional Replacement (`CASE WHEN`)
```sql
SELECT 
    emp_name,
    CASE 
        WHEN department_id IS NULL THEN 'Unassigned'
        ELSE dept_name 
    END AS department
FROM employees;
```

#### D. NULLs in Aggregations & Joins
- Aggregation functions (`SUM()`, `AVG()`, `COUNT(col)`) ignore `NULL` values, whereas `COUNT(*)` counts all rows including `NULL`s.
- To treat `NULL` as matchable in joins:
  ```sql
  -- Standard SQL NULL-safe equal
  SELECT * FROM table_a a JOIN table_b b 
  ON a.code IS NOT DISTINCT FROM b.code;
  ```

---

## 2. Handling NULL Values in PySpark

PySpark provides DataFrame API functions within `pyspark.sql.functions` and `DataFrame.na` sub-modules.

### A. Detecting & Filtering NULLs
```python
from pyspark.sql import functions as F

# Filter rows where column is null / not null
df_nulls = df.filter(F.col("email").isNull())
df_valid = df.filter(F.col("email").isNotNull())
```

### B. Dropping NULL Values (`dropna` / `na.drop`)
```python
# Drop rows if ANY column is null
df_clean = df.na.drop()

# Drop rows if ALL columns are null
df_clean = df.na.drop(how="all")

# Drop rows if specific columns have nulls
df_clean = df.na.drop(subset=["customer_id", "transaction_date"])

# Drop rows that have less than 'thresh' non-null values
df_clean = df.na.drop(thresh=3)
```

### C. Imputing / Replacing NULL Values (`fillna` / `na.fill`)
```python
# Replace nulls across types or specific columns
df_imputed = df.na.fill({
    "age": 0,
    "salary": 50000.0,
    "city": "Unknown",
    "is_active": False
})

# Using coalesce function in DataFrame transformation
df_coalesced = df.withColumn(
    "effective_phone",
    F.coalesce(F.col("home_phone"), F.col("work_phone"), F.lit("N/A"))
)
```

### D. Advanced NULL Handling in PySpark

#### NULL-Safe Equality Operator (`<=>`)
```python
# Join or filter treating null == null as True
df_joined = df1.join(df2, df1["id"].eqNullSafe(df2["id"]))
# or using SQL expression
df_joined = df1.join(df2, F.expr("df1.id <=> df2.id"))
```

#### Imputing with Statistical Aggregates (Mean/Median)
```python
from pyspark.ml.feature import Imputer

imputer = Imputer(
    inputCols=["age", "income"], 
    outputCols=["age_imputed", "income_imputed"]
).setStrategy("mean") # or "median", "mode"

model = imputer.fit(df)
df_imputed = model.transform(df)
```

---

## Summary Matrix

| Operation | SQL | PySpark |
| :--- | :--- | :--- |
| **Check Null** | `IS NULL` / `IS NOT NULL` | `F.col("c").isNull()` / `.isNotNull()` |
| **Coalesce** | `COALESCE(col1, col2, default)` | `F.coalesce(F.col("c1"), F.col("c2"), F.lit(d))` |
| **Drop Nulls**| `WHERE col IS NOT NULL` | `df.na.drop(subset=["col"])` |
| **Fill Default**| `COALESCE(col, 'N/A')` / `NVL` | `df.na.fill({"col": "N/A"})` |
| **Null-Safe Join**| `IS NOT DISTINCT FROM` | `.eqNullSafe()` or `<=>` |
