Yes — based on your Databricks result, **Power BI is not matching the SQL output**.

Your Databricks SQL result starts with:

```text
Acme Manufacturing
Adage Capital
Addenda Capital
Aimia
...
```

But Power BI starts with:

```text
1832 Asset Management
Adam Donaldson
...
```

So Power BI is likely using a different filter context or a different measure logic.

The most likely issue is this:

> Your Power BI Top Readers matrix is showing rows even when they do not match the same `Last 30 Days + Channel + Grand Total > 0` logic used in Databricks.

## Fix 1: Add a visual filter to remove non-matching rows

Create this measure in Power BI:

```DAX
Reader Activity Total Last 30 Days =
[Online Last 30 Days] + [Email Click Last 30 Days] + [Email Open Last 30 Days]
```

Then select your **Top Readers** matrix and add this measure to:

```text
Filters on this visual
```

Set it to:

```text
is greater than 0
```

This matches your SQL condition:

```sql
WHERE Grand_Total_Last_30_Days > 0
```

## Fix 2: Make sure your Top Readers measures are Last 30 Days measures

For Top Readers, use these measures:

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

```DAX
Email Click Last 30 Days =
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
    cib_tbl_readership[Channel] = "Email (Click)"
)
```

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

Then your matrix should use:

```text
Rows:
AccountProperName
ReaderName
Title

Values:
Online Last 30 Days
Email Click Last 30 Days
Email Open Last 30 Days
```

Do not use all-time measures in this visual.

## Fix 3: Use the same account field as SQL

In SQL you are using:

```sql
a.AccountProperName AS Account_Name
```

So in Power BI, use the same equivalent field:

```text
cib_lookup_accountpropername_clienttype[AccountProperName]
```

Do not use:

```text
cib_tbl_readership[AccountID]
```

or raw account fields if Strategy/SQL is using the lookup proper name.

## Fix 4: Turn off “Show items with no data”

In the Matrix:

1. In the **Rows** bucket, click dropdown for `Account`.
    
2. Make sure **Show items with no data** is unchecked.
    
3. Do the same for:
    
    - `Reader Name`
        
    - `Title`
        

## Final check

Your Power BI Top Readers should match Databricks when all of these are true:

```text
Rows:
AccountProperName
ReaderName
Title

Values:
Online Last 30 Days
Email Click Last 30 Days
Email Open Last 30 Days

Visual filter:
Reader Activity Total Last 30 Days > 0

Sort:
AccountProperName ascending
ReaderName ascending
Title ascending
```

Right now, the first thing I would fix is the visual filter:

```text
Reader Activity Total Last 30 Days > 0
```

That should remove rows like `1832 Asset Management` if they are not present in the Databricks result.