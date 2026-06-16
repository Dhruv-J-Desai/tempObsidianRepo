That dropdown is not changing anything because your **Range slicer is from a disconnected table**. It will only affect visuals if the measures are written to read the selected range.

Right now, your visuals are probably still using fixed measures like:

```DAX
Email_Open Last 30 Days
Online Last 30 Days
Grand Total Last 30 Days
```

Those measures do **not** care what you select in the Range slicer.

## Fix: create dynamic measures

### 1. Selected Days measure

Create this measure:

```DAX
Selected Range Days =
SWITCH(
    SELECTEDVALUE('Date Range Filter'[Range], "All"),
    "Last 3 days", 3,
    "Last 7 days", 7,
    "Last 30 days", 30,
    "All", 99999,
    99999
)
```

---

### 2. Dynamic Email Open

```DAX
Email_Open Dynamic =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
VAR DaysBack = [Selected Range Days]
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxDate - DaysBack,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

---

### 3. Dynamic Online

```DAX
Online Dynamic =
VAR MaxDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
VAR DaysBack = [Selected Range Days]
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxDate - DaysBack,
    cib_tbl_readership[ReadDateTime] <= MaxDate,
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

---

### 4. Dynamic Grand Total

```DAX
Grand Total Dynamic =
[Email_Open Dynamic] + [Online Dynamic]
```

---

### 5. Dynamic Total Hits Display

```DAX
Total Hits Dynamic Display =
"TOTAL HITS: " & FORMAT([Grand Total Dynamic], "#,##0")
```

Use this in the top card instead of your old `Total Hits Display`.

---

## Then update the visuals

For the Top Documents table, remove the fixed measures and use:

```text
Title
Email_Open Dynamic
Online Dynamic
Grand Total Dynamic
```

Rename them for the visual as:

```text
Email_Open
Online
Grand_Total
```

For the card, use:

```text
Total Hits Dynamic Display
```

## Also change slicer setting

Your Range slicer has multiple checkboxes. Better set it to single select:

1. Select the Range slicer.
    
2. Format visual.
    
3. Slicer settings → Selection.
    
4. Turn **Single select = On**.
    
5. Turn **Show “Select all” = Off**.
    

Then selecting `Last 7 days` or `Last 30 days` should actually change the table and card.

The key point: the slicer is working, but your current measures are not listening to it.