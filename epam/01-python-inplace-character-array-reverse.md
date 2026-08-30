# Python: In-Place Character Array Reversal ($O(N)$ Time, $O(1)$ Space)

## Problem Statement
Given an array of characters `s`, reverse the elements **in-place** without allocating an additional array.
- **Time Complexity:** $O(N)$
- **Space Complexity:** $O(1)$ auxiliary space

---

## 1. Algorithmic Approach: Two-Pointer Swap

We maintain two pointers:
- `left` initialized to index `0` (start of the array).
- `right` initialized to index `len(s) - 1` (end of the array).

Iterate while `left < right`, swapping the elements at `left` and `right`, incrementing `left`, and decrementing `right`.

```
Initial:   [ 'E', 'P', 'A', 'M', '!' ]
             ^                   ^
            left               right

Step 1:    [ '!', 'P', 'A', 'M', 'E' ]  (Swapped E and !)
                   ^         ^
                  left     right

Step 2:    [ '!', 'M', 'A', 'P', 'E' ]  (Swapped P and M)
                         ^
                    left==right (Loop Terminates)
```

---

## 2. Python Implementation

```python
from typing import List


def reverse_string_inplace(s: List[str]) -> None:
    left = 0
    right = len(s) - 1
    
    while left < right:
        # Tuple unpacking swap in O(1) space
        s[left], s[right] = s[right], s[left]
        left += 1
        right -= 1


# --- Unit Tests & Verification ---
if __name__ == "__main__":
    test_1 = ["h", "e", "l", "l", "o"]
    reverse_string_inplace(test_1)
    print("Test 1 Result:", test_1)
    assert test_1 == ["o", "l", "l", "e", "h"]

    test_2 = ["E", "P", "A", "M"]
    reverse_string_inplace(test_2)
    print("Test 2 Result:", test_2)
    assert test_2 == ["M", "A", "P", "E"]

    test_3 = ["D"]
    reverse_string_inplace(test_3)
    print("Test 3 Result:", test_3)
    assert test_3 == ["D"]

    print("All test cases passed successfully!")
```

---

## 3. Complexity Analysis
- **Time Complexity:** $O(N)$ — Exactly $\lfloor N/2 \rfloor$ swap operations executed.
- **Auxiliary Space Complexity:** $O(1)$ — Only two integer pointers (`left`, `right`) allocated in memory.
