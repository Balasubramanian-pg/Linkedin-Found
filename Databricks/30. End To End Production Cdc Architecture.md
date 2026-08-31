# Designing an End-to-End Production CDC Architecture

## Question
Design an end-to-end Change Data Capture (CDC) architecture streaming database changes from an OLTP database (Oracle / SQL Server) into a Databricks Lakehouse with zero data loss.

---

## 1. End-to-End CDC Architecture Diagram

```
+---------------------------------------------------------------------------------------------------+
|                              End-to-End Production CDC Architecture                               |
+---------------------------------------------------------------------------------------------------+
  [ Source OLTP: Oracle DB / SQL Server ]
                 │ (Database Redo / Transaction Logs)
                 ▼
  [ Debezium / Qlik Replicate / Azure SQL CDC ]
                 │ (Emits JSON / Avro change event stream)
                 ▼
  [ Message Broker: Apache Kafka Cluster / Azure Event Hubs ]
                 │
                 ▼ (Databricks Structured Streaming with Auto Loader / Kafka source)
  ┌───────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥉 BRONZE DELTA TABLE (Append-Only Change Audit Ledger)                                       │
  │ • Captures raw CDC envelope: `_op` ('I', 'U', 'D'), `_source_ts`, `_lsn`, `before`, `after`  │
  └───────────────────────────────────────────────────────────────────────────────────────────────┘
                 │
                 ▼ (Micro-batch Deduplication & Sequence Ranking)
  ┌───────────────────────────────────────────────────────────────────────────────────────────────┐
  │ 🥈 SILVER DELTA TABLE (Real-Time Replicated Current State)                                    │
  │ • Applies Delta `MERGE INTO` with LSN sequence checks and hard/soft deletion handling         │
  │ • Delta Change Data Feed (CDF) enabled for downstream consumers                               │
  └───────────────────────────────────────────────────────────────────────────────────────────────┘
                 │
                 ▼
  [ 🥇 GOLD CONSUMPTION LAYER: Star Schema Dimensional Models, Power BI DirectLake, Feature Stores ]
+---------------------------------------------------------------------------------------------------+
```

---

## 2. Key Resiliency & Integrity Rules
1. **Never Merge Directly from Kafka to Silver:** Always land CDC records into **Bronze Delta** first to maintain an immutable audit trail and enable historical replays.
2. **LSN Ordering Condition:** Ensure older out-of-order Kafka events cannot overwrite newer database states.
3. **Enable Delta Change Data Feed (CDF):** Allows downstream Gold models to consume row-level deltas without re-scanning full Silver tables.
