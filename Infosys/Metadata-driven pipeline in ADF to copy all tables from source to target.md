A metadata-driven pipeline in Azure Data Factory replaces dozens (or hundreds) of hard-coded copy pipelines with **one generic pipeline** that reads a control table, iterates over the table list, and resolves the source and destination for each table at runtime. The official Microsoft pattern describes this as the appropriate approach when you need to copy “huge amounts of objects (for example, thousands of tables) or load data from large variety of sources”. Instead of redeploying pipelines to add or remove a table, you update a row in the control table.

The sections below cover the architecture, the control table design, the step-by-step pipeline construction, scaling considerations, and production best practices.

---

## Architecture overview

<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/29dd2161-3ae7-4fc0-a552-135278957586" />

The pattern has four moving parts:

1. **Control table** (in Azure SQL Database or another queryable store) holds one row per table to copy, with columns for source schema, source table, target path/table, load type, watermark column, active flag, and any copy behavior settings.
2. **Lookup activity** reads the control table and returns the list of tables as a JSON array.
3. **ForEach activity** iterates over that array, with a configurable **batch count** for parallel execution.
4. **Copy activity** (inside the ForEach) receives the current row’s values as parameters and uses **parameterized datasets** for both source and sink, so the same Copy activity handles any table in the list.

A **parent pipeline** orchestrates this flow, and optionally delegates to **modular child pipelines** for different transfer types (database-to-database, file-to-storage, etc.). Logging and watermark updates are written back to the control database after each copy completes.

---

## Designing the control table

The control table is the single source of truth for what gets copied and how. A practical schema includes:

| Column | Purpose |
|---|---|
| `id` | Surrogate key |
| `source_schema` | Schema name (e.g., `Sales`, `dbo`) |
| `source_table` | Table name |
| `target_schema` | Target schema or folder path |
| `target_table` | Target table name or file name pattern |
| `load_type` | `Full` or `Incremental` (delta) |
| `watermark_column` | Column used for incremental filtering (e.g., `ModifiedDate`) |
| `last_watermark_value` | Stored value from the previous successful run |
| `is_active` | Flag to include/exclude a table without deleting the row |
| `copy_behavior` | Optional settings (e.g., fault tolerance, write mode) |

Microsoft’s Copy Data tool generates two control tables: a **main control table** for the object list and copy behaviors, and a **connection control table** for linked-service values when parameterized linked services are used. For most implementations, one control table with connection references (e.g., Key Vault secret names rather than raw credentials) is sufficient.

The InfoWorld architectural case study recommends separating **Connections**, **CopyOperations**, and **ETLConfig** into distinct tables if you need to support many source types and reusable connection definitions. For a simpler “copy all tables” scenario, a single table with source and target identifiers is adequate.

---

## Building the pipeline

### Step 1: Create parameterized datasets

You need at least one **source dataset** and one **sink dataset**, each with parameters for the schema and table (or path). For an Azure SQL source, create dataset parameters `schemaName` and `tableName`, and in the connection settings replace the hard-coded table with `@dataset().schemaName.@dataset().tableName` (or use the equivalent expression for your connector).

For a data lake sink, parameterize the **folder path** and **file name** so that each table lands in its own directory:

```text
@concat('bronze/', item().source_schema, '/', item().source_table, '/')
```

The same parameterization principle applies to the sink table name if the target is a database:

```text
@dataset().target_schema.@dataset().target_table
```

### Step 2: Configure the Lookup activity

Add a **Lookup** activity with a query against the control table:

```sql
SELECT
    source_schema,
    source_table,
    target_schema,
    target_table,
    load_type,
    watermark_column,
    last_watermark_value
FROM dbo.control_table
WHERE is_active = 1
ORDER BY id;
```

Set **First row only** to `false` so that Lookup returns the full array. The output is a JSON array accessible as `@activity('LookupControlTable').output.value`.

**Critical limit:** The Lookup activity has a hard limit of **5,000 rows** and **4 MB** of output. If the result set exceeds 5,000 rows, only the first 5,000 are returned, and if the output exceeds 4 MB, the activity fails. For environments with more than a few thousand tables, you must either:

- **Batch the control table** into chunks (e.g., using a stored procedure that returns one batch per run), or
- **Use a different discovery mechanism** (e.g., a stored procedure that returns a compact JSON payload), or
- **Split the pipeline into multiple parallel branches**, each handling a subset of the control table.

Microsoft’s Copy Data tool exposes a parameter `MaxNumberOfObjectsReturnedFromLookupActivity` to control this threshold.

### Step 3: Configure the ForEach activity

Set the **Items** property of the ForEach to the Lookup output:

```text
@activity('LookupControlTable').output.value
```

Inside the ForEach, the current row is accessed as `@item()`, so `@item().source_table`, `@item().target_table`, and so on are available to downstream activities.

**Batch count and parallelism.** The ForEach **Batch count** controls how many items are processed in parallel. The default is 20, and the maximum is **50**. Microsoft’s documentation notes that the number of concurrent copy activities is configurable, with a default of 20. Start with a moderate batch count (e.g., 5–10) and increase it based on source and sink throughput limits. A batch count that is too high can saturate the source database or the integration runtime.

### Step 4: Configure the Copy activity with dynamic parameters

Inside the ForEach, add a **Copy** activity. On the **Source** tab, select the parameterized source dataset and pass:

- `schemaName` → `@item().source_schema`
- `tableName` → `@item().source_table`

On the **Sink** tab, select the parameterized sink dataset and pass the target parameters. For a database sink:

- `target_schema` → `@item().target_schema`
- `target_table` → `@item().target_table`

For a file sink, build the folder path and file name from the current item.

If you need a **query-based source** (e.g., for incremental loads), use the **Query** option in the Copy activity source and build a dynamic SQL string:

```text
@concat('SELECT * FROM ', item().source_schema, '.', item().source_table,
        ' WHERE ', item().watermark_column, ' > ''', item().last_watermark_value, '''')
```

### Step 5: Handle full load vs. incremental load

The control table’s `load_type` column allows a single pipeline to handle both modes.

- **Full load**: Copy the entire table on every run. Use the sink’s **Write behavior** as `insert` (append) or `overwrite` depending on whether you want a fresh snapshot each time.
- **Incremental load**: Filter the source by the watermark column and update the `last_watermark_value` in the control table after a successful copy. Microsoft’s generated scripts include a stored procedure specifically for updating the watermark value in the main control table when incremental copy jobs complete.

For incremental loads, the Copy activity source query uses the current row’s `watermark_column` and `last_watermark_value`. After the copy succeeds, call a stored procedure (or use a Script activity) to write the new maximum watermark value back to the control table. This write-back should be part of the same pipeline logic, typically in the **On Success** path of the Copy activity.

A practical safeguard is to subtract a **lookback window** from the stored watermark (e.g., 1 hour) to catch late-arriving data, then deduplicate downstream. Without a lookback, rows that arrive with a timestamp older than the last watermark are permanently missed.

### Step 6: Add error handling and logging

Wrap the Copy activity in a **try–catch pattern** so that a failure on one table does not abort the entire ForEach loop. The standard approach is:

1. Put the Copy activity inside a **scope** (or use an `If Condition` / `Switch` structure).
2. On failure, route to a **logging activity** (e.g., a stored procedure call or a Script activity) that writes the table name, error message, timestamp, and status to a `pipeline_log` table.
3. On success, log the success status and update the watermark.

The InfoWorld case study describes exactly this: comprehensive logging of start times, end times, statuses, and errors in Azure SQL Database, with Application Insights for real-time monitoring and a utility pipeline for alerts. A logging table with columns for `unique_id`, `table_name`, `status`, `error_message`, and `rows_copied` gives you an audit trail and the ability to **resume from failed tables** without rerunning the entire pipeline.

---

<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/bb657bb2-6dc4-403f-87aa-235d58add937" />

## Scaling considerations

### Lookup limit and batching

The 5,000-row Lookup limit is the most common scaling bottleneck. For control tables larger than 5,000 rows, consider:

- **Adding a batch identifier** to the control table (e.g., `batch_id`) and running the pipeline once per batch.
- **Using a stored procedure** that returns a paginated JSON result, called by the Lookup activity.
- **Splitting by source system or schema** into separate control tables and running parallel parent pipelines.

### ForEach batch count and throughput

The ForEach batch count controls concurrency, but it is not dynamically adjustable at runtime—the maximum is hard-coded at 50. For very large table counts, a **two-level batching pattern** works well: an outer ForEach iterates over batches of tables (e.g., 50 per batch), and an inner ForEach (or parallel Copy activities) processes the tables within the batch.

Also consider the **Copy activity’s own parallelism** settings (DIU or parallel copies) for large tables. A single large table may benefit more from higher DIU than from running many small tables concurrently.

### Modular child pipelines

The InfoWorld architecture uses **modular child pipelines** for different transfer types: one child pipeline for database-to-database transfers, another for file-to-storage, etc. The parent pipeline reads the metadata and delegates to the appropriate child pipeline, passing the necessary parameters. This keeps each pipeline template simple and avoids a monolithic pipeline with complex branching logic. The same case study notes that ADF’s **Switch activity is capped at 25 cases**, so for many transfer types, a categorization layer or multiple parent pipelines may be needed.

<img width="1376" height="768" alt="image" src="https://github.com/user-attachments/assets/3eb67fae-6171-493b-b222-1bea686899a4" />


---

## Security and credential management

The control table should **never store raw passwords or connection strings**. Instead:

- Use **parameterized linked services** with credentials stored in **Azure Key Vault**. The control table or a separate connection table stores the **secret name**, not the secret value.
- Reference Key Vault secrets in linked service definitions, and grant the ADF managed identity `Get` permission on the vault.
- The InfoWorld case study emphasizes integrating Key Vault early so that “sensitive data remains encrypted and inaccessible outside of authorized processes”.

---

## Production checklist

- [ ] **Control table has an `is_active` flag** so tables can be excluded without deleting metadata.
- [ ] **Lookup output is within the 5,000-row / 4 MB limit**; if not, implement batching or a stored-procedure-based discovery mechanism.
- [ ] **ForEach batch count is tuned** to source and sink capacity (start at 5–10, maximum 50).
- [ ] **Copy activity uses parameterized datasets** for both source and sink; no hard-coded table names remain in the pipeline.
- [ ] **Incremental loads use a watermark column** and write the new watermark back to the control table after success, with a lookback window for late-arriving data.
- [ ] **Try–catch / failure routing is implemented** so one table failure does not stop the loop; failures are logged to a `pipeline_log` table.
- [ ] **Credentials are referenced via Key Vault**, not stored in the control table or pipeline parameters.
- [ ] **Monitoring and alerting** are configured (Application Insights or Azure Monitor alerts on pipeline failure).
- [ ] **Idempotency is considered**: rerunning a failed batch should not duplicate data. Use overwrite semantics for full loads, or watermark-based upserts for incremental loads.

The metadata-driven pattern’s core value is **maintainability at scale**: adding a table is a control-table insert, not a pipeline deployment. The official Microsoft guidance frames this as the single place to see which objects are copied by which pipelines and triggers, with defined copy behaviors. The architectural case study reinforces that the upfront investment in a metadata schema and modular child pipelines pays off in reduced maintenance and faster adaptation to new sources.
