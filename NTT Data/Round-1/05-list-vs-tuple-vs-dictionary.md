# Python Data Structures: List vs Tuple vs Dictionary

## Question
Explain `list` vs `tuple` vs `dictionary` with characteristics and real-world use cases.

---

## 1. High-Level Comparison Matrix

| Feature | List (`list`) | Tuple (`tuple`) | Dictionary (`dict`) |
| :--- | :--- | :--- | :--- |
| **Syntax** | `[1, 2, 3]` | `(1, 2, 3)` | `{"key": "value"}` |
| **Data Structure** | Dynamic Array | Static / Fixed-size Array | Hash Map (Hash Table) |
| **Mutability** | **Mutable** (modifiable) | **Immutable** (read-only) | **Mutable** (keys must be hashable) |
| **Ordering** | Ordered (maintains insertion order)| Ordered | Insertion Ordered (Python 3.7+) |
| **Access Method** | 0-based Integer Index (`list[0]`)| 0-based Integer Index (`tup[0]`)| Key-based Lookup (`dict["k"]`) |
| **Lookup Time** | $O(N)$ for search, $O(1)$ by index | $O(N)$ for search, $O(1)$ by index | $O(1)$ average for key lookup |
| **Memory Footprint**| Higher (pre-allocates space for growth)| Minimal (exact memory allocation)| Higher (hash table overhead) |
| **Hashability** | No (cannot be dict key / set item)| Yes (if all elements are hashable)| No |

---

## 2. Deep Dive and Code Examples

### A. List (`list`)
- **Nature:** An ordered, mutable collection of arbitrary objects.
- **When to use:** When you need a dynamic sequence of items where elements can be appended, removed, sorted, or updated in-place.
- **Example in Data Engineering:**
  ```python
  # Storing dynamic batch partition paths to process
  partition_paths = [
      "s3://datalake/raw/2026-08-01/",
      "s3://datalake/raw/2026-08-02/"
  ]
  partition_paths.append("s3://datalake/raw/2026-08-03/")
  partition_paths.sort()
  ```

---

### B. Tuple (`tuple`)
- **Nature:** An ordered, immutable collection of items.
- **When to use:** For constant/read-only records, composite keys, or returning multiple values from a function. Guarantees data integrity by preventing accidental modifications and uses less memory.
- **Example in Data Engineering:**
  ```python
  # Defining database connection configurations or coordinate/dimension tuples
  DB_CONFIG = ("db-prod.internal.net", 5432, "analytics_dw")

  # Function returning multiple values
  def get_table_metrics(df) -> tuple[int, int]:
      row_count = df.count()
      col_count = len(df.columns)
      return (row_count, col_count)

  # Tuple as dictionary key (composite primary key)
  record_cache = {
      ("US", "2026-08-30"): 154000.50,
      ("EU", "2026-08-30"): 210000.00
  }
  ```

---

### C. Dictionary (`dict`)
- **Nature:** A collection of key-value pairs backed by an optimized hash table.
- **When to use:** When ultra-fast $O(1)$ lookups, mappings, schema definitions, JSON representation, or entity associations are needed.
- **Example in Data Engineering:**
  ```python
  # Schema mapping and ETL column renaming dictionary
  column_mapping = {
      "cust_id": "customer_id",
      "tx_dt": "transaction_date",
      "amt": "transaction_amount"
  }

  # Fast record lookup / metadata registry
  pipeline_status = {
      "pipeline_id": "adf_sales_ingest_01",
      "status": "RUNNING",
      "retry_count": 0,
      "parameters": {"batch_id": 9821}
  }

  # Renaming PySpark columns using dictionary lookup
  for old_col, new_col in column_mapping.items():
      df = df.withColumnRenamed(old_col, new_col)
  ```

---

## 3. Summary of Best Practice Selection Rules

1. Choose a **Tuple** if data is fixed, represents a heterogeneous record (like a database row), or must serve as a hashable dictionary key.
2. Choose a **List** if elements are homogenous, will change dynamically in length, or need sorting/reordering.
3. Choose a **Dictionary** if you need fast key-to-value lookups, configurations, JSON handling, or deduplication maps.
