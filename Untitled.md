Yes, exactly.

For **Top Document - Last 7 Days**, each row should represent **one document**, and the columns should be **column-level totals for that document**.

So for every `Title`, you should calculate:

```text
Email_Open count for that document
Online count for that document
Grand_Total for that document
```

Example:

```text
Title                                      Email_Open   Online   Grand_Total
US Leveraged Finance: Covenant Trends     9            5        14
```

That means:

```text
For this specific document:
Email_Open = 9
Online = 5
Grand_Total = 9 + 5 = 14
```

So yes — the total is **per document row**, not just one overall total at the bottom.

---

## Important correction for your Power BI version

Right now, if your visual shows many rows with `1`, it means the measure is counting each readership row individually, but the grouping/ranking may not be matching the original.

For the first visual, use these measures:

### Email_Open per document

```DAX
Email_Open Last 7 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

### Online per document

```DAX
Online Last 7 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

### Grand Total per document

Use this if original dashboard total is only Email_Open + Online:

```DAX
Grand Total Last 7 Days =
[Email_Open Last 7 Days] + [Online Last 7 Days]
```

This is probably the better match to your screenshot because:

```text
9 + 5 = 14
8 + 2 = 10
```

---

## Visual should be

```text
Rows/Columns:
Title from cib_tbl_dim_doc
Email_Open Last 7 Days
Online Last 7 Days
Grand Total Last 7 Days
```

Sort by:

```text
Grand Total Last 7 Days descending
```

Filter:

```text
Title Top N by Grand Total Last 7 Days
```

So yes, your understanding is correct:

> We need document-level totals, where each measure is calculated per document title using the `DocID` relationship between `cib_tbl_dim_doc` and `cib_tbl_readership`.