# Star Schema vs Snowflake Schema in Dimensional Modeling

## Question
Explain the difference between **Star Schema** and **Snowflake Schema** with a real-time data engineering example.

---

## 1. High-Level Comparison

| Feature | Star Schema | Snowflake Schema |
| :--- | :--- | :--- |
| **Dimension Normalization** | **Denormalized** (Dimensions are flattened into single tables). | **Normalized** (Dimensions are split into sub-dimension tables / 3NF). |
| **Complexity of Queries** | Simple SQL (fewer joins, directly connecting Fact to Dimensions). | Complex SQL (multiple nested joins across normalized hierarchy). |
| **Query Performance** | **Faster** for OLAP / BI reporting (minimizes join overhead). | Slower (requires joining parent and child dimension tables). |
| **Data Redundancy & Storage**| Higher redundancy (repeated attributes in dimension rows). | Minimal redundancy (normalized lookup tables save storage). |
| **Maintenance / ETL** | Simpler updates; larger dimension rows. | Complex ETL pipelines to maintain foreign keys across sub-dimensions. |
| **Modern Cloud DW Suitability**| **Preferred standard** for Synapse, Snowflake, Databricks, BigQuery. | Used when dimensional hierarchies are exceptionally deep or wide. |

---

## 2. Real-Time Retail E-Commerce Example

Consider a Global Retail Store analytics model tracking daily item sales.

### A. Star Schema Design (Denormalized)

In a Star Schema, the central **Fact** table connects directly to flat **Dimension** tables.

```
       [ Dim_Customer ]               [ Dim_Date ]
              \                              /
               \                            /
                v                          v
             +-------------------------------+
             |         Fact_Daily_Sales      |
             |-------------------------------|
             | sales_id (PK)                 |
             | date_key (FK)                 |
             | customer_key (FK)             |
             | product_key (FK)              |
             | store_key (FK)                |
             | quantity                      |
             | revenue_amount                |
             +-------------------------------+
                ^                          ^
               /                            \
              /                              \
        [ Dim_Store ]                  [ Dim_Product ]
                                      (Contains Product,
                                      Category, Sub-category,
                                      and Brand all in one)
```

#### SQL Query in Star Schema:
```sql
SELECT 
    p.category_name,
    d.year,
    SUM(f.revenue_amount) AS total_revenue
FROM Fact_Daily_Sales f
JOIN Dim_Product p ON f.product_key = p.product_key
JOIN Dim_Date d ON f.date_key = d.date_key
GROUP BY p.category_name, d.year;
```
> **Performance:** Only 2 simple hash joins against the Fact table.

---

### B. Snowflake Schema Design (Normalized)

In a Snowflake Schema, the `Dim_Product` table is normalized into multiple lookup tables to eliminate repetitive string columns.

```
[ Dim_Brand ]
      |
      v
[ Dim_Product ] ---> [ Dim_SubCategory ] ---> [ Dim_Category ] ---> [ Dim_Department ]
      |
      v
[ Fact_Daily_Sales ]
```

#### Normalized Tables:
1. `Dim_Product` (`product_id`, `product_name`, `subcategory_id`, `brand_id`, `unit_price`)
2. `Dim_SubCategory` (`subcategory_id`, `subcategory_name`, `category_id`)
3. `Dim_Category` (`category_id`, `category_name`, `department_id`)
4. `Dim_Department` (`department_id`, `department_name`)

#### SQL Query in Snowflake Schema:
```sql
SELECT 
    c.category_name,
    d.year,
    SUM(f.revenue_amount) AS total_revenue
FROM Fact_Daily_Sales f
JOIN Dim_Product p ON f.product_key = p.product_key
JOIN Dim_SubCategory sc ON p.subcategory_id = sc.subcategory_id
JOIN Dim_Category c ON sc.category_id = c.category_id
JOIN Dim_Date d ON f.date_key = d.date_key
GROUP BY c.category_name, d.year;
```
> **Performance:** Requires 4 joins across hierarchical dimension tables before computing aggregates.

---

## 3. Which One Should You Choose?

### Choose **Star Schema** (Recommended for Modern Cloud DWs):
- When querying large volumes of analytical data in **Power BI / Synapse / Databricks / Snowflake**.
- Modern columnar storage (Parquet / Delta Lake) uses dictionary encoding and run-length encoding (RLE), making string redundancy compression nearly identical to normalized storage without join penalties.

### Choose **Snowflake Schema**:
- When dimensions have distinct, many-to-many relationship levels with frequent updates in low-storage environments.
- When dimension tables are themselves massive (hundreds of millions of rows) and storage savings outweigh join compute costs.
