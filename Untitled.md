Yes. Replace `MaxDate` logic with the **current date** using `TODAY()`.

Use this for **Total Reads Last 30 Days based on current date**:

```DAX
Total Reads Last 30 Days =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30,
    cib_tbl_readership[ReadDateTime] < TODAY() + 1
)
```

Use `< TODAY() + 1` instead of `<= TODAY()` because `ReadDateTime` includes time. This includes all records from today as well.

### For your channel-specific measures

**Online Last 30 Days**

```DAX
Online Last 30 Days =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30,
    cib_tbl_readership[ReadDateTime] < TODAY() + 1,
    KEEPFILTERS(cib_tbl_readership[Channel] = "Bloomberg")
)
```

**Email Open Last 30 Days**

```DAX
Email Open Last 30 Days =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30,
    cib_tbl_readership[ReadDateTime] < TODAY() + 1,
    KEEPFILTERS(cib_tbl_readership[Channel] = "Email (Open)")
)
```

**Email Click Last 30 Days**

```DAX
Email Click Last 30 Days =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30,
    cib_tbl_readership[ReadDateTime] < TODAY() + 1,
    KEEPFILTERS(cib_tbl_readership[Channel] = "Email (Click)")
)
```

**Grand Total Last 30 Days**

```DAX
Grand Total Last 30 Days =
[Online Last 30 Days]
+ [Email Open Last 30 Days]
+ [Email Click Last 30 Days]
```

Or, if matching the original Top Documents logic where click is excluded:

```DAX
Grand Total Last 30 Days =
[Online Last 30 Days]
+ [Email Open Last 30 Days]
```

One important note: if your sample data is historical and not within the actual last 30 days from today, these measures will return blank/0. In that case, keep using the `MaxDate`-based version for demo/testing.