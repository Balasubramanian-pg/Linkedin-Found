# Python Data Structures: List vs Tuple vs Set vs Dictionary

## Question
Explain the differences between `List`, `Tuple`, `Set`, and `Dictionary` in Python with syntax, properties, and use cases.

---

## 1. Comparison Matrix

| Property | List | Tuple | Set | Dictionary |
| :--- | :--- | :--- | :--- | :--- |
| **Syntax** | `[1, 2, 3]` | `(1, 2, 3)` | `{1, 2, 3}` | `{"a": 1, "b": 2}` |
| **Mutability** | **Mutable** | **Immutable** | **Mutable** | **Mutable** |
| **Ordering** | Ordered | Ordered | **Unordered** | Insertion Ordered (3.7+) |
| **Duplicates** | Allows duplicates | Allows duplicates | **Unique elements only** | Unique keys, duplicate values |
| **Indexing** | 0-based index `lst[0]` | 0-based index `tup[0]`| No indexing (hash-based) | Key-based `d["key"]` |
| **Lookup Speed**| $O(N)$ for values | $O(N)$ for values | **$O(1)$ Average** (Hash table) | **$O(1)$ Average** |

---

## 2. Practical Examples & Use Cases

```python
# 1. List: Ordered collection of items that change
sales_pipeline = ["Lead", "Contacted", "Qualified", "Closed"]
sales_pipeline.append("Retained")

# 2. Tuple: Immutable record / coordinates
db_credentials = ("localhost", 5432, "analytics_dw")

# 3. Set: Fast membership testing and deduplication
unique_user_ids = {101, 102, 103, 101} # {101, 102, 103}
if 102 in unique_user_ids: # O(1) instant lookup
    print("User exists")

# 4. Dictionary: Key-value lookups / JSON data
customer_profile = {
    "user_id": 101,
    "name": "Sarah Connor",
    "tier": "Platinum"
}
```
