# Behavioral: Troubleshooting a Critical Production Pipeline Failure

## Question
Describe a situation where a critical data pipeline failed in production. How did you troubleshoot, communicate, and resolve the issue?

---

## 1. Situation & Task
During a month-end financial closing window, our core billing aggregation pipeline failed at 03:30 AM with an out-of-memory error. Downstream executive dashboards were scheduled to be reviewed by leadership at 08:00 AM.

---

## 2. Action
1. **Immediate Triage:** Acknowledged the PagerDuty alert and joined the incident bridge, notifying stakeholders that the issue was under active triage with updates every 30 minutes.
2. **Root Cause Analysis (RCA):** Inspected the Spark UI and identified that month-end transaction volume had spiked 4x, causing heavy shuffle spills in the default 200 shuffle partitions.
3. **Emergency Fix:** Scaled the cluster compute, increased `spark.sql.shuffle.partitions` to `1,200`, and enabled Adaptive Query Execution (AQE).
4. **Target Partition Reprocessing:** Re-ran only the failed month-end partition slice using Delta Lake partition overwrite (`replaceWhere`).

---

## 3. Result
- Pipeline completed successfully by 06:15 AM (almost 2 hours ahead of the executive review).
- Hardened the pipeline by configuring dynamic autoscaling and automated partition sizing for future quarter-end spikes.
