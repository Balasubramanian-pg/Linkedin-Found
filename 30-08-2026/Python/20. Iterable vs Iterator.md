# Difference Between Iterable and Iterator in Python

## Question
What is the difference between an **Iterable** and an **Iterator** in Python (the Iteration Protocol)?

---

## 1. Core Differences

| Feature | Iterable | Iterator |
| :--- | :--- | :--- |
| **Definition** | Any object that can be looped over (produces an iterator). | An object representing a stream of data; produces next value. |
| **Protocol Methods** | Implements `__iter__()`. | Implements both `__iter__()` and `__next__()`. |
| **State** | Stateless (can be looped multiple times). | **Stateful** (consumed one element at a time; exhausted after 1 pass). |
| **Examples** | `list`, `tuple`, `str`, `dict`, `set`, `range`. | `iter(list)`, generators, file objects, `map`, `zip`. |

---

## 2. Code Demonstration

```python
# 1. An Iterable (List)
numbers = [10, 20, 30]

# 2. Convert Iterable to Iterator
num_iterator = iter(numbers)

# 3. Retrieve items using next()
print(next(num_iterator)) # 10
print(next(num_iterator)) # 20
print(next(num_iterator)) # 30

# 4. Exhaustion raises StopIteration
try:
    print(next(num_iterator))
except StopIteration:
    print("Iterator is exhausted!")
```
