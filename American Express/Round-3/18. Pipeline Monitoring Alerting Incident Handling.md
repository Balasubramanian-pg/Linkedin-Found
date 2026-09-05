# Production Data Pipeline Monitoring, Alerting & Incident Response Framework

## Question
How do you design an enterprise-grade monitoring, observability, and automated alerting framework for mission-critical payment data pipelines, and systematically handle production failures?

---

## 1. The 5 Pillars of Data Observability

```
+-------------------------------------------------------------------------------+
|                       Enterprise Data Observability Pillars                   |
+-------------------+-------------------+-------------------+-------------------+
|     Freshness     |      Volume       |      Schema       |      Quality      |
| (Pipeline Latency)| (Record Counts)   | (Drift & Breaking)| (Nulls/Duplicates)|
+-------------------+-------------------+-------------------+-------------------+
                                        │
                                        ▼
                  [ Central Observability & Lineage Layer ]
                  (Datadog / Prometheus / Azure Monitor / PagerDuty)
```

---

## 2. Monitoring & Metrics Instrumentation Architecture

```
[ Spark / Airflow Pipeline Run ]
               │
               ├── (1. Emits Custom Metrics via OpenTelemetry / StatsD)
               │      ├── `pipeline.duration.seconds`
               │      ├── `pipeline.records_ingested.count`
               │      └── `pipeline.dlq_records.count`
               │
               ▼
[ Azure Monitor / Prometheus / Datadog ]
               │
               ├── Evaluation against SLA thresholds & Dynamic Anomaly Bounds (±3σ)
               │
               ▼
[ Alert Routing Tier ]
   ├── P1 Critical (Data Stoppage / Revenue Impact) ──> PagerDuty (On-call phone call)
   ├── P2 Warning (SLA breach / Volume anomaly)     ──> MS Teams / Slack Urgent Channel
   └── P3 Info (Daily summary / Data Docs)          ──> Email / Operational Dashboard
```

---

## 3. Practical Implementation: PySpark Listener & Structured Logging

```python
import json
import logging
import time
from pyspark.sql import SparkSession

logger = logging.getLogger("PipelineObservability")
logging.basicConfig(level=logging.INFO)


class PipelineMonitor:
    def __init__(self, pipeline_name: str, batch_id: str):
        self.pipeline_name = pipeline_name
        self.batch_id = batch_id
        self.start_time = None

    def __enter__(self):
        self.start_time = time.time()
        self._emit_event("STARTED")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        duration = time.time() - self.start_time
        if exc_type is not None:
            self._emit_event("FAILED", duration=duration, error=str(exc_val))
            # Trigger Webhook Alert (Teams / PagerDuty)
            self._trigger_pagerduty_alert(str(exc_val), duration)
        else:
            self._emit_event("COMPLETED", duration=duration)

    def _emit_event(self, status: str, duration: float = 0.0, error: str = None):
        metric_payload = {
            "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ"),
            "pipeline": self.pipeline_name,
            "batch_id": self.batch_id,
            "status": status,
            "duration_seconds": round(duration, 2),
            "error_message": error
        }
        # Emits structured JSON to stdout (Scraped by FluentBit / Logstash / Datadog Agent)
        logger.info(json.dumps(metric_payload))

    def _trigger_pagerduty_alert(self, error_msg: str, duration: float):
        # Dispatches HTTPS PagerDuty Events API v2 payload
        pass
```

### Usage in Production ETL Script:
```python
with PipelineMonitor(pipeline_name="amex_settlement_clearing", batch_id="20260830_2200"):
    df = spark.read.table("bronze_settlements")
    # Processing logic...
    df.write.format("delta").mode("append").saveAsTable("silver_settlements")
```

---

## 4. Production Failure Runbook & Incident Handling

### Scenario A: Out of Memory (OOM) / Executor Killed
1. **Triage:** Check Spark UI &rarr; Executor tab to identify whether Driver or Executor died.
2. **Action:**
   - If single executor failed: Check Task Deserialization and max shuffle read size &rarr; indicates **Data Skew** &rarr; apply **Salting** or enable `spark.sql.adaptive.skewJoin.enabled`.
   - If Driver died: Verify no script called `.collect()` or `.toPandas()` on multi-million row datasets.

---

### Scenario B: Schema Drift / Incompatible Upstream Data
1. **Triage:** Check if ingestion failed on column type incompatibility.
2. **Action:**
   - Divert corrupt payloads to Dead Letter Queue (`quarantine_table`).
   - Allow valid rows to proceed to meet SLA.
   - Open ticket with upstream provider with sample corrupt JSON payload from DLQ.

---

### Scenario C: Pipeline Hang / Stuck Microbatch
1. **Triage:** Check thread dumps and active lock state on target Delta table.
2. **Action:**
   - Identify concurrent writes causing serialization conflicts.
   - Cancel zombie tasks via REST API / Spark UI.
   - Resume pipeline with checkpoint offsets intact.
