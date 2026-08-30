# SQL: Scheduling Matches Between Teams (Round-Robin Fixtures)

## Question
Given a table of sports teams, write an SQL query to generate a complete tournament schedule where every team plays every other team exactly once (no self-matches, no duplicate fixtures like Team A vs Team B and Team B vs Team A).

---

## 1. Solution: Self-Join with Strict Inequality (<)

Using `t1.team_id < t2.team_id` prevents pairing a team with itself and eliminates reverse duplicate match fixtures:

```sql
SELECT 
    t1.team_name AS home_team,
    t2.team_name AS away_team,
    CONCAT(t1.team_name, ' vs ', t2.team_name) AS match_fixture
FROM teams t1
INNER JOIN teams t2 
    ON t1.team_id < t2.team_id
ORDER BY 
    t1.team_id, 
    t2.team_id;
```

---

## 2. Sample Data & Walkthrough

### Input Table (`teams`):
| team_id | team_name |
| :--- | :--- |
| 1 | India |
| 2 | Australia |
| 3 | England |
| 4 | South Africa |

### Generated Output:
| home_team | away_team | match_fixture |
| :--- | :--- | :--- |
| India | Australia | India vs Australia |
| India | England | India vs England |
| India | South Africa | India vs South Africa |
| Australia | England | Australia vs England |
| Australia | South Africa | Australia vs South Africa |
| England | South Africa | England vs South Africa |

---

## 3. Home & Away Double Round-Robin Tournament

If teams must play each other twice (Home and Away):

```sql
SELECT 
    t1.team_name AS home_team,
    t2.team_name AS away_team,
    CONCAT(t1.team_name, ' (H) vs ', t2.team_name, ' (A)') AS fixture
FROM teams t1
INNER JOIN teams t2 
    ON t1.team_id <> t2.team_id
ORDER BY t1.team_id;
```
