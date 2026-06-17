Got it — then your Power BI logic should match Strategy and use **`Max(ReadDateTime)` as the reference date**, not today’s date.

You can pass the message like this:

Hi Nilanka,

I will keep the Power BI Last 7 / Last 30 Days logic based on `Max(ReadDateTime)`, similar to how it is handled in Strategy.

So the Power BI calculation will use:

Last 7 Days = Max(ReadDateTime) - 7 days  
Last 30 Days = Max(ReadDateTime) - 30 days

This should keep the Power BI logic consistent with the Strategy Free-form SQL approach.

And update your DAX like this.

### Last 30 Days using Max(ReadDateTime)

```DAX
Total Reads Last 30 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate
)
```

### Last 7 Days using Max(ReadDateTime)

```DAX
Total Reads Last 7 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate
)
```

For each channel-specific measure, use the same `MaxReadDate` pattern.