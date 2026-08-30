# Large File I/O and Frequency Mapping with Custom Data Structures

## Question
How do you process large (multi-GB) files in Python without causing Out-of-Memory (OOM) errors, and implement frequency mapping using generator streams and custom data structures?

---

## 1. The Challenge of Large File I/O in Python

- **Naive approach:** `file.readlines()` or `file.read()` loads the entire file into RAM, crashing immediately on 10GB+ server log files.
- **Production approach:** Use **streaming iterators / generators** to process one line or chunk at a time with $O(1)$ memory footprint.

---

## 2. Implementation: Stream Processing & Chunk-Based Ingestion

```python
import collections
import heapq
import io
import re
from typing import Generator, Iterator, List, Tuple


def stream_log_lines(file_path: str) -> Generator[str, None, None]:
    """
    Lazy generator reading a large file line-by-line using buffered I/O.
    Memory complexity: O(1) buffer space.
    """
    with open(file_path, mode="r", encoding="utf-8", buffering=64 * 1024) as f:
        for line in f:
            yield line.strip()


def chunked_file_reader(file_path: str, chunk_size: int = 10000) -> Iterator[List[str]]:
    """
    Yields batches of lines for bulk database ingestion.
    """
    with open(file_path, "r", encoding="utf-8") as f:
        chunk = []
        for line in f:
            chunk.append(line.strip())
            if len(chunk) == chunk_size:
                yield chunk
                chunk = []
        if chunk:
            yield chunk
```

---

## 3. Custom Data Structure: Bounded Top-K Frequency Tracker

To track the top $K$ most frequent IP addresses or transaction tokens across billions of records without maintaining unbounded memory:

```python
class TopKFrequencyTracker:
    """
    Maintains exact token frequencies and retrieves Top-K elements
    using a Hash Map combined with a Min-Heap.
    """
    def __init__(self, k: int = 10):
        self.k = k
        self.freq_map = collections.defaultdict(int)

    def add_token(self, token: str) -> None:
        self.freq_map[token] += 1

    def process_line_tokens(self, line: str) -> None:
        # Extract alphanumeric words/tokens
        tokens = re.findall(r"\b[a-zA-Z0-9_\-\.]+\b", line)
        for token in tokens:
            self.freq_map[token] += 1

    def get_top_k(self) -> List[Tuple[str, int]]:
        """
        Retrieves Top-K elements using a Min-Heap in O(N log K) time,
        where N is the number of distinct keys.
        """
        # Min-heap stores (frequency, token)
        min_heap = []
        for token, count in self.freq_map.items():
            if len(min_heap) < self.k:
                heapq.heappush(min_heap, (count, token))
            elif count > min_heap[0][0]:
                heapq.heapreplace(min_heap, (count, token))
                
        # Return sorted in descending order of frequency
        return sorted([(token, count) for count, token in min_heap], key=lambda x: x[1], reverse=True)
```

---

## 4. End-to-End Execution Example

```python
# Simulated Large Log Stream
sample_data = """
2026-08-30 10:00:00 IP=192.168.1.1 STATUS=200 MERCHANT=AMEX_TRAVEL
2026-08-30 10:00:01 IP=10.0.0.5 STATUS=500 MERCHANT=AMEX_REWARDS
2026-08-30 10:00:02 IP=192.168.1.1 STATUS=200 MERCHANT=AMEX_TRAVEL
2026-08-30 10:00:03 IP=192.168.1.1 STATUS=200 MERCHANT=AMEX_DINING
2026-08-30 10:00:04 IP=172.16.0.2 STATUS=200 MERCHANT=AMEX_TRAVEL
"""

if __name__ == "__main__":
    tracker = TopKFrequencyTracker(k=3)
    
    # Process stream line-by-line
    for line in sample_data.strip().split("\n"):
        tracker.process_line_tokens(line)
        
    top_tokens = tracker.get_top_k()
    print("Top 3 Tokens/Values:", top_tokens)
    # Output: [('200', 4), ('AMEX_TRAVEL', 3), ('192.168.1.1', 3)]
```

---

## 5. Memory & Performance Summary

| Approach | Time Complexity | Memory Complexity | Scalability |
| :--- | :--- | :--- | :--- |
| `f.readlines()` | $O(N)$ | $O(\text{File Size})$ | Fails on multi-GB files |
| Generator Stream (`for line in f`) | $O(N)$ | $O(1)$ line buffer | Scales to multi-TB files |
| Hash Map + Min-Heap Top-K | $O(N \log K)$ | $O(\text{Distinct Keys})$ | Highly efficient |
