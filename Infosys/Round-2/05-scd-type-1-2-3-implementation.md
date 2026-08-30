# Slowly Changing Dimensions (SCD Type 1, 2 & 3)

## Question
Explain the differences between **SCD Type 1**, **SCD Type 2**, and **SCD Type 3**, and explain which one you have implemented and why.

---

## 1. Summary Comparison

```
SCD Type 1 (Overwrite):
  [ Alice | Texas ]  ==Update==>  [ Alice | California ]  (Zero historical record)

SCD Type 2 (Add New Version Row - Full History):
  Row 1: [ Alice | Texas      | 2024-01-01 | 2026-08-30 | is_current = FALSE ]
  Row 2: [ Alice | California | 2026-08-30 | 9999-12-31 | is_current = TRUE  ]

SCD Type 3 (Add Previous Column - Partial History):
  [ Alice | Current_State: California | Previous_State: Texas ]
```

---

## 2. Implementation Case Study: SCD Type 2 in Delta Lake

**Why Implemented:** In retail and banking analytics, when a customer moves states, historical revenue for prior years must remain assigned to their previous state for accurate sales tax and regional analytics.

```sql
MERGE INTO dim_customer_scd2 AS target
USING (
  SELECT source.customer_id, source.address, source.state, current_timestamp() AS effective_date, NULL AS merge_key
  FROM customer_updates source
  JOIN dim_customer_scd2 target 
    ON source.customer_id = target.customer_id AND target.is_current = TRUE
  WHERE target.state <> source.state
  
  UNION ALL
  
  SELECT source.customer_id, source.address, source.state, current_timestamp() AS effective_date, source.customer_id AS merge_key
  FROM customer_updates source
) AS staged
ON target.customer_id = staged.merge_key AND target.is_current = TRUE
WHEN MATCHED THEN
  UPDATE SET target.end_date = staged.effective_date, target.is_current = FALSE
WHEN NOT MATCHED THEN
  INSERT (customer_id, address, state, start_date, end_date, is_current)
  VALUES (staged.customer_id, staged.address, staged.state, staged.effective_date, '9999-12-31', TRUE);
```
