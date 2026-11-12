Incremental loading in Databricks is not a single feature but a set of complementary patterns, each suited to a different source behavior and latency requirement. The four mechanisms you named—watermark filtering, Delta `MERGE`, Change Data Capture (CDC via CDF or AUTO CDC), and Auto Loader—operate at different layers: Auto Loader handles **file-level** incremental ingestion, watermarking handles **time-based** row filtering, `MERGE` handles **key-based** upserts, and CDC handles **row-level change streams**. A production pipeline typically combines two or three of them across the Bronze, Silver, and Gold layers of a medallion architecture.

---

## The four incremental patterns at a glance

| Pattern | What it detects | Best for | Key limitation |
|---|---|---|---|
| **Watermark filtering** | Rows where a timestamp/ID exceeds a stored value | Append-only sources with a reliable monotonic column | Cannot detect updates or deletes; misses late-arriving data with old timestamps |
| **Delta `MERGE`** | Key matches between source and target, with operation flags for insert/update/delete | Sources that send a full CDC payload or periodic upserts | Performance degrades with high-frequency small upserts; creates small files without `OPTIMIZE` |
| **Change Data Feed (CDF)** | Row-level changes between Delta table versions | Downstream Silver/Gold consumers of Delta tables | Only works on Delta source tables, not external systems |
| **AUTO CDC** (Lakeflow) | Declarative CDC with keys and sequence columns | SCD Type 1/2 with out-of-order handling | Tied to Lakeflow Declarative Pipelines / streaming tables |
| **Auto Loader** | Newly arrived **files** in cloud storage | Raw ingestion from cloud storage at scale | File-level, not row-level; requires checkpoint and schema management |

---

## 1. Watermark-based incremental load

A watermark is a column—typically a `modified_date` timestamp or a monotonically increasing ID—that lets you read only rows newer than the last successfully processed value. The watermark value itself is stored in a small Delta table that acts as a control table.

### Implementation pattern

```python
from pyspark.sql import functions as F
from delta.tables import DeltaTable

WATERMARK_PATH = "/mnt/datalake/metadata/watermarks/orders"
TARGET_PATH = "/mnt/datalake/silver/orders"

def get_last_watermark():
    try:
        return (spark.read.format("delta").load(WATERMARK_PATH)
                .agg(F.max("watermark_value")).first()[0])
    except Exception:
        return "1900-01-01T00:00:00"

last_wm = get_last_watermark()

new_data = (
    spark.read.format("jdbc")
    .option("dbtable", f"(SELECT * FROM orders WHERE modified_date > '{last_wm}') AS q")
    .load()
)

if new_data.count() > 0:
    new_wm = new_data.agg(F.max("modified_date")).first()[0]

    if DeltaTable.isDeltaTable(spark, TARGET_PATH):
        (DeltaTable.forPath(spark, TARGET_PATH).alias("t")
         .merge(new_data.alias("s"), "t.order_id = s.order_id")
         .whenMatchedUpdateAll()
         .whenNotMatchedInsertAll()
         .execute())
    else:
        new_data.write.format("delta").save(TARGET_PATH)

    (spark.createDataFrame([(new_wm,)], ["watermark_value"])
     .write.format("delta").mode("overwrite").save(WATERMARK_PATH))
```

### Best practices for watermarks

- **Use a lookback window.** Even with a watermark, late-arriving data can carry timestamps older than the last watermark. Subtract a safety margin (e.g., 1 hour) from the last watermark when filtering, then deduplicate downstream. This trades a small amount of reprocessing for correctness.
- **The watermark column must be immutable and monotonic for the purpose it serves.** If your source updates the `modified_date` on every change, the watermark catches those updates. If it only sets `created_at` once, updates will be missed entirely.
- **Watermarking is not a substitute for CDC when deletes matter.** A timestamp filter cannot see hard deletes in the source. If the source physically removes rows, you need CDC or periodic snapshot comparison.

---

## 2. Delta `MERGE`: key-based upserts

`MERGE INTO` is the workhorse for incremental loads where the source sends a payload containing inserts, updates, and delete markers. It matches each source row against the target on a business key and applies the appropriate action in one atomic transaction.

### Full CDC payload pattern

When the source includes an operation column (`I`, `U`, `D`), `MERGE` can handle all three actions in a single statement:

```sql
MERGE INTO silver.customers AS target
USING (
  SELECT * FROM bronze.customers_cdc
  WHERE batch_id = current_batch_id()
) AS source
ON target.customer_id = source.customer_id
WHEN MATCHED AND source.operation = 'DELETE' THEN DELETE
WHEN MATCHED THEN UPDATE SET *
WHEN NOT MATCHED AND source.operation != 'DELETE' THEN INSERT *
```

The `WHEN NOT MATCHED AND source.operation != 'DELETE'` guard prevents delete markers for rows that never existed in the target from being inserted as new records.

### Python `DeltaTable` API

```python
from delta.tables import DeltaTable

(DeltaTable.forName(spark, "silver.customers").alias("t")
 .merge(df_updates.alias("s"), "t.customer_id = s.customer_id")
 .whenMatchedDelete(condition="s.operation = 'DELETE'")
 .whenMatchedUpdateAll(condition="s.operation = 'UPDATE'")
 .whenNotMatchedInsertAll(condition="s.operation != 'DELETE'")
 .execute())
```

### MERGE performance best practices

- **Enable low-shuffle merge.** Databricks has an optimized `MERGE` implementation that reduces shuffle operations by processing unmodified rows in a separate, streamlined mode. This is the default behavior in recent Databricks Runtime versions and substantially improves performance for common workloads.
- **Run `OPTIMIZE` regularly on heavily merged tables.** High-frequency upserts create many small Parquet files. Schedule `OPTIMIZE` at least daily on Silver tables that receive frequent `MERGE` operations. For Unity Catalog managed tables, enable Predictive Optimization to automate this.
- **Consider liquid clustering.** Databricks recommends liquid clustering for all new tables. It reduces the need for frequent `OPTIMIZE ZORDER BY` after `MERGE` operations and maintains query performance as the table grows.
- **Ensure the `ON` condition is selective.** A `MERGE` on a non-unique or low-selectivity key can degrade into a full cross-join-like operation. Verify that the source has at most one row per target key; in Databricks Runtime 16.0+, duplicate matches in the source raise an error unless handled explicitly.

---

## 3. Change Data Capture (CDC) in Delta Lake

CDC treats a database as a stream of changes rather than a static snapshot. In Delta Lake, two mechanisms provide CDC: **Change Data Feed (CDF)** on Delta tables, and **AUTO CDC** in Lakeflow Declarative Pipelines.

### Change Data Feed (CDF)

CDF records row-level changes (`insert`, `update_preimage`, `update_postimage`, `delete`) between Delta table versions, along with `_commit_version` and `_commit_timestamp` metadata columns. It must be enabled on the source table.

```sql
ALTER TABLE silver.orders
SET TBLPROPERTIES (delta.enableChangeDataFeed = true);
```

Read changes incrementally by version or timestamp:

```python
changes = (
    spark.read.format("delta")
    .option("readChangeFeed", "true")
    .option("startingVersion", last_processed_version + 1)
    .table("silver.orders")
)

inserts_updates = changes.filter(
    F.col("_change_type").isin(["insert", "update_postimage"])
)
deletes = changes.filter(F.col("_change_type") == "delete")
```

CDF is most powerful when combined with **Structured Streaming** and `foreachBatch` to apply changes to a downstream Delta table via `MERGE`. This creates a fully incremental Silver-to-Gold pipeline without reprocessing the full table.

**Best practices for CDF:**

- **Track the last processed version in a control table** so the pipeline knows where to resume after a restart. Do not rely on `_commit_timestamp` alone, because clock skew and concurrent writes can make timestamps non-monotonic across commits.
- **CDF is for Delta-to-Delta propagation.** It cannot capture changes from an external database. For external sources, you need a CDC connector (e.g., Lakeflow Connect) or a `MERGE`-based pattern.
- **CDF adds storage overhead** because it retains change data alongside the table. Enable it only on tables that downstream consumers actually read incrementally.

### AUTO CDC in Lakeflow Declarative Pipelines

AUTO CDC reduces a 150-line `MERGE` script to a few declarative lines. You declare the business key, the sequencing column, and whether the result should be SCD Type 1 (direct updates) or SCD Type 2 (history tracking). AUTO CDC handles out-of-order events automatically using the sequence column.

```sql
CREATE OR REFRESH STREAMING TABLE silver_customers;

CREATE FLOW customers_cdc_flow AS AUTO CDC
INTO silver_customers
FROM stream(bronze_customers_cdc)
KEYS (customer_id)
SEQUENCE BY event_timestamp
STORED AS SCD TYPE 1;
```

For SCD Type 2, include `__START_AT` and `__END_AT` columns in the target schema, with the same data type as the `SEQUENCE BY` column.

AUTO CDC is the recommended approach when you are building on **Lakeflow Declarative Pipelines** (formerly Delta Live Tables) and want declarative CDC without hand-written `MERGE` logic. It is not a replacement for CDF; CDF is the underlying Delta mechanism that surfaces changes, while AUTO CDC is a higher-level API for applying those changes to a streaming table.

---

## 4. Auto Loader: file-level incremental ingestion

Auto Loader (`cloudFiles`) incrementally discovers and processes **new files** in cloud storage (ADLS, S3, GCS) without reprocessing the entire directory. It is the standard Bronze-layer ingestion mechanism in Databricks.

### Basic Auto Loader stream

```python
(spark.readStream
 .format("cloudFiles")
 .option("cloudFiles.format", "parquet")
 .option("cloudFiles.schemaLocation", "/mnt/checkpoints/schema/orders")
 .load("/mnt/raw/orders")
 .writeStream
 .option("checkpointLocation", "/mnt/checkpoints/orders")
 .trigger(availableNow=True)   # batch-style incremental run
 .start("/mnt/bronze/orders"))
```

### Checkpoint and schema location best practices

- **Always specify an explicit checkpoint location.** Without it, Auto Loader may store checkpoints in `/tmp/checkpoints/<stream-id>/`, which is ephemeral and can cause the stream to lose state.
- **Store checkpoints and schema evolution information in Unity Catalog–managed storage.** Databricks recommends this for governance and reliability. The checkpoint location must live in a Unity Catalog external location, separate from the table data path.
- **Never apply cloud storage lifecycle policies to checkpoint directories.** Deleting checkpoint files corrupts the stream state and causes the stream to fail or reprocess data incorrectly.
- **Reuse the same checkpoint only when the query is logically identical.** Changing output mode (append ↔ complete/update), adding or removing stateful operations (aggregations, stream-stream joins), or changing the source format requires a new checkpoint location (or deletion of the old one).

### Schema inference and evolution

Specifying `cloudFiles.schemaLocation` enables schema inference and evolution. Auto Loader samples the first 50 GB or 1,000 files (whichever comes first) to infer the initial schema, then tracks schema changes over time in the `_schemas` directory.

Supported schema evolution modes (via `cloudFiles.schemaEvolutionMode`):

- **`addNewColumns`** (default): New columns are added to the schema; existing columns are not modified.
- **`addNewColumnsWithTypeWidening`**: Automatically widens types (e.g., `int` → `long`) when new data requires it.
- **`rescue`**: Puts mismatched or unexpected data into a JSON `_rescued_data` column instead of failing.
- **`failOnNewColumns`**: Fails the stream when a new column appears, requiring manual intervention.
- **`none`**: Ignores new columns.

For typed formats (Parquet, Avro), Auto Loader merges schemas across sampled files. For untyped formats (JSON, CSV, XML), it infers all columns as strings by default, which avoids type-mismatch failures but requires downstream casting.

### Auto Loader with `MERGE` for streaming upserts

Auto Loader ingests files, but it does not automatically deduplicate or upsert rows. To achieve row-level incremental loading from files, combine Auto Loader with `foreachBatch` and `MERGE`:

```python
def upsert_to_delta(batch_df, batch_id):
    (DeltaTable.forName(spark, "silver.orders").alias("t")
     .merge(batch_df.alias("s"), "t.order_id = s.order_id")
     .whenMatchedUpdateAll()
     .whenNotMatchedInsertAll()
     .execute())

(spark.readStream
 .format("cloudFiles")
 .option("cloudFiles.format", "json")
 .option("cloudFiles.schemaLocation", "/checkpoints/schema/orders")
 .load("/raw/orders")
 .writeStream
 .foreachBatch(upsert_to_delta)
 .option("checkpointLocation", "/checkpoints/orders")
 .trigger(availableNow=True)
 .start())
```

This pattern is common in Bronze-to-Silver transitions where raw files arrive incrementally but the target table must reflect deduplicated, key-based state.

---

## How the patterns fit together in a medallion pipeline

A typical production pipeline combines the mechanisms rather than choosing one:

```
Raw files (ADLS)
    │
    ▼
Auto Loader  →  Bronze (append-only, schema evolution)
    │
    ▼
CDF or MERGE  →  Silver (deduplicated, upserted, SCD)
    │
    ▼
AUTO CDC or CDF + MERGE  →  Gold (business aggregates, SCD Type 2)
```

- **Bronze**: Auto Loader with `availableNow=True` for scheduled incremental ingestion, or continuous triggers for streaming. Schema location and checkpoint are mandatory. Data is appended, not updated.
- **Silver**: Read Bronze changes via CDF (if Bronze is a Delta table and CDF is enabled) and apply `MERGE` to deduplicate and upsert. Alternatively, if the Bronze layer is a streaming table in Lakeflow, use AUTO CDC directly.
- **Gold**: Use CDF on Silver for incremental aggregation, or AUTO CDC for SCD Type 2 dimensions. Materialized views handle incremental aggregate refresh in Databricks SQL.

---

## Decision guide

**Start with Auto Loader if your source is files in cloud storage.** It is the only pattern designed for file-level incremental discovery and is the standard Bronze ingestion mechanism. Pair it with `MERGE` in `foreachBatch` when you need row-level upserts from those files.

**Use watermark filtering only when the source is append-only and you can tolerate missing late or out-of-order data.** It is the simplest pattern but the least robust. Always add a lookback window and deduplicate downstream.

**Use Delta `MERGE` when the source sends a full CDC payload (insert/update/delete flags) or when you need periodic upserts from a batch source.** Enable low-shuffle merge, schedule `OPTIMIZE`, and consider liquid clustering for new tables.

**Use CDF when the source is already a Delta table and you need row-level change propagation to a downstream Delta table.** Combine it with Structured Streaming and `foreachBatch` + `MERGE` for a fully incremental Silver-to-Gold pipeline.

**Use AUTO CDC when you are building on Lakeflow Declarative Pipelines and want declarative SCD Type 1/2 without hand-written `MERGE` logic.** It handles out-of-order events automatically and reduces maintenance overhead.

---

## Operational checklist

- [ ] Store all watermarks, last-processed versions, and checkpoints in a **Delta control table** or Unity Catalog–managed location, not in ephemeral storage.
- [ ] Add a **lookback window** to every watermark filter to catch late-arriving data.
- [ ] Enable **low-shuffle merge** and schedule **`OPTIMIZE`** on every table that receives frequent `MERGE` operations.
- [ ] Use **liquid clustering** for new Delta tables instead of manual `ZORDER`.
- [ ] Enable **CDF** only on Delta tables that downstream consumers read incrementally; track the last processed version explicitly.
- [ ] For Auto Loader: specify an **explicit checkpoint location**, keep it in Unity Catalog storage, and **exclude it from lifecycle policies**.
- [ ] Configure **schema evolution** (`addNewColumnsWithTypeWidening` or `rescue`) on Auto Loader streams; restart the stream via Lakeflow Jobs after schema changes.
- [ ] Validate that the **`ON` condition** in every `MERGE` matches at most one source row per target key; handle duplicates in the source before the merge.
- [ ] Monitor **small-file counts** and `MERGE` stage durations in the Spark UI; a rising small-file count is an early signal that compaction is lagging behind upsert frequency.
