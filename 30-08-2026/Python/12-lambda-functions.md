# Python Lambda Functions Explained

## Question
What are Lambda Functions in Python, how do they work, and when should you use them?

---

## 1. What is a Lambda Function?

A **Lambda Function** is a small, anonymous (unnamed) function defined using the `lambda` keyword. It can take any number of arguments, but can only evaluate a **single expression** and return its result.

### Syntax:
```python
lambda arguments: expression
```

---

## 2. Comparison with Regular Functions

```python
# Standard Function
def square(x):
    return x ** 2

# Equivalent Lambda Function
square_lambda = lambda x: x ** 2

print(square(5))        # 25
print(square_lambda(5)) # 25
```

---

## 3. Practical Use Cases in Data Analysis

### A. Sorting with Custom Keys
```python
employees = [
    {"name": "Alice", "salary": 95000},
    {"name": "Bob", "salary": 120000},
    {"name": "Charlie", "salary": 80000}
]

# Sort by salary descending
sorted_employees = sorted(employees, key=lambda emp: emp["salary"], reverse=True)
```

### B. With `map()` and `filter()`
```python
numbers = [1, 2, 3, 4, 5, 6]

# Filter even numbers
evens = list(filter(lambda x: x % 2 == 0, numbers)) # [2, 4, 6]

# Double each number
doubled = list(map(lambda x: x * 2, numbers))       # [2, 4, 6, 8, 10, 12]
```

### C. In Pandas Transformations (`apply`)
```python
import pandas as pd

df = pd.DataFrame({"revenue": [1000, 5000, 15000]})
df["tier"] = df["revenue"].apply(lambda x: "Enterprise" if x >= 10000 else "SMB")
```
