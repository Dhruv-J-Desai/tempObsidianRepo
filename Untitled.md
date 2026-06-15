Yes, you can create this as a **second dashboard page** using your existing Power BI semantic model. This view needs more than the first two-table view because it uses **reader, account, document title, and channel counts**.

## Tables needed

For this page, use these tables:

```text
cib_tbl_readership
cib_tbl_dim_doc
cib_tbl_dim_reader
cib_tbl_bmcontacts
```

Relationships needed:

```text
cib_tbl_dim_doc[DocID]       1 → * cib_tbl_readership[DocID]
cib_tbl_dim_reader[ReaderID] 1 → * cib_tbl_readership[ReaderID]
cib_tbl_dim_reader[ReaderID] 1 → * cib_tbl_bmcontacts[ReaderID]
```

The page in your screenshot has three main areas:

```text
1. Top Readers table
2. Top 10 Trade Ideas bar chart
3. Top 10 Market Musing bar chart
```

---

# 1. Create the channel measures

Since your actual channel values are:

```text
Bloomberg
Email (Click)
Email (Open)
```

Create these measures in `cib_tbl_readership`.

### Bloomberg Reads

```DAX
Bloomberg Reads =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

### Email Link Reads

```DAX
Email Link Reads =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[Channel] = "Email (Click)"
)
```

### Email Open Reads

```DAX
Email Open Reads =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

### Total Reads

```DAX
Total Reads =
COUNTROWS(cib_tbl_readership)
```

---

# 2. Create Last 30 Days versions

For this dashboard, use last 30 days based on the **latest date in your sample data**, not today.

### Total Reads Last 30 Days

```DAX
Total Reads Last 30 Days =
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

### Bloomberg Reads Last 30 Days

```DAX
Bloomberg Reads Last 30 Days =
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
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

### Email Link Reads Last 30 Days

```DAX
Email Link Reads Last 30 Days =
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
    cib_tbl_readership[Channel] = "Email (Click)"
)
```

### Email Open Reads Last 30 Days

```DAX
Email Open Reads Last 30 Days =
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
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

---

# 3. Build the “Top Readers” table

Insert a **Table visual** on the left side.

Add these fields:

```text
cib_tbl_bmcontacts[Account]
cib_tbl_dim_reader[ReaderName]
cib_tbl_dim_doc[Title]
Bloomberg Reads Last 30 Days
Email Link Reads Last 30 Days
Email Open Reads Last 30 Days
```

Rename the column headers for this visual:

```text
Account
Reader Name
Title
Bloomberg Library
Email (Link)
Email (Open)
```

Then sort by:

```text
Email Open Reads Last 30 Days descending
```

or by:

```text
Total Reads Last 30 Days descending
```

To make it look like the screenshot, apply a **Top N filter**:

1. Select the table visual.
    
2. Go to **Filters on this visual**.
    
3. Add `ReaderName` or `Title`.
    
4. Change filter type to **Top N**.
    
5. Enter `25` or `50`.
    
6. Use `Total Reads Last 30 Days` as the value.
    
7. Click **Apply filter**.
    

---

# 4. Build “Top 10 Trade Ideas”

Use a **Stacked bar chart**.

Fields:

```text
Y-axis: cib_tbl_dim_doc[Title]
X-axis: Total Reads Last 30 Days
Legend: cib_tbl_readership[Channel]
```

Then filter it to only Trade Ideas documents.

Use whichever field identifies trade ideas in your data. Most likely one of these:

```text
cib_tbl_dim_doc[DocumentType]
cib_tbl_dim_doc[Contribute_Group]
```

For example:

```text
DocumentType = Trade Ideas
```

or:

```text
Contribute_Group = Trade Ideas
```

Then apply **Top N = 10** by `Total Reads Last 30 Days`.

Title:

```text
Top 10 Trade Ideas
```

---

# 5. Build “Top 10 Market Musing”

Copy the Trade Ideas chart and change the filter.

For example:

```text
DocumentType = Market Musing
```

or:

```text
Contribute_Group = Market Musing
```

Apply:

```text
Top N = 10 by Total Reads Last 30 Days
```

Title:

```text
Top 10 Market Musing
```

---

# 6. Add the green dashboard header

At the top:

1. Go to **Insert → Shape → Rectangle**.
    
2. Put it across the top of the page.
    
3. Set fill color to green.
    
4. Add a text box:
    

```text
Research Sales Opportunities
```

5. Add smaller text below:
    

```text
Welcome Sukumar, Uma
```

You can replace the name with your test user name.

---

# 7. Add slicers on the right side

The screenshot has filters on the right. You can add slicers for:

```text
Region
Strategist
Account
DocumentType
Contribute_Group
Date
Channel
```

Good starting slicers:

```text
cib_tbl_dim_doc[DocumentType]
cib_tbl_readership[Channel]
cib_tbl_bmcontacts[Account]
cib_tbl_bmcontacts[Region]
cib_tbl_dim_doc[PublishDate]
```

---

# Most important validation

Before styling, make sure this table works:

```text
Account
ReaderName
Title
Channel
Total Reads
```

If this returns rows correctly, your relationships are working and the dashboard can be built.

For your Jira/Confluence description, you can say:

> Created a second Power BI dashboard page similar to the Research Sales Opportunities view. This page uses the semantic model relationships between readership, document, reader, and contact tables to show top readers, top trade ideas, and top market musing documents by channel-level readership activity.