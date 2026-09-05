# Difference Between Import Mode vs DirectQuery in Power BI

## Question
Compare **Import Mode**, **DirectQuery**, and **Composite/Dual Mode** in Power BI.

---

## 1. Comparison Matrix

| Feature | Import Mode | DirectQuery Mode |
| :--- | :--- | :--- |
| **Data Storage** | Loaded and compressed into Power BI RAM (VertiPaq). | Left in underlying source database; zero data stored in PBI. |
| **Query Performance** | **Ultra-fast (sub-second)** in-memory engine. | Slower (depends on source DB latency & network). |
| **DAX Functionality** | Full DAX capabilities supported without limits. | Certain DAX functions restricted / translated to SQL. |
| **Dataset Size Limit** | 1 GB (Pro) / Up to 400 GB (Premium). | Unlimited (queries directly against petabyte warehouses). |
| **Data Freshness** | Scheduled refresh (e.g., 8x/day or 48x/day). | **Real-time (live queries on visual interaction)**. |
