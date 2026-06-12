You do not see data because the measures are probably filtering on:

```DAX
TODAY() - 7
```

But your sample data may not have records in the **actual last 7 days from today**. So Power BI returns blank.

For testing, use the **latest date available in your data** instead of `TODAY()`.

## Fix the DAX measures

Create/replace these measures in `cib_tbl_readership`.

### 1. Total Reads

```DAX
Total Reads =
COUNTROWS(cib_tbl_readership)
```

### 2. Max Read Date in Data

```DAX
Max Read Date =
CALCULATE(
    MAX(cib_tbl_readership[ReadDateTime]),
    ALL(cib_tbl_readership)
)
```

### 3. Reads Last 7 Days based on data

```DAX
Reads Last 7 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxDate
)
```

### 4. Email Reads Last 7 Days

```DAX
Email Reads Last 7 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Email"
)
```

### 5. Online Reads Last 7 Days

```DAX
Online Reads Last 7 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxDate - 7,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Online"
)
```

## Also create Last 30 Days versions

```DAX
Reads Last 30 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxDate
)
```

```DAX
Email Reads Last 30 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Email"
)
```

```DAX
Online Reads Last 30 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= MaxDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Online"
)
```

## Then refresh the visual

For the table visual, use:

```text
cib_tbl_dim_doc[Title]
Email Reads Last 7 Days
Online Reads Last 7 Days
Reads Last 7 Days
```

Do **not** use `Title` from `cib_tbl_readership`. It should come from:

```text
cib_tbl_dim_doc[Title]
```

## Quick check

Before using the Last 7 Days measures, test with only:

```text
Title
Total Reads
```

If that shows data, your relationship is fine and the issue was only the date filter.

If even `Title + Total Reads` is blank, then the issue is likely:

```text
cib_tbl_dim_doc[DocID] values do not match cib_tbl_readership[DocID]
```

In that case, check the `DocID` values in both tables.