This SQL result explains the difference clearly.

For this document:

```text
US Leveraged Finance: Covenant Trends
```

Databricks shows:

```text
Bloomberg      = 5
Email (Click)  = 2
Email (Open)   = 9
```

So the original visual numbers:

```text
Email_Open = 9
Online     = 5
Grand_Total = 14
```

are calculated as:

```text
Email_Open = Email (Open)
Online = Bloomberg
Grand_Total = Email (Open) + Bloomberg
```

It is **not including Email Click** in the `Grand_Total`.

That is why your Power BI version becomes different if your `Grand Total Last 7 Days` includes all channels:

```text
9 + 5 + 2 = 16
```

But the original is:

```text
9 + 5 = 14
```

### So for Top Document - Last 7 Days, use these definitions

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

```DAX
Grand Total Last 7 Days =
[Email_Open Last 7 Days] + [Online Last 7 Days]
```

### Do not include this in Grand Total

```text
Email (Click)
```

For this specific visual, `Email (Click)` exists in the data, but it is not part of the original `Grand_Total` logic.

You can explain it like this:

> I checked the data for one document in Databricks. The original visual is using `Email (Open)` as Email_Open and `Bloomberg` as Online. Grand_Total appears to be calculated as Email_Open + Online only. Email Click exists in the data but is not included in this specific Top Documents total.