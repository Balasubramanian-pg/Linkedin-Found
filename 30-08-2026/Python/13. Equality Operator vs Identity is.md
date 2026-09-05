# Difference Between `==` and `is` in Python

## Question
What is the difference between the equality operator (`==`) and the identity operator (`is`) in Python?

---

## 1. Core Difference

- **`==` (Equality Operator):** Compares the **values** or contents of two objects (calls `__eq__()`).
- **`is` (Identity Operator):** Compares the **memory addresses** of two objects, checking if they refer to the exact same object in RAM (`id(a) == id(b)`).

---

## 2. Code Examples

```python
# Two distinct lists with identical values
list1 = [1, 2, 3]
list2 = [1, 2, 3]
list3 = list1

print(list1 == list2) # True  (Values are identical)
print(list1 is list2) # False (Different memory locations)
print(list1 is list3) # True  (Point to the same object in RAM)

print(id(list1)) # e.g., 14023450912
print(id(list2)) # e.g., 14023450984 (different)
```

---

## 3. Best Practice: Checking for `None`

> Always use `is` or `is not` when comparing against singleton objects like `None`, `True`, or `False`.

```python
# ✅ Correct
if value is None:
    pass

# ❌ Suboptimal
if value == None:
    pass
```
