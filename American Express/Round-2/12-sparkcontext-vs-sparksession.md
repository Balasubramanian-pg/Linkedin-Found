# Spark Architecture: SparkContext vs SparkSession

## Question
What is the difference between `SparkContext` and `SparkSession` in Apache Spark, and why was `SparkSession` introduced in Spark 2.0+?

---

## 1. Evolution from Spark 1.x to Spark 2.x+

In early versions of Apache Spark (1.x), developers had to instantiate and juggle multiple fragmented context objects depending on the data source and API:

```
Spark 1.x (Fragmented Contexts):
           ┌──> SparkContext (Core RDD API)
           ├──> SQLContext (Structured SQL & DataFrames)
Master ────┼──> HiveContext (Hive Metastore & HQL)
           └──> StreamingContext (DStreams / Spark Streaming)

Spark 2.x+ / 3.x (Unified Entry Point):
Master ────> [ SparkSession ] (Unified entry point encapsulating all contexts)
                ├── SparkContext (`spark.sparkContext`)
                ├── SQLContext / Hive Metastore
                ├── Catalog Interface (`spark.catalog`)
                └── Structured Streaming (`spark.readStream`)
```

---

## 2. Comparison Matrix

| Feature | SparkContext | SparkSession |
| :--- | :--- | :--- |
| **Introduced In** | Spark 1.0 (Original Core Engine) | Spark 2.0 |
| **Primary Abstraction** | **RDD (Resilient Distributed Dataset)** | **DataFrame, Dataset, SQL, Streaming** |
| **Scope** | Low-level cluster resource connection and distributed task execution. | High-level unified interface combining SQL, Hive, Streaming, and RDDs. |
| **Hive Integration** | Required creating a distinct `HiveContext(sc)`. | Seamlessly enabled via `.enableHiveSupport()`. |
| **Catalog Access** | None (manual schema / metastore interactions). | Rich metadata catalog interface (`spark.catalog.listTables()`). |
| **Multiple User Sessions**| Single `SparkContext` per JVM. | Multiple isolated `SparkSession` instances sharing 1 underlying `SparkContext` via `.newSession()`. |

---

## 3. Practical Code Examples

### Initializing a Production `SparkSession`:
```python
from pyspark.sql import SparkSession

# Build unified session with Hive support and Delta Lake extensions
spark = (
    SparkSession.builder
    .appName("Amex_Payment_Clearing_Pipeline")
    .master("yarn") # or "local[*]" / "k8s"
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension")
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog")
    .enableHiveSupport()
    .getOrCreate()
)

# Accessing underlying low-level SparkContext if RDD operations are required:
sc = spark.sparkContext
rdd = sc.parallelize([1, 2, 3, 4, 5])

# Accessing the Catalog API
tables = spark.catalog.listTables("default")
print([t.name for t in tables])
```

---

## 4. Multi-Tenant Isolated Sessions (`newSession`)
In multi-tenant environments (e.g., Databricks or shared web servers), different users can create independent `SparkSession` objects with their own temporary views and SQL configurations while sharing the same active cluster resources:

```python
# Create an isolated session sharing the existing SparkContext
user_session = spark.newSession()
user_session.conf.set("spark.sql.shuffle.partitions", "50")
```
