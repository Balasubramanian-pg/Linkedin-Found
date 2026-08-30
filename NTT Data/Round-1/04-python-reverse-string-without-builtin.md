# Reversing a String in Python Without Built-in Functions

## Question
Write a Python program to reverse a string without using built-in functions (such as `reversed()`, `[::-1]`, or `.reverse()`).

---

## Approach 1: Two-Pointer Swap with List Conversion (Optimal: $O(N)$ Time, $O(N)$ Space)

Since strings in Python are immutable, we convert characters to a list, swap elements from both ends moving toward the center, and reconstruct the string.

```python
def reverse_string_two_pointer(s: str) -> str:
    chars = list(s)
    left = 0
    right = len(chars) - 1
    
    while left < right:
        # Swap characters
        chars[left], chars[right] = chars[right], chars[left]
        left += 1
        right -= 1
        
    return "".join(chars)

# Test
input_str = "DataEngineering"
print(reverse_string_two_pointer(input_str))
# Output: gnireenignEataD
```

---

## Approach 2: Iterative Prepending (Loop-Based)

Iterate through each character and prepend it to an accumulator string.

```python
def reverse_string_loop(s: str) -> str:
    reversed_str = ""
    for char in s:
        reversed_str = char + reversed_str
    return reversed_str

# Test
print(reverse_string_loop("AzureDatabricks"))
# Output: skcirbataDeruzA
```

> **Note on Performance:** While simple, string concatenation inside a loop creates new string objects each iteration in standard Python, leading to $O(N^2)$ runtime for very long strings. Approach 1 or using an array buffer is preferred in production.

---

## Approach 3: Decrementing Index Loop

Iterate backward from the last index (`len(s) - 1`) down to `0`.

```python
def reverse_string_indexed(s: str) -> str:
    reversed_chars = []
    idx = len(s) - 1
    while idx >= 0:
        reversed_chars.append(s[idx])
        idx -= 1
    return "".join(reversed_chars)

# Test
print(reverse_string_indexed("NTTData"))
# Output: ataDTTN
```

---

## Approach 4: Recursive Solution

Divide the string into the head (first char) and tail (remaining chars), reversing recursively.

```python
def reverse_string_recursive(s: str) -> str:
    if len(s) <= 1:
        return s
    return reverse_string_recursive(s[1:]) + s[0]

# Test
print(reverse_string_recursive("Python"))
# Output: nohtyP
```

---

## Complexity Analysis

| Method | Time Complexity | Auxiliary Space | Remarks |
| :--- | :--- | :--- | :--- |
| **Two-Pointer Swap** | $O(N)$ | $O(N)$ | Highly efficient, demonstrates classical pointer manipulation. |
| **Iterative Append/Prepend** | $O(N)$ to $O(N^2)$ | $O(N)$ | Clean and straightforward. |
| **Indexed Loop** | $O(N)$ | $O(N)$ | Avoids repeated string allocation using array buffer. |
| **Recursion** | $O(N)$ | $O(N)$ call stack | Elegant, but limited by recursion stack depth limit (1000). |
