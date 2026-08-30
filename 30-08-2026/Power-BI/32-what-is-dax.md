# Understanding DAX (Data Analysis Expressions) in Power BI

## Question
What is DAX in Power BI, and what are its core calculation paradigms?

---

## 1. What is DAX?

**DAX (Data Analysis Expressions)** is the native formula and query language used in Power BI, Analysis Services (SSAS), and Power Pivot in Excel. It is designed specifically for dimensional modeling and OLAP calculations.

---

## 2. Core Building Blocks of DAX
1. **Scalar Functions:** Return a single scalar value (`SUM`, `AVERAGE`, `COUNTROWS`, `MAX`).
2. **Table Functions:** Return an entire table to be evaluated within another formula (`FILTER`, `ALL`, `VALUES`, `SUMMARIZE`).
3. **Time Intelligence Functions:** Simplify calendar comparisons (`YTD`, `SAMEPERIODLASTYEAR`, `DATEADD`).
4. **Context Modifiers:** Dynamically alter filter propagation (`CALCULATE`, `ALLSELECTED`, `USERELATIONSHIP`).
