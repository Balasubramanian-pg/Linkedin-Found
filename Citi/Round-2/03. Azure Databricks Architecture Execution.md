# Azure Databricks Architecture & Spark Job Execution Lifecycle

## Question
Explain the architecture of Azure Databricks (Control Plane vs Data Plane) and detail how a Spark job is compiled and executed across Driver and Worker nodes.

---

## 1. Azure Databricks Architecture: Control Plane vs Data Plane

```
[ Control Plane (Managed by Microsoft/Databricks) ]
  ├── Web UI Workspace & Notebooks
  ├── Job Scheduler & Cluster Manager
  └── Unity Catalog Metadata Service
         │
         │ (Secure Cluster Connectivity / VNet Peering via TLS)
         ▼
[ Customer Data Plane (Inside Customer Azure Subscription VNet) ]
  ├── Azure Virtual Machines (Driver VM & Worker Node VMs)
  ├── Local Attached NVMe SSD Storage (Cache / Shuffle Spills)
  └── ADLS Gen2 Storage Account (`abfss://`)
```

---

## 2. Spark Job Execution Lifecycle

```
[ User Submits Query / Action (e.g. `df.count()`) ]
                     │
                     ▼
[ 1. Catalyst Optimizer (Driver Node) ]
     • Unresolved Logical Plan ──> Resolved Plan ──> Optimized Plan ──> Physical Plan
                     │
                     ▼
[ 2. DAG Scheduler (Driver Node) ]
     • Breaks physical plan into Stages based on Shuffle boundaries (Wide Transformations)
                     │
                     ▼
[ 3. Task Scheduler (Driver Node) ]
     • Splits stages into individual Tasks (1 Task per data partition)
     • Assigns tasks to Executors across Worker Nodes
                     │
                     ▼
[ 4. Executors on Worker Nodes ]
     • Executes tasks in parallel threads
     • Performs Whole-Stage Java CodeGen & Tungsten vectorization
     • Returns partition results or writes to ADLS Gen2
```
