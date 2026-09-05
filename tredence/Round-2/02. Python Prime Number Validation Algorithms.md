# Python: Prime Number Validation & Sieve of Eratosthenes

## Question
Write Python functions to:
1. Validate whether a given integer N is prime in optimal O(sqrt(N)) time.
2. Generate all prime numbers up to N using the **Sieve of Eratosthenes** (O(N log log N)).

---

## 1. Single Number Prime Check (O(sqrt(N)) Time, O(1) Space)

```python
def is_prime(n: int) -> bool:
    if n <= 1:
        return False
    if n <= 3:
        return True
    if n % 2 == 0 or n % 3 == 0:
        return False
        
    # Check divisors of form 6k ± 1 up to sqrt(n)
    i = 5
    while i * i <= n:
        if n % i == 0 or n % (i + 2) == 0:
            return False
        i += 6
        
    return True
```

---

## 2. Generating All Primes up to N (Sieve of Eratosthenes)

```python
from typing import List

def sieve_of_eratosthenes(limit: int) -> List[int]:
    # Generates all prime numbers <= limit in O(N log log N) time
    if limit < 2:
        return []
        
    primes = [True] * (limit + 1)
    primes[0] = primes[1] = False
    
    p = 2
    while p * p <= limit:
        if primes[p]:
            # Mark multiples of p starting from p^2
            for multiple in range(p * p, limit + 1, p):
                primes[multiple] = False
        p += 1
        
    return [num for num, is_p in enumerate(primes) if is_p]

# Test
if __name__ == "__main__":
    assert is_prime(29) is True
    assert is_prime(30) is False
    print("Primes up to 50:", sieve_of_eratosthenes(50))
```
