# Python Core Fundamentals: Concurrency, Memory Management & Mutability

## Question
Explain **Multiprocessing vs Multithreading (GIL)**, Python **Memory Management (Reference Counting & GC)**, and **Mutable vs Immutable objects** with practical data engineering implications.

---

## 1. Multithreading vs Multiprocessing (The Python GIL)

In standard CPython, the **Global Interpreter Lock (GIL)** is a mutex that prevents multiple native OS threads from executing Python bytecodes simultaneously in a single process.

```
Multithreading (1 Process, Shared Memory, Restricted by GIL):
[ OS Process (Memory Space) ]
   ├── Thread 1 ──[ Holds GIL ]──> Executing Python Bytecode
   ├── Thread 2 ──[ Waiting  ]
   └── Thread 3 ──[ Waiting  ]
   * Best for I/O-Bound tasks (Network calls, DB queries, File downloads)

Multiprocessing (Multiple Independent Processes, Separate Memory):
[ Process 1 (Core 1) ] ──> GIL 1 ──> Core 1 Compute (100% CPU)
[ Process 2 (Core 2) ] ──> GIL 2 ──> Core 2 Compute (100% CPU)
[ Process 3 (Core 3) ] ──> GIL 3 ──> Core 3 Compute (100% CPU)
   * Best for CPU-Bound tasks (Data transformations, Encryption, Regex parsing)
```

### Comparison Matrix:

| Feature | `threading` / `asyncio` | `multiprocessing` / `concurrent.futures` |
| :--- | :--- | :--- |
| **Execution Model** | Single process, multiple threads. | Multiple independent OS processes. |
| **GIL Impact** | Subject to GIL; only 1 thread computes at a time. | Bypasses GIL; true parallel multi-core execution. |
| **Memory Sharing** | Shared memory space (Fast, but risk of race conditions). | Isolated memory space (IPC / serialization overhead via `pickle`). |
| **Best Used For** | Web scraping, concurrent API calls, database pooling. | Data enrichment, heavy CPU JSON parsing, mathematical computation. |

---

## 2. Python Memory Management & Garbage Collection

Python uses a dual memory management architecture:

### A. Reference Counting (Primary Real-time Mechanism)
- Every Python object has an internal `ob_refcnt` tracking how many variables/pointers reference it.
- When `ref_count == 0`, the memory is deallocated immediately.
```python
import sys
a = [1, 2, 3] # ref count = 1
b = a         # ref count = 2
del a         # ref count = 1
del b         # ref count = 0 -> Memory freed instantly
```

### B. Cyclic Garbage Collector (Generational GC)
- Reference counting fails on **circular references** (e.g., Object A references Object B, and Object B references Object A).
- Python runs a generational garbage collector with 3 generations (Gen 0, Gen 1, Gen 2). Objects surviving young collections are promoted to older generations.
- In long-running PySpark jobs or drivers, tuning GC prevents driver freezing:
  ```python
  import gc
  gc.collect() # Manually trigger collection after processing a massive batch
  ```

---

## 3. Mutable vs Immutable Objects

| Category | Data Types | Behavior |
| :--- | :--- | :--- |
| **Immutable** | `int`, `float`, `str`, `tuple`, `frozenset`, `bytes` | Object value **cannot** be changed in-place after creation. Modifications create a new object in memory with a new `id()`. |
| **Mutable** | `list`, `dict`, `set`, `bytearray` | Object content can be modified in-place without altering its memory address (`id()`). |

### Critical Gotcha: Mutable Default Arguments
```python
# ❌ Anti-pattern: Default list is shared across all function calls
def append_event(event_id: str, batch: list = []) -> list:
    batch.append(event_id)
    return batch

print(append_event("TX1")) # ['TX1']
print(append_event("TX2")) # ['TX1', 'TX2']  <-- BUG!

# ✅ Best Practice: Use None as default
def append_event_safe(event_id: str, batch: list = None) -> list:
    if batch is None:
        batch = []
    batch.append(event_id)
    return batch
```
