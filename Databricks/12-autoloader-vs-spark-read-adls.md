# Auto Loader (`cloudFiles`) vs `spark.read` on ADLS Gen2

## Question
Why should you choose Databricks Auto Loader (`cloudFiles`) instead of standard `spark.read` / `spark.readStream` when ingesting files from ADLS Gen2?

---

## 1. Feature Comparison Matrix

| Feature | Standard `spark.read` / `spark.readStream` | Databricks Auto Loader (`cloudFiles`) |
| :--- | :--- | :--- |
| **File Discovery Mechanism** | Full directory listing scan (O(N) HTTP listing calls). | **File Notification (Event Grid/Queue)** or incremental RocksDB lexical scan. |
| **Scalability** | Becomes painfully slow as folder reaches 100K+ files. | Scales seamlessly to **billions of files** with sub-second discovery. |
| **Schema Inference Cost** | Scans entire dataset during startup (causes OOMs). | **Asynchronous / Sample-based schema inference**. |
| **Schema Evolution** | Pipeline fails on new columns or type drift. | Supports **automatic schema evolution** (`addNewColumns`, `rescue`). |
| **Malformed JSON / CSV** | Fails job or requires error-prone regex workarounds. | Built-in **Rescued Data Column (`_rescued_data`)**. |
| **Batch Support** | Must manually manage watermark dates/state. | Native batch support via **`trigger(availableNow=True)`**. |

---

## 2. Production Code Example

```python
streaming_df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/lake/checkpoints/payments_schema")
    .option("cloudFiles.schemaEvolutionMode", "addNewColumns")
    .option("cloudFiles.rescuedDataColumn", "_rescued_data")
    .option("cloudFiles.useNotifications", "true") # Azure Event Grid notifications
    .load("abfss://raw@datalake.dfs.core.windows.net/incoming_payments/")
)
```
