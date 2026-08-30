# Triggers in Azure Data Factory (ADF) & Efficient Scheduling

## Question
Explain the different types of triggers in Azure Data Factory (ADF) and how to schedule pipelines efficiently to prevent cluster cold-start delays and queue blocking.

---

## 1. Types of ADF Triggers

| Trigger Type | Mechanism | Primary Use Case |
| :--- | :--- | :--- |
| **Schedule Trigger** | Executes on a wall-clock calendar schedule (e.g., Every day at 06:00 AM UTC). | Regular daily/weekly batch ETL jobs. |
| **Tumbling Window Trigger**| Operates over continuous, non-overlapping time slices. Supports **self-dependency** and automatic historical backfills. | Hourly / daily incremental watermarking where order of execution matters. |
| **Storage Event Trigger** | Fires automatically when a blob is created or deleted in Azure Blob / ADLS Gen2 (`BlobCreated`). | Event-driven real-time file ingestion. |
| **Custom Event Trigger** | Subscribes to custom Azure Event Grid topics. | Microservice-driven workflow orchestration. |

---

## 2. Tumbling Window Self-Dependency for Safe Ordered Loads

A Tumbling Window trigger can be configured with a **Self-Dependency**:
- If the `10:00 - 11:00` window fails, the `11:00 - 12:00` window will automatically wait and not execute until the previous slice succeeds.
