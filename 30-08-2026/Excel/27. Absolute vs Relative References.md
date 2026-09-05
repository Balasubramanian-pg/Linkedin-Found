# Absolute, Relative, and Mixed Cell References in Excel

## Question
Explain the difference between Relative, Absolute, and Mixed cell references in Excel using the `$` symbol.

---

## 1. Reference Types Summary

| Reference Type | Example | Behavior when copied across rows & columns |
| :--- | :--- | :--- |
| **Relative** | `A1` | Both column and row change relative to destination. |
| **Absolute** | `$A$1` | Column and row are **locked**; never changes. |
| **Mixed (Row Lock)** | `A$1` | Column changes, but row 1 stays locked. |
| **Mixed (Column Lock)**| `$A1` | Column A stays locked, but row increments. |

---

## 2. Practical Use Case: Tax Calculation Matrix

```excel
-- Formula to calculate Sales Tax using a single tax rate cell ($B$1):
=A4 * $B$1
```
When copied down from row 4 to row 100, `A4` changes to `A5`, `A6`... while `$B$1` remains firmly locked on the tax rate.
