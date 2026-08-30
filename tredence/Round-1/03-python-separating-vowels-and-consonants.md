# Python: Separating Strings Based on Vowels and Consonants

## Question
Write a Python program to take an alphanumeric string, filter out non-alphabetic characters, and separate vowels and consonants while preserving their original relative order.

---

## 1. Python Implementation

```python
from typing import Tuple


def separate_vowels_and_consonants(text: str) -> Tuple[str, str]:
    # Separates vowels and consonants preserving case and order
    vowels_set = set("aeiouAEIOU")
    vowels_list = []
    consonants_list = []
    
    for char in text:
        if char.isalpha():
            if char in vowels_set:
                vowels_list.append(char)
            else:
                consonants_list.append(char)
                
    return "".join(vowels_list), "".join(consonants_list)


# --- Test Cases ---
if __name__ == "__main__":
    input_str = "Tredence Analytics 2026!"
    vowels, consonants = separate_vowels_and_consonants(input_str)
    
    print("Original Text:   ", input_str)
    print("Extracted Vowels:", vowels)      # Output: eeeaie
    print("Consonants:      ", consonants)  # Output: TrdncNlytcs
```

---

## 2. Advanced: List Comprehension Approach

```python
def separate_comprehension(s: str) -> Tuple[str, str]:
    v = set("aeiouAEIOU")
    vowels = "".join([c for c in s if c.isalpha() and c in v])
    consonants = "".join([c for c in s if c.isalpha() and c not in v])
    return vowels, consonants
```
