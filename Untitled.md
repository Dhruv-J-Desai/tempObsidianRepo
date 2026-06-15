Yes — this is the **right-side filter / summary panel**. You can build it in Power BI using **cards, slicers, and one table/matrix visual**.

## 1. Create “Last Updated” measure

Create a new measure in `cib_tbl_readership`:

```DAX
Last Updated =
"Last Updated  " &
FORMAT(
    MAX(cib_tbl_readership[DataUploadDatetime]),
    "M/D/YYYY h:mm AM/PM"
)
```

Then add a **Card** visual and place this measure in it.

---

## 2. Create “TOTAL HITS” measure

Use your existing total reads/hits measure, or create:

```DAX
Total Hits =
COUNTROWS(cib_tbl_readership)
```

Then create another measure for display:

```DAX
Total Hits Display =
"TOTAL HITS: " & FORMAT([Total Hits], "#,##0")
```

Add this to a **Card** visual.

---

## 3. Read Date Time dropdown filter

Add a **Slicer** visual.

Use:

```text
cib_tbl_readership[ReadDateTime]
```

Then in slicer settings, change it to dropdown.

But to get options like:

```text
Last 3 days
Last 7 days
Last 30 days
```

you need a disconnected table.

Go to **Modeling → New table**:

```DAX
Date Range Filter =
DATATABLE(
    "Range", STRING,
    {
        {"Last 3 days"},
        {"Last 7 days"},
        {"Last 30 days"},
        {"All"}
    }
)
```

Put `Date Range Filter[Range]` in a slicer.

For now, since you already have Last 30 Days measures, you can keep it simple and manually create separate measures later.

---

## 4. Search by Contact Owner

Add a **Slicer** visual using:

```text
cib_tbl_bmcontacts[ContactOwner]
```

Then set slicer style to:

```text
Dropdown
```

or:

```text
Text search enabled
```

In the slicer, click the three dots `...` and turn on **Search**.

Title it:

```text
Search by Contact Owner
```

---

## 5. Search by Account Name

Add another **Slicer** visual using:

```text
cib_tbl_bmcontacts[Account]
```

Turn on **Search**.

Title it:

```text
Search by Account Name
```

---

## 6. Filter by Document Type

Add a **Slicer** visual using:

```text
cib_tbl_dim_doc[DocumentType]
```

Set it as dropdown.

Title:

```text
Filter By Document Type
```

---

## 7. Filter by Channel

Add a slicer using:

```text
cib_tbl_readership[Channel]
```

Set the slicer to list style.

It should show your values:

```text
Bloomberg
Email (Click)
Email (Open)
```

If you want the names to match the screenshot, you can just leave them as your actual channel values.

Title:

```text
Filter by Channel
```

---

## 8. Filter by Region

Add a slicer using whichever region field you want.

Most likely:

```text
cib_tbl_bmcontacts[Region]
```

or:

```text
cib_lookup_countrytoregion[Region]
```

For this screenshot, because it is sales/contact focused, I would use:

```text
cib_tbl_bmcontacts[Region]
```

Title:

```text
Filter by Region
```

---

## 9. Create “Document Types” table

Use a **Matrix visual**, not a normal table, because your screenshot groups document types.

Add these fields:

### Rows

```text
cib_tbl_dim_doc[Contribute_Group]
cib_tbl_dim_doc[DocumentType]
```

### Values

```text
Total Hits
```

Rename the columns visually:

```text
Contribute_Group → Document Type (group)
DocumentType     → Document Type
Total Hits       → Total Count
```

Title the visual:

```text
Document Types
```

---

## 10. Make it look like the screenshot

For the right panel:

1. Insert a rectangle shape.
    
2. Make it light gray or white.
    
3. Place all slicers/cards on top of it.
    
4. Add green title text where needed.
    
5. Use small font sizes.
    

Recommended order:

```text
Last Updated
TOTAL HITS
Read Date Time
Search by Contact Owner
Search by Account Name
Filter By Document Type
Filter by Channel
Filter by Region
Document Types matrix
```

---

## Important relationship check

For these filters to affect the charts/table, these relationships need to be active:

```text
cib_tbl_dim_doc[DocID] → cib_tbl_readership[DocID]
cib_tbl_dim_reader[ReaderID] → cib_tbl_readership[ReaderID]
cib_tbl_dim_reader[ReaderID] → cib_tbl_bmcontacts[ReaderID]
```

For account and contact owner filters, the important path is:

```text
BMContacts → Dim_Reader → Readership
```

If slicers do not affect the charts, then the cross-filter direction may need to be checked. For testing, you can use **Both** between:

```text
cib_tbl_dim_reader and cib_tbl_bmcontacts
```

But normally keep the model clean with single direction unless needed.