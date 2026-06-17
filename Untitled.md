Yes — use separate measures for **Email Open** and **Online/Bloomberg**, both based on `Max(ReadDateTime)`.

### Email Open - Last 7 Days

```DAX
Email Open Last 7 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate,
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

### Online / Bloomberg - Last 7 Days

```DAX
Online Last 7 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate,
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

### Grand Total - Last 7 Days

To match the Strategy visual:

```DAX
Grand Total Last 7 Days =
[Email Open Last 7 Days] + [Online Last 7 Days]
```

---

### Email Open - Last 30 Days

```DAX
Email Open Last 30 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate,
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

### Online / Bloomberg - Last 30 Days

```DAX
Online Last 30 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate,
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

### Grand Total - Last 30 Days

```DAX
Grand Total Last 30 Days =
[Email Open Last 30 Days] + [Online Last 30 Days]
```

For the message, you can say:

Hi Nilanka,

I will update the Power BI measures for the Strategy-matching logic using `Max(ReadDateTime)` as the reference date.

Specifically:

Email Open Last 7 / 30 Days = count of rows where `Channel = Email (Open)` within `Max(ReadDateTime) - 7 / 30 days`

Online Last 7 / 30 Days = count of rows where `Channel = Bloomberg` within `Max(ReadDateTime) - 7 / 30 days`

Grand Total = Email Open + Online

This should match the way Strategy is calculating the Last 7 Days and Last 30 Days visuals from the custom Free-form SQL table.