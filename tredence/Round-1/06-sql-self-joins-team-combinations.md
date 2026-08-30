# SQL: Self-Joins and Generating Combinations vs Permutations

## Question
Explain how SQL Self-Joins work and demonstrate how to generate unique pair combinations (N choose 2) versus directed permutations.

---

## 1. Combinations (A < B) vs Permutations (A <> B)

- **Combinations (Order Does Not Matter):** Pair `(India, Australia)` is identical to `(Australia, India)`. Use `t1.id < t2.id`.
- **Permutations (Order Matters / Directed):** Used for directed flights or sender-receiver networks. Use `t1.id <> t2.id`.

---

## 2. Generating Unique Pairs (Combinations)

```sql
-- Generating distinct pairs of employees in the same department for mentorship
SELECT 
    e1.dept_id,
    e1.emp_name AS senior_emp,
    e2.emp_name AS junior_emp
FROM employees e1
INNER JOIN employees e2 
    ON e1.dept_id = e2.dept_id 
   AND e1.emp_id < e2.emp_id  -- Prevents self-pairing and mirror duplicates
ORDER BY e1.dept_id, e1.emp_name;
```

---

## 3. Finding Employees Earning More Than Their Manager (Classic Self-Join)

```sql
SELECT 
    e.emp_name AS employee,
    e.salary AS emp_salary,
    m.emp_name AS manager,
    m.salary AS manager_salary
FROM employees e
INNER JOIN employees m 
    ON e.manager_id = m.emp_id
WHERE 
    e.salary > m.salary;
```
