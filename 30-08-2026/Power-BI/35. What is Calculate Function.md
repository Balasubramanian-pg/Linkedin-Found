# The `CALCULATE()` Function in DAX

## Question
What is the `CALCULATE()` function in DAX, why is it the most powerful function in Power BI, and how does it perform **Context Transition**?

---

## 1. What is `CALCULATE()`?

`CALCULATE()` evaluates a DAX expression in a **modified filter context**. It is the only function in DAX capable of overriding, removing, or adding filters to a calculation.

### Syntax:
```dax
CALCULATE(<expression>, <filter1>, <filter2>, ...)
```

---

## 2. Practical DAX Examples

```dax
-- Total Sales for only the 'Electronics' category
Sales_Electronics = 
CALCULATE(
    SUM(Sales[Amount]),
    Dim_Product[Category] = "Electronics"
)

-- Percentage of All Sales (Removing Filter Context using ALL)
Sales_Pct_Of_All = 
DIVIDE(
    SUM(Sales[Amount]),
    CALCULATE(SUM(Sales[Amount]), ALL(Dim_Product))
)
```

---

## 3. What is Context Transition?
When a measure is called inside a row context (like `SUMX` or a Calculated Column), `CALCULATE()` automatically converts the current row's values into an equivalent **filter context**.
