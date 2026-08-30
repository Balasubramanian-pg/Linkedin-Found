# PySpark: Joining Employee & Department Data with Complex Filters

## Question
Write a PySpark script to join an `employees` DataFrame with a `departments` DataFrame, apply department-level and salary-level filters, resolve ambiguous column names, and optimize the join execution.

---

## 1. PySpark Join Implementation

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.functions import broadcast


def join_employees_departments():
    spark = SparkSession.builder.appName("Tredence_Employee_Join").getOrCreate()

    # Sample Employee DataFrame
    emp_data = [
        (101, "Alice", "IT", 120000, "2021-03-01"),
        (102, "Bob", "HR", 85000, "2020-05-15"),
        (103, "Charlie", "IT", 95000, "2023-01-10"),
        (104, "David", "FINANCE", 140000, "2019-11-20"),
        (105, "Emma", "SALES", 70000, "2022-08-05")
    ]
    df_emp = spark.createDataFrame(emp_data, ["emp_id", "emp_name", "dept_code", "salary", "hire_date"])

    # Sample Department DataFrame
    dept_data = [
        ("IT", "Information Technology", "Building A", "Active"),
        ("HR", "Human Resources", "Building B", "Active"),
        ("FINANCE", "Financial Planning", "Building A", "Active"),
        ("MARKETING", "Global Marketing", "Building C", "Active")
    ]
    df_dept = spark.createDataFrame(dept_data, ["dept_code", "dept_name", "location", "dept_status"])

    # 1. Join with Broadcast Optimization (df_dept is small < 10MB)
    df_joined = (
        df_emp
        .join(
            broadcast(df_dept), 
            on="dept_code", 
            how="inner"
        )
        # 2. Apply complex business filters
        .filter(
            (F.col("salary") >= 90000) & 
            (F.col("location") == "Building A") &
            (F.col("dept_status") == "Active")
        )
        .select(
            "emp_id",
            "emp_name",
            "dept_name",
            "salary",
            "location"
        )
        .orderBy(F.col("salary").desc())
    )

    df_joined.show()


if __name__ == "__main__":
    join_employees_departments()
```
