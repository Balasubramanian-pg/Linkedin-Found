# Monitoring, Automated Retries & Failure Handling in ADF & Databricks

## Question
How do you monitor pipeline execution, implement automated retry policies, and configure alerting in Azure Data Factory (ADF) and Azure Databricks?

---

## 1. Retry Policies in ADF Activities
In every ADF Copy Activity and Databricks Notebook Activity:
- **Retry Count:** Set to `3`.
- **Retry Interval (Seconds):** Set to `60` seconds (with exponential backoff where available) to absorb transient network blips and API throttling.

---

## 2. Centralized Monitoring & Alerting Architecture

```
[ ADF Pipeline / Databricks Job Run ]
                  │
                  ▼ (Diagnostic Settings)
[ Azure Log Analytics Workspace ]
                  │
                  ▼ (Kusto Query Language - KQL Alert Rule)
[ Azure Monitor Action Groups ]
                  ├──> PagerDuty / On-Call Incident
                  ├──> Microsoft Teams / Slack Webhook Channel
                  └──> Email Distribution List
```

### KQL Query for Failed ADF Pipeline Alert:
```kusto
ADFPipelineRun
| where Status == "Failed"
| project TimeGenerated, PipelineName, ErrorMessage, Parameters
| order by TimeGenerated desc
```
