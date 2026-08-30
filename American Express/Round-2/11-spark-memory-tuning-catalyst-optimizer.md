# Spark Memory Management & Catalyst Optimizer Internals

## Question
How is Apache Spark Executor memory structured (On-Heap vs Off-Heap), and what are the internal execution stages of the **Catalyst Optimizer**?

---

## 1. Spark Executor Memory Architecture

Every Spark Executor operates inside a JVM process whose memory is partitioned into distinct regions:

```
+-------------------------------------------------------------------------------+
|                       Total Executor Memory (JVM Container)                   |
+-------------------------------------------------------------------------------+
| OverHead Memory (`spark.executor.memoryOverhead` - Default max(384MB, 10%))  |
| (Py4J IPC, OS Native buffers, C++/Python process overhead)                     |
+-------------------------------------------------------------------------------+
| On-Heap JVM Memory (`spark.executor.memory`)                                  |
| +---------------------------------------------------------------------------+ |
| | Reserved Memory (300MB fixed for Spark internal system objects)           | |
| +---------------------------------------------------------------------------+ |
| | User Memory (Default 40% of remaining: UDFs, custom data structures)      | |
| +---------------------------------------------------------------------------+ |
| | Spark Unified Memory Pool (Default 60% - `spark.memory.fraction` = 0.6)   | |
| |   ├── Execution Memory (Joins, Aggregations, Shuffles, Sorts)             | |
| |   └── Storage Memory (Cached DataFrames, Broadcast tables)                | |
| |   * Dynamic Borrowing: Execution can evict Storage if needed              | |
| +---------------------------------------------------------------------------+ |
+-------------------------------------------------------------------------------+
| Off-Heap Memory (Project Tungsten - `spark.memory.offHeap.enabled = true`)   |
| (Direct C-style memory allocation, zero JVM GC pauses, compact binary format) |
+-------------------------------------------------------------------------------+
```

---

## 2. On-Heap vs Off-Heap Memory (Project Tungsten)

| Attribute | On-Heap Memory | Off-Heap Memory (Tungsten) |
| :--- | :--- | :--- |
| **Location** | Inside standard Java Virtual Machine (JVM). | Outside JVM, managed via `sun.misc.Unsafe`. |
| **Garbage Collection (GC)**| High GC overhead and pause times for large objects. | **Zero JVM GC overhead**; avoids pauses entirely. |
| **Data Representation** | Java Object overhead (16-byte object headers, boxing). | Raw compact binary format (compact memory layout). |
| **Configuration** | `spark.executor.memory = 16g` | `spark.memory.offHeap.enabled = true`<br>`spark.memory.offHeap.size = 4g` |

---

## 3. The 4 Stages of the Catalyst Optimizer

The Catalyst Optimizer automatically translates high-level SQL/DataFrame code into an optimized physical execution graph:

```
[ SQL / DataFrame Query ]
           │
           ▼
[ 1. Unresolved Logical Plan ] ── (Validates table names, column existence against Catalog)
           │
           ▼
[ 2. Resolved (Analyzed) Logical Plan ] ── (Applies logical rules: Predicate Pushdown, Projection Pruning)
           │
           ▼
[ 3. Optimized Logical Plan ] ── (Cost-Based Optimizer (CBO) evaluates physical strategies)
           │
           ▼
[ 4. Physical Plans (Multiple) ] ── (Selects Best Plan: BHJ vs SMJ, Partitioning)
           │
           ▼
[ Whole-Stage Code Generation (Tungsten Bytecode) ] ──> Native RDD Execution
```

### Stage Breakdown:
1. **Analysis:** Resolves column types, table aliases, and relations using the Spark Catalog / Hive Metastore.
2. **Logical Optimization:** Applies standard relational algebra rules:
   - **Predicate Pushdown:** Moves `.filter()` clauses as close to the storage scan as possible.
   - **Projection Pruning:** Reads only requested columns from Parquet/Delta storage.
   - **Constant Folding:** Computes literal math (`1 + 1` &rarr; `2`) at compile time.
3. **Physical Planning:** Generates one or more candidate physical plans and assigns concrete execution algorithms (e.g., choosing BroadcastHashJoin vs SortMergeJoin).
4. **Code Generation (Whole-Stage Java Bytecode Generation):** Synthesizes optimized JVM bytecode on the fly that collates entire query stages into a single clean loop, avoiding virtual function call overhead.
