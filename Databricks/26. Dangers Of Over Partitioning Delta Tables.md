# The Dangers of Over-Partitioning Delta Tables

## Question
Why is over-partitioning a Delta table (e.g., partitioning by minute, hour, or high-cardinality IDs) dangerous, and how does it degrade lakehouse performance?

---

## 1. Negative Impacts of Over-Partitioning

```
[ Over-Partitioning Anti-Pattern: `/year=2026/month=08/day=30/hour=14/minute=25/` ]
                                       │
         ┌─────────────────────────────┼─────────────────────────────┐
         ▼                             ▼                             ▼
  [ Small File Explosion ]     [ Metadata Sprawl ]          [ Driver Bottleneck ]
  • Millions of 50KB files     • Massive Delta JSON logs    • Driver spends minutes
  • Horrible compression       • Slow file listing in ADLS    listing directory trees
```

1. **Small File Problem:** Creates millions of tiny KB-sized files, causing massive I/O overhead.
2. **Metadata Explosion:** Delta transaction log (`_delta_log`) balloons in size, causing query planning to take minutes before execution even begins.
3. **Photon / Columnar Inefficiency:** Columnar compression algorithms require large row groups (~128MB–1GB) to achieve 5x–10x compression ratios.

---

## 2. Golden Rule of Lakehouse Partitioning
- **Rule of Thumb:** A partition should contain **at least 1 GB** of data.
- For high-cardinality columns or multi-column queries, use **Liquid Clustering (`CLUSTER BY`)** instead of physical directory partitioning.
