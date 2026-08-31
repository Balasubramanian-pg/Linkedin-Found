# Scenario 07: Incremental File Ingestion from ADLS Gen2 using Databricks Auto Loader

## Problem Statement
Source applications continuously drop CSV, JSON, and Parquet files into Azure Data Lake Storage (ADLS Gen2). Reading the entire storage directory daily is slow and expensive. How do you design an incremental ingestion solution that processes only newly arrived files with schema evolution?

---

## 1. The Superior Solution: Databricks Auto Loader (`cloudFiles`)

Traditional file listing (`spark.read.load()`) performs expensive directory enumeration across millions of blob objects on ADLS Gen2 ($O(N)$ HTTP listing calls).

**Databricks Auto Loader** provides two high-performance modes:
1. **Directory Listing Mode:** Incrementally tracks newly discovered files via internal RocksDB state with lexical ordering.
2. **File Notification Mode:** Subscribes directly to **Azure Event Grid & Azure Storage Queue (Queue Storage)** to ingest files as soon as they land without listing directories.

```
[ External Producers ] ──> Drop files into ADLS Gen2 (`/raw/landing/`)
                                      │
                                      ▼ (Triggers Azure Event Grid Event)
                             [ Azure Storage Queue ]
                                      │
                                      ▼ (Subscribed by Databricks Auto Loader)
                      [ Spark Structured Streaming Job (`cloudFiles`) ]
                                      │
                                      ▼ (Appends atomically with Schema Evolution)
                             [ Bronze Delta Lake Table ]
```

---

## 2. Production PySpark Auto Loader Implementation

```python
from pyspark.sql import SparkSession

checkpoint_path = "abfss://checkpoints@datalake.dfs.core.windows.net/autoloader/bronze_payments"
raw_data_path = "abfss://raw@datalake.dfs.core.windows.net/incoming_payments/"
bronze_table_path = "abfss://bronze@datalake.dfs.core.windows.net/tables/bronze_payments"

# Auto Loader Stream with Schema Inference & Evolution
streaming_df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    # Store inferred schema and rescue corrupted records
    .option("cloudFiles.schemaLocation", f"{checkpoint_path}/schema")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("cloudFiles.schemaHints", "transaction_id STRING, amount DECIMAL(18,2)")
    # Capture unparsed/malformed JSON into a rescue column
    .option("cloudFiles.rescuedDataColumn", "_rescued_data")
    # File Notification Mode configuration (for sub-second event ingestion)
    .option("cloudFiles.useNotifications", "true")
    .option("cloudFiles.maxFilesPerTrigger", 5000)
    .load(raw_data_path)
)

# Append incrementally to Bronze Delta Table
query = (
    streaming_df.writeStream
    .format("delta")
    .outputMode("append")
    .option("checkpointLocation", f"{checkpoint_path}/write")
    .trigger(availableNow=True) # Runs incrementally like a batch job, then shuts down cluster!
    .toTable("bronze_payments")
)
```

---

## 3. Why `trigger(availableNow=True)` is a Game-Changer for Batch Workloads

- **Legacy `trigger(once=True)`:** Only processed one microbatch of files.
- **`trigger(availableNow=True)`:** Ingests **all** backlogged files across multiple rate-limited microbatches, maintains exact streaming checkpoint state, and cleanly shuts down the cluster when complete.
- **Result:** Gives the cost benefits of scheduled batch jobs with the exactly-once fault tolerance and incremental bookkeeping of streaming engines.
