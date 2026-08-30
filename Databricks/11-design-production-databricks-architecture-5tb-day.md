# Designing a Production Databricks Architecture for 5 TB/Day

## Question
How do you architect a production Databricks Lakehouse processing 5 TB of multi-source data daily with strict SLAs, cost control, and high query performance?

---

## 1. Architectural Blueprint (Enterprise Medallion Lakehouse)

```
+---------------------------------------------------------------------------------------------------+
|                        5 TB/Day Enterprise Databricks Architecture                                |
+---------------------------------------------------------------------------------------------------+
  [ Sources: Kafka Streams, On-Prem DB CDC, Third-Party APIs, Microservice JSON drops ]
                                    │
                                    ▼ (ADF / Kafka Connect / Event Grid)
  ┌───────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥉 BRONZE LAYER (Raw Ingestion)                                                               │
  │ • Tech: Databricks Auto Loader (`cloudFiles`) with notification mode                          │
  │ • Storage: ADLS Gen2, append-only Delta with `_rescued_data` column & `ingestion_date`         │
  │ • Compute: Ephemeral Jobs cluster with Spot worker instances (`availableNow=True`)            │
  └───────────────────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (Micro-batch ETL: Cleaning, Deduplication, SCD2 Merge)
  ┌───────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥈 SILVER LAYER (Cleansed, Conformed, Enriched Entities)                                      │
  │ • Tech: Delta Live Tables (DLT) with expectations (`@dlt.expect_or_drop`)                     │
  │ • Deduplication: `MERGE INTO` with partition pruning predicates                               │
  │ • Clustering: Liquid Clustering (`CLUSTER BY (entity_id, event_date)`)                        │
  └───────────────────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ (Dimensional Modeling & Aggregations)
  ┌───────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥇 GOLD LAYER (Curated Star Schemas & Feature Stores)                                         │
  │ • Tech: Databricks SQL Serverless Warehouses with Photon Vectorized Engine                     │
  │ • Data Products: Daily Financial Fact Tables, Customer 360, Real-time BI Dashboards           │
  │ • Maintenance: Automated weekly `OPTIMIZE` and `VACUUM RETAIN 168 HOURS`                      │
  └───────────────────────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
  [ Consumption: Power BI DirectLake, Databricks Machine Learning / Feature Store, Data APIs ]
+---------------------------------------------------------------------------------------------------+
| Governance & Security: Unity Catalog (Fine-grained ABAC/RBAC, Table/Column Lineage, Masking)     |
+---------------------------------------------------------------------------------------------------+
```

---

## 2. Key Architecture Pillars for 5 TB/Day Scale
1. **Auto Loader with File Notifications:** Avoids costly directory listing scans on millions of blobs by leveraging Azure Event Grid and Queue Storage.
2. **Compute Segregation:** Dedicated single-job ephemeral clusters for heavy ETL; Serverless SQL warehouses with auto-stop (10 mins) for BI queries.
3. **Spot Instance Strategy:** Driver on On-Demand VM; 80% of worker nodes on Spot VMs with auto-scaling to slash compute costs by up to 60%.
