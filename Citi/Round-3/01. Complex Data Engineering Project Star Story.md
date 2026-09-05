# Behavioral: Complex Data Engineering Project (STAR Method)

## Question
Tell me about a complex data engineering project you worked on, the architectural challenges you faced, and the business impact delivered.

---

## 1. Situation
At my previous financial institution, the legacy overnight batch settlement pipeline was failing to meet its 06:00 AM SLA. Ingesting ~1.5 TB of daily multi-asset trade records across 14 fragmented on-premises databases took over 8 hours, causing morning risk dashboard delays for trading desks.

---

## 2. Task
As the Lead Data Engineer, my objective was to migrate and re-architect the batch processing pipeline to **Azure Databricks and Delta Lake (Medallion Architecture)**, guaranteeing completion within 45 minutes with 100% financial accuracy and automated data reconciliation.

---

## 3. Action
1. **Engineered Incremental Auto Loader Ingestion:** Replaced full daily table dumps with metadata-driven incremental ADF copy pipelines and Databricks Auto Loader (`cloudFiles`).
2. **Delta Lake Medallion Design:** Structured Bronze (raw append), Silver (deduplicated & conformed with `MERGE INTO`), and Gold (Z-Ordered star schemas).
3. **Resolved Shuffle Spills & Data Skew:** Identified key skew in high-frequency trading desks and applied **Salting** techniques and tuned Adaptive Query Execution (AQE).
4. **CI/CD & Secret Security:** Built automated Azure DevOps deployment pipelines with zero plain-text secrets using Azure Key Vault secret scopes.

---

## 4. Result
- **Performance:** Reduced end-to-end pipeline execution time from **8 hours to 38 minutes** (a 12x performance improvement).
- **Reliability:** Delivered 99.99% pipeline uptime and eliminated morning risk report delays.
- **Cost Reduction:** Auto-scaling single-node driver/worker pools reduced monthly Azure cloud compute costs by 35%.
