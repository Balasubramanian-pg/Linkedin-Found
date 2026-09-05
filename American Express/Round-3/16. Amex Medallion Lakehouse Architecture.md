# Medallion Lakehouse Architecture with Data Quality SLAs

## Question
How do you structure an enterprise Medallion Lakehouse Architecture (Bronze, Silver, Gold) for a financial institution like American Express, including schema evolution, retention policies, and formal Data Quality SLAs?

---

## 1. Architectural Layout & Data Flow

```
+------------------------------------------------------------------------------------------------+
|                         Enterprise Medallion Architecture Layout                               |
+------------------------------------------------------------------------------------------------+
  [ Source Systems ]
  - Payment Gateways (Kafka / Event Hubs)
  - Core Banking Mainframe (DB2 / Oracle via CDC)
  - Partner REST APIs / Credit Bureaus
           │
           ▼
  ┌────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥉 BRONZE LAYER (Raw Ingestion / Landing)                                                  │
  │ • Storage: Append-only raw Delta tables with full audit headers (`_file_name`, `_ingested`) │
  │ • Retention: 7 Years (Regulatory compliance / PCI-DSS / Replayability)                     │
  │ • SLA: Ingestion Latency < 2 minutes | Zero data loss guarantee (100% durability)          │
  └────────────────────────────────────────────────────────────────────────────────────────────┘
           │
           ▼ (Validation, Cleaning, Struct Flattening, Type Casting, Deduplication)
  ┌────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥈 SILVER LAYER (Cleaned, Conformed, Enriched Enterprise Dimensions & Facts)               │
  │ • Storage: Delta tables with SCD Type 1/2 upserts via `MERGE INTO`                         │
  │ • Data Quality: DLT Expectations (Non-null PKs, Foreign key referential integrity)        │
  │ • Retention: 3 - 5 Years active storage                                                    │
  │ • SLA: Processing Latency < 15 minutes | Data Accuracy > 99.999%                           │
  └────────────────────────────────────────────────────────────────────────────────────────────┘
           │
           ▼ (Domain Aggregations, Business Metrics, Feature Engineering, Star Schemas)
  ┌────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥇 GOLD LAYER (Curated Business & Semantic Consumption Models)                             │
  │ • Storage: Star Schema Fact/Dim tables, Z-Ordered & Liquid Clustered for BI queries         │
  │ • Consumers: Power BI DirectLake, Executive Dashboards, Real-time Fraud ML Feature Store   │
  │ • Retention: 3 Years active aggregates + Cold Archive Tier                                 │
  │ • SLA: Available by 06:00 AM UTC daily | Sub-second query response time                    │
  └────────────────────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Layer-by-Layer Detailed Specification

### A. Bronze Layer (Raw & Unaltered)
- **Role:** Immutable single source of truth. Data is stored in raw JSON/Avro/Parquet representation inside Delta Lake.
- **Audit Columns Added:**
  - `_input_file_name`: Source file name.
  - `_ingestion_timestamp`: Server UTC ingestion timestamp.
  - `_batch_id`: Unique pipeline execution run identifier.
- **Rule:** Never apply business transformations or destructive filtering in Bronze.

---

### B. Silver Layer (Conformed & Validated)
- **Role:** Enterprise-wide single version of the truth.
- **Transformations Applied:**
  - Column name standardization (`camelCase` &rarr; `snake_case`).
  - Strict type casting and null imputation.
  - De-identification / Tokenization of PCI card numbers (PAN tokenization / SHA-256 hashing).
  - SCD Type 2 dimension versioning.
- **Data Quality Expectations:** Bad records quarantined to Dead Letter Queue (DLQ) without interrupting processing.

---

### C. Gold Layer (Business Aggregates & Star Schemas)
- **Role:** High-performance analytical consumption models.
- **Data Products:**
  - `Fact_Daily_Card_Spend` (Aggregated by cardholder, MCC merchant category, geographic region).
  - `Dim_Customer_360` (Consolidated customer credit profile, active status, lifetime value).
  - Real-time features for Machine Learning fraud detection models.

---

## 3. Data Quality SLAs & Observability Matrix

| Metric | Bronze Layer SLA | Silver Layer SLA | Gold Layer SLA |
| :--- | :--- | :--- | :--- |
| **Freshness (Latency)** | $\le 2\text{ min}$ (Streaming) / $1\text{ hr}$ (Batch) | $\le 15\text{ min}$ | Daily by 06:00 AM UTC |
| **Completeness** | 100% of raw bytes recorded | 0% nulls on primary/foreign keys | Zero missing dimension mappings |
| **Accuracy** | Payload matches source checksum | Business rules verified | Financial reconciliations balanced to \$0.00 |
| **Schema Integrity** | Schema evolution enabled | Strict schema enforced | Frozen semantic schema |
