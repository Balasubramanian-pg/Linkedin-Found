# Designing Large-Scale Data Pipelines (TB-Scale Architecture)

## Question
How do you architect and design robust, cost-effective data pipelines for large-scale data (Terabytes/Petabytes per day)?

---

## 1. Architectural Blueprint for Terabyte-Scale Data Processing

```
+-----------------------------------------------------------------------------------------+
|                                    Terabyte-Scale Architecture                          |
+-----------------------------------------------------------------------------------------+
  [ Sources ]                [ Ingestion ]              [ Storage & Compute ]   [ Serving ]
  - Kafka / Event Hubs ----> Auto Loader (Databricks) -> ADLS Gen2 (Delta) ---> Databricks SQL
  - DBs / APIs ------------> ADF Incremental Copy ----> - Bronze (Append-Only)  - Power BI DirectLake
                                                        - Silver (MERGE Upsert) - Feature Store
                                                        - Gold (Star Schema)
+-----------------------------------------------------------------------------------------+
| Governance & Infrastructure: Unity Catalog | Azure Key Vault | Spot Instances | Terraform |
+-----------------------------------------------------------------------------------------+
```

---

## 2. Core Architectural Pillars

### A. Ingestion Layer: Micro-Batching & Streaming
- **Auto Loader (`cloudFiles`):** Efficiently processes new files incrementally as they land in ADLS Gen2 without expensive directory file-listing operations.
- **Backpressure & Rate Limiting:** Configure `maxFilesPerTrigger` or `maxBytesPerTrigger` to prevent executor overload during burst traffic:
  ```python
  spark.readStream.format("cloudFiles") \
      .option("cloudFiles.format", "parquet") \
      .option("cloudFiles.maxFilesPerTrigger", 1000) \
      .load("abfss://raw@datalake.dfs.core.windows.net/transactions/")
  ```

---

### B. Storage & Partitioning Strategy (Delta Lake)
1. **Avoid Over-Partitioning:**
   - Partitioning by minute/hour or high-cardinality columns creates millions of tiny files, destroying metadata performance.
   - **Best Practice:** Partition only when partitions are $\ge 1\text{ GB}$ (e.g., `date=YYYY-MM-DD` or `region`).
2. **Liquid Clustering / Z-ORDER:**
   - Replaces static multi-level partitioning with dynamic clustering on high-frequency query columns.
3. **Data Lifecycle & Vacuuming:**
   - Schedule periodic `VACUUM table RETAIN 168 HOURS;` (7-day safety retention) to prune stale Parquet files and reduce storage costs.

---

### C. Compute & Cluster Optimization
1. **Dynamic Auto-Scaling:**
   - Use Min-Max worker scaling with autoscaling enabled to accommodate fluctuating hourly volumes.
2. **Cost Optimization with Azure Spot Instances:**
   - Run stateless Spark worker nodes on Spot/Preemptible VMs (up to 70% cost discount), while keeping the Driver node on an On-Demand VM.
3. **Cluster Node Sizing:**
   - Use memory-optimized VM families (e.g., `Standard_E16ds_v5` with NVMe SSD local caching) for heavy shuffle operations.

---

### D. Idempotency & Fault Tolerance
1. **Strict Idempotency:**
   - Ensure rerun pipelines produce identical results. Never do bare `INSERT INTO`; use deterministic `MERGE INTO` or partition replacement (`replaceWhere`):
   ```python
   df.write.format("delta") \
       .mode("overwrite") \
       .option("replaceWhere", "transaction_date = '2026-08-30'") \
       .save(gold_path)
   ```
2. **Checkpoints & Transaction Logs:**
   - Store checkpoints on ADLS Gen2 so failed streaming runs resume from the exact committed micro-batch offset.

---

### E. Data Quality & Monitoring
1. **Dead Letter Queue (DLQ) Pattern:**
   - Quarantine schema mismatches or corrupt records into an `_invalid_records` table without halting the main pipeline.
2. **Alerting & Observability:**
   - Emit custom metrics to Azure Monitor / Datadog: records processed per second, byte volume, execution duration, and schema drifts.
