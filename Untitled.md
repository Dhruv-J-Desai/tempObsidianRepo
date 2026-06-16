For **Power BI Top Documents - Last 30 Days**, create the same three measures as the 7-day version, but change the window to **30 days**.

Use these in `cib_tbl_readership`.

### 1. Email_Open Last 30 Days

```DAX
Email_Open Last 30 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

### 2. Online Last 30 Days

```DAX
Online Last 30 Days =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

### 3. Grand Total Last 30 Days

Use this to match the original logic:

```DAX
Grand Total Last 30 Days =
[Email_Open Last 30 Days] + [Online Last 30 Days]
```

Do **not** include Email Click in this Grand Total if you want it to match the original visual.

---

## Visual setup

Create a **Table visual**.

Add these fields:

```text
cib_tbl_dim_doc[Title]
Email_Open Last 30 Days
Online Last 30 Days
Grand Total Last 30 Days
```

Rename columns for this visual:

```text
Email_Open Last 30 Days → Email_Open
Online Last 30 Days → Online
Grand Total Last 30 Days → Grand_Total
```

Title:

```text
Top Documents - Last 30 Days
```

Sort by:

```text
Grand Total Last 30 Days descending
```

---

## Top N filter

In **Filters on this visual**:

1. Open `Title`.
    
2. Set **Filter type** to:
    

```text
Top N
```

3. Show items:
    

```text
Top 25
```

or whatever count the original shows.

4. By value:
    

```text
Grand Total Last 30 Days
```

5. Click **Apply filter**.
    

---

## Important

If you want the same documents/counts as the original, first validate this against Databricks using the same 30-day logic. Then your Power BI visual should match:

```text
Email_Open = Channel Email (Open)
Online = Channel Bloomberg
Grand_Total = Email_Open + Online
```