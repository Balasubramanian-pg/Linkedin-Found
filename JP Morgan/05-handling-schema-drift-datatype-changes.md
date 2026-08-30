# Handling Schema Drift and Unexpected Datatype Mutations

## Question
A critical source system unexpectedly changes a column's datatype (e.g., `trade_volume` changed from `INTEGER` to `VARCHAR` containing formatted numbers like `"1,500"`), breaking production pipelines. How do you design pipelines to prevent, isolate, and handle schema drift gracefully?

---

## 1. Architectural Strategy: Defense-in-Depth

```
[ Inbound Data ]
       │
       ▼
[ Schema Registry & Contract Enforcement (Avro / Protobuf) ]
       │
       ├── Breaking Drift (Incompatible Mutation) ──> [ Quarantine / Dead Letter Queue ]
       │                                                    │
       ▼ (Rescued Data Column)                              ▼ (Slack/Teams Pager Alert)
[ Bronze Raw Layer (String / JSON Unaltered) ]       [ Data Governance / Provider Ticket ]
       │
       ▼ (Defensive Parsing / Casting with Fallbacks)
[ Silver Normalized Clean Layer ]
```

---

## 2. Prevention Pillar 1: Contract Enforcement with Schema Registry

For Kafka/Event streams, use **Confluent Schema Registry** with `BACKWARD_TRANSITIVE` or `FULL` compatibility:
- The producer cannot publish payloads that violate the registered Avro/Protobuf schema contract.
- Unregistered schema mutations are rejected at the message broker boundary.

---

## 3. Ingestion Pillar 2: Rescued Data Column (Databricks Auto Loader)

If ingesting files from cloud storage, configure `cloudFiles.rescuedDataColumn`:

```python
streaming_df = (
    spark.readStream
    .format("cloudFiles")
    .option("cloudFiles.format", "json")
    .option("cloudFiles.schemaLocation", "/mnt/checkpoints/trade_schema")
    .option("cloudFiles.schemaEvolutionMode", "rescue") # Or 'addNewColumns'
    .option("cloudFiles.rescuedDataColumn", "_rescued_data")
    .load("abfss://raw@datalake.dfs.core.windows.net/trades/")
)
```
- **How it works:** If a column `amount` was expected as `FLOAT` but receives `"INVALID"`, the record is **not dropped**; the corrupt attribute is saved into `_rescued_data` while the pipeline continues uninterrupted.

---

## 4. Transformation Pillar 3: Defensive Casting & Cleansing Functions

In the Silver transformation layer, use safe type-casting logic:

```python
from pyspark.sql import functions as F

def sanitize_and_cast_numeric(df, col_name: str, target_type="double"):
    # Strip currency symbols, commas, and whitespace before casting
    sanitized_col = F.regexp_replace(F.col(col_name), r"[^\d\.-]", "")
    return df.withColumn(
        f"{col_name}_clean",
        sanitized_col.cast(target_type)
    )
```
