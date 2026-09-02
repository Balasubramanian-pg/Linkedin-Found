# Idempotent Payment Ingestion Pipeline: Replay, Retries & Exactly-Once Semantics

## Question
How do you architect an end-to-end payment transaction ingestion pipeline that guarantees **strict idempotency**, handles **automated retries with Dead Letter Queues (DLQ)**, and supports **historical replays** without double-charging or duplicate record generation?

## 1. Architectural Blueprint

```
+------------------------------------------------------------------------------------------------+
|                             Idempotent Payment Ingestion Pipeline                              |
+------------------------------------------------------------------------------------------------+
  [ Payment Terminals / Webhooks ]
                │
                ▼ (TLS + Signature Verification)
  [ API Gateway / Ingestion Service ]
        ├── Generates / Verifies `Idempotency-Key` (UUIDv4) in Redis (TTL: 24h)
        └── Publishes to Apache Kafka / Azure Event Hubs (Partitioned by `cardholder_id`)
                │
                ▼
  [ Spark Structured Streaming / Delta Engine ]
        ├── Ingests with Checkpointing & Write-Ahead Logs (WAL)
        ├── Schema Validation & Business Rule Validation
        │      ├── Valid Record   ──> Atomic Delta `MERGE INTO` (Target Table)
        │      └── Malformed / DLQ ──> Kafka DLQ Topic / Quarantine Delta Table
        └── Emits Audit Metrics to Prometheus / CloudWatch
+------------------------------------------------------------------------------------------------+
```

## 2. Core Pillars of Idempotent Design

### Pillar 1: Idempotency Key at the Edge
- Every incoming transaction request must carry a client-generated `Idempotency-Key` header (e.g., `Idempotency-Key: 9b1deb4d-3b7d-4bad-9bdd-2b0d7b3dcb6d`).
- The API gateway checks a distributed Redis cache:
  - If the key exists: Returns the cached response immediately without re-executing.
  - If the key does not exist: Sets the key with a `SETNX` lock and 24-hour expiration.

### Pillar 2: Deterministic Delta Lake Upsert (`MERGE INTO`)
In the storage layer, downstream writers must ensure that re-running a batch produces the exact same state without duplicate records:

```sql
MERGE INTO gold_fact_payments AS target
USING (
    SELECT 
        transaction_id,
        idempotency_key,
        cardholder_id,
        merchant_id,
        amount,
        currency,
        auth_status,
        event_timestamp,
        current_timestamp() AS ingestion_timestamp
    FROM staged_payment_microbatch
    -- Deduplicate within the microbatch itself first
    QUALIFY ROW_NUMBER() OVER (
        PARTITION BY transaction_id 
        ORDER BY event_timestamp DESC
    ) = 1
) AS source
ON target.transaction_id = source.transaction_id
WHEN MATCHED AND target.auth_status <> source.auth_status THEN
  UPDATE SET 
    target.auth_status = source.auth_status,
    target.last_modified_at = source.ingestion_timestamp
WHEN NOT MATCHED THEN
  INSERT (
    transaction_id, idempotency_key, cardholder_id, merchant_id, 
    amount, currency, auth_status, event_timestamp, created_at
  )
  VALUES (
    source.transaction_id, source.idempotency_key, source.cardholder_id, source.merchant_id,
    source.amount, source.currency, source.auth_status, source.event_timestamp, source.ingestion_timestamp
  );
```

## 3. Automated Retry Strategy & Dead Letter Queue (DLQ)

```
[ Incoming Event ] ──> Try Processing
                           │
                           ├── Success ──> Write to Delta Lake
                           │
                           └── Failure ──> Retry 1 (Delay: 1s)
                                              ├── Retry 2 (Delay: 4s)
                                              └── Retry 3 (Delay: 16s)
                                                    └── Exceeded Max Retries ──> [ Dead Letter Queue (DLQ) ]
```

1. **Transient Errors (Network timeouts, throttling):** Retried automatically using **Exponential Backoff with Jitter** (`delay = min(max_delay, base * 2^attempt + rand(0, 1))`).
2. **Deterministic Poison Pills (Invalid JSON, corrupt types):** Routed immediately to the DLQ table (`dlq_payment_failures`) with error stack trace and raw payload preserved.

## 4. Replay & Backfill Mechanism

When bug fixes are deployed or source data is re-sent:
1. **Time-Bound Replay:** Reprocess the Kafka stream starting from a specific timestamp/offset:
   ```python
   spark.readStream.format("kafka") \
       .option("startingOffsetsByTimestamp", """{"payments_topic":{"0": 1756540800000}}""") \
       .load()
   ```
2. **Partition Overwrite / Delta Merge:** Because the downstream writes use `MERGE INTO` keyed on `transaction_id`, replaying old data updates existing records without creating duplicates.
