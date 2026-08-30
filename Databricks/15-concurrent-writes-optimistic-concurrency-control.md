# Concurrent Writes in Delta Lake: Optimistic Concurrency Control (OCC)

## Question
Two data engineering jobs simultaneously attempt to write to or update the same Delta table. What happens under the hood, and how are write conflicts resolved?

---

## 1. Optimistic Concurrency Control (OCC) Workflow

Delta Lake uses **Optimistic Concurrency Control (OCC)** to manage concurrent transactions:

```
[ Job A (Merge) ] ──> Reads Snapshot v10 ──> Computes changes locally
[ Job B (Append) ]──> Reads Snapshot v10 ──> Computes changes locally
                           │
                           ▼
  [ Job B completes first: Commits `000011.json` successfully ]
                           │
                           ▼
  [ Job A attempts to commit `000011.json` ] ──> CONFLICT DETECTED!
                           │
                           ▼
  [ Delta Engine Conflict Check ]
     • Did Job B modify any files that Job A read/modified?
       ├── NO (Mutually exclusive partitions) ──> Job A auto-retries & commits `000012.json` cleanly!
       └── YES (Overlapping files/rows)        ──> Job A throws `ConcurrentAppendException` / `ConcurrentModificationException`
```

---

## 2. Conflict Matrix

| Operation 1 | Operation 2 | Can Conflict? | Resolution |
| :--- | :--- | :--- | :--- |
| **Append** | **Append** | ❌ No conflict | Both append commits succeed consecutively. |
| **Append** | **Update / Merge** | ⚠️ Only if append touches matching partition | Auto-retried by default in Delta. |
| **Update / Merge**| **Update / Merge** | ✅ Yes (if touching same files) | One job commits; the other fails after retry limit. |
| **Delete / Overwrite**| **Any Write** | ✅ Yes | Conflict if touching same partitions. |
