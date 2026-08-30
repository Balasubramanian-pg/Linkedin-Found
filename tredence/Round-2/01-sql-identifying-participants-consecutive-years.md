# SQL: Identifying Participants with Consecutive Years (Gaps and Islands)

## Question
Given a table of competition participants and the years they participated, write an SQL query to identify all participants who participated for **3 or more consecutive years**.

---

## 1. Solution: Classic Gaps and Islands Technique

By subtracting a continuous `ROW_NUMBER()` from the `participation_year`, all consecutive years produce the exact same constant difference value (`island_group`):

```sql
WITH DistinctParticipations AS (
    -- Step 1: Ensure 1 record per participant per year
    SELECT DISTINCT 
        participant_id, 
        participant_name, 
        participation_year
    FROM tournament_history
),
NumberedSequences AS (
    -- Step 2: Compute island group
    SELECT
        participant_id,
        participant_name,
        participation_year,
        participation_year - ROW_NUMBER() OVER (
            PARTITION BY participant_id 
            ORDER BY participation_year
        ) AS island_group
    FROM DistinctParticipations
),
ConsecutiveStreaks AS (
    -- Step 3: Count streak length per island
    SELECT
        participant_id,
        participant_name,
        MIN(participation_year) AS streak_start_year,
        MAX(participation_year) AS streak_end_year,
        COUNT(*) AS consecutive_years_count
    FROM NumberedSequences
    GROUP BY 
        participant_id, 
        participant_name, 
        island_group
    HAVING 
        COUNT(*) >= 3
)
SELECT 
    participant_id,
    participant_name,
    streak_start_year,
    streak_end_year,
    consecutive_years_count
FROM ConsecutiveStreaks
ORDER BY participant_id, streak_start_year;
```
