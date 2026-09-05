# Difference Between Pandas `loc[]` and `iloc[]`

## Question
What is the difference between `df.loc[]` and `df.iloc[]` in Pandas?

---

## 1. Comparison Matrix

| Feature | `df.loc[]` | `df.iloc[]` |
| :--- | :--- | :--- |
| **Access Method** | **Label-based** (Index names / column names). | **Integer position-based** (0-indexed integer indices). |
| **Slicing End Bound**| **Inclusive** of end label (`"A":"C"` includes C). | **Exclusive** of end integer (`0:3` includes 0, 1, 2). |
| **Boolean Masking** | ✅ Directly accepts boolean Series / conditions. | ❌ Requires boolean numpy arrays or integer arrays. |

---

## 2. Code Examples

```python
import pandas as pd

df = pd.DataFrame(
    {"revenue": [100, 200, 300], "cost": [60, 120, 180]},
    index=["Store_A", "Store_B", "Store_C"]
)

# --- 1. loc (Label-based) ---
print(df.loc["Store_A", "revenue"])           # 100
print(df.loc["Store_A":"Store_B", "revenue"]) # Includes Store_B!
print(df.loc[df["revenue"] > 150])            # Filtering by condition

# --- 2. iloc (Integer Position-based) ---
print(df.iloc[0, 0])     # 100 (Row 0, Col 0)
print(df.iloc[0:2, 0:1]) # Row 0 and 1, Col 0 (Exclusive of index 2)
```
