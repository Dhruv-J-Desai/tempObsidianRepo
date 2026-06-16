The **Channel slicer is not affecting Top Readers** because your Top Readers table is using **separate hardcoded channel measures** like:

```DAX
Online Last 30 Days =
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

and:

```DAX
Email Open Last 30 Days =
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

Those measures force their own channel filter, so the slicer selection can get ignored/overridden.

## Fix option 1: Use `KEEPFILTERS`

Update your channel measures like this.

### Online Last 30 Days

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
    KEEPFILTERS(cib_tbl_readership[Channel] = "Bloomberg")
)
```

### Email Click Last 30 Days

```DAX
Email Click Last 30 Days =
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
    KEEPFILTERS(cib_tbl_readership[Channel] = "Email (Click)")
)
```

### Email Open Last 30 Days

```DAX
Email Open Last 30 Days =
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
    KEEPFILTERS(cib_tbl_readership[Channel] = "Email (Open)")
)
```

`KEEPFILTERS` makes the measure respect the slicer instead of overriding it.

---

## Fix option 2: Check Edit interactions

Also confirm the slicer is allowed to filter Top Readers:

1. Click the **Filter by Channel** slicer.
    
2. Go to top ribbon **Format**.
    
3. Click **Edit interactions**.
    
4. On the **Top Readers** visual, make sure the **filter icon** is selected.
    
5. Turn off **Edit interactions**.
    

---

## Add a visual filter to hide blank rows

After using `KEEPFILTERS`, some rows may still appear blank. Create this measure:

```DAX
Reader Activity Total Last 30 Days =
[Online Last 30 Days]
+ [Email Click Last 30 Days]
+ [Email Open Last 30 Days]
```

Then add it to **Filters on this visual** for Top Readers:

```text
Reader Activity Total Last 30 Days is greater than 0
```

That will remove rows that do not match the selected channel.

---

So the main issue is: **your slicer is fine, but the Top Readers measures are hardcoded by channel. Add `KEEPFILTERS` so the slicer selection is respected.**