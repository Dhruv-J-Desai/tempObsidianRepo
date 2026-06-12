Yes — start this dashboard page using only these **two tables**:

```text
cib_tbl_dim_doc
cib_tbl_readership
```

Relationship needed:

```text
cib_tbl_dim_doc[DocID]  1 → *  cib_tbl_readership[DocID]
```

## Goal

Create two table visuals:

```text
Top Documents - Last 7 Days
Top Documents - Last 30 Days
```

Each table should show:

```text
Title | Email | Online | Grand Total
```

---

## Step 1: Confirm the relationship

In **Model view**, make sure this exists:

```text
cib_tbl_dim_doc[DocID] → cib_tbl_readership[DocID]
```

Cardinality:

```text
One to many
```

Cross-filter direction:

```text
Single
```

---

## Step 2: Create the measures

Go to **Report view** or **Model view** → select `cib_tbl_readership` → **New measure**.

Create these measures:

### Total Reads

```DAX
Total Reads =
COUNTROWS(cib_tbl_readership)
```

### Email Reads

```DAX
Email Reads =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[Channel] = "Email"
)
```

### Online Reads

```DAX
Online Reads =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[Channel] = "Online"
)
```

### Reads Last 7 Days

```DAX
Reads Last 7 Days =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 7
)
```

### Email Reads Last 7 Days

```DAX
Email Reads Last 7 Days =
CALCULATE(
    [Email Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 7
)
```

### Online Reads Last 7 Days

```DAX
Online Reads Last 7 Days =
CALCULATE(
    [Online Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 7
)
```

### Reads Last 30 Days

```DAX
Reads Last 30 Days =
CALCULATE(
    [Total Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30
)
```

### Email Reads Last 30 Days

```DAX
Email Reads Last 30 Days =
CALCULATE(
    [Email Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30
)
```

### Online Reads Last 30 Days

```DAX
Online Reads Last 30 Days =
CALCULATE(
    [Online Reads],
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30
)
```

---

## Step 3: Build “Top Documents - Last 7 Days”

Insert a **Table visual**.

Add these fields:

```text
cib_tbl_dim_doc[Title]
Email Reads Last 7 Days
Online Reads Last 7 Days
Reads Last 7 Days
```

Rename the columns in the visual:

```text
Email Reads Last 7 Days  → Email
Online Reads Last 7 Days → Online
Reads Last 7 Days        → Grand Total
```

Then sort by:

```text
Grand Total descending
```

To make it top documents only:

1. Select the table visual.
    
2. Go to **Filters on this visual**.
    
3. Add `Title`.
    
4. Change filter type to **Top N**.
    
5. Enter something like `25`.
    
6. Use **Reads Last 7 Days** as the “By value”.
    
7. Apply filter.
    

Title the visual:

```text
Top Documents - Last 7 Days
```

---

## Step 4: Build “Top Documents - Last 30 Days”

Copy the first table visual and paste it to the right.

Replace the measures with:

```text
Email Reads Last 30 Days
Online Reads Last 30 Days
Reads Last 30 Days
```

Rename them visually as:

```text
Email
Online
Grand Total
```

Sort by:

```text
Grand Total descending
```

Apply Top N using:

```text
Reads Last 30 Days
```

Title the visual:

```text
Top Documents - Last 30 Days
```

---

## Step 5: Add the header image / title area

For the top-left TD Strategy Weekly Update style section:

In Power BI Desktop:

```text
Insert → Image
```

Add the TD image/logo if you have it.

Then add a text box:

```text
TD Strategy Weekly Update
```

Use:

```text
Insert → Text box
```

For a quick version, create a simple title:

```text
TD Strategy Weekly Update
```

Then place the two table visuals below it.

---

## Step 6: Validate the numbers

Before styling, validate with a simple table:

```text
Title
Channel
Total Reads
```

Make sure Email and Online counts are splitting correctly.

Also check actual channel values in Power BI. Sometimes the values may be:

```text
email
Email
EMAIL
Online
Web
```

If the channel values are not exactly `Email` and `Online`, adjust the DAX.

For example, safer version:

```DAX
Email Reads =
CALCULATE(
    [Total Reads],
    LOWER(cib_tbl_readership[Channel]) = "email"
)
```

```DAX
Online Reads =
CALCULATE(
    [Total Reads],
    LOWER(cib_tbl_readership[Channel]) = "online"
)
```

---

## Description you can use in Jira/Confluence

> Created an initial Power BI dashboard page using `cib_tbl_dim_doc` and `cib_tbl_readership`. The page recreates the first Strategy-style view with top documents for the last 7 days and last 30 days. The visuals use `DocID` to join document metadata with readership activity and calculate Email, Online, and Grand Total readership counts by document title.