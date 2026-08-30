# How to Remove Duplicates in Excel

## Question
What are the different ways to identify and remove duplicate records in Excel?

---

## 1. Method 1: 'Remove Duplicates' Tool (Ribbon)
1. Select data range &rarr; Go to **Data Tab** &rarr; Click **Remove Duplicates**.
2. Select columns that define unique business records.
3. Permanently deletes duplicate rows.

---

## 2. Method 2: Modern `UNIQUE()` Dynamic Array Formula (Excel 365)
Non-destructive extraction of unique records into a dynamic spill range:
```excel
=UNIQUE(A2:D100)
```

---

## 3. Method 3: Using Power Query
1. Load table into Power Query &rarr; Select unique key columns &rarr; Right-click &rarr; **Remove Duplicates**.
2. Automatically executes whenever the workbook is refreshed.
