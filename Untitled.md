Yes — for this layout, you should use a **Matrix visual**, not a normal table.

The original is grouped like this:

```text
Contact Owner
  → Account
     → Reader Name
        → Title
```

That is why the same Contact Owner / Account is shown once and then the titles are listed underneath. A normal table will repeat every value on every row.

## Build it like the original

### 1. Replace the current Top Readers table with Matrix

Select your current Top Readers visual, then click **Matrix** in Visualizations.

### 2. Put these fields in Rows, in this exact order

```text
Rows:
cib_tbl_bmcontacts[ContactOwner]
cib_tbl_bmcontacts[Account]
cib_tbl_dim_reader[ReaderName]
cib_tbl_dim_doc[Title]
```

Important: use `Account` from:

```text
cib_tbl_bmcontacts[Account]
```

not from the lookup account table.

### 3. Put these metrics in Values

```text
Values:
Bloomberg Library
Email Link
Email Open
```

or your current measures:

```text
Bloomberg Reads Last 30 Days
Email Link Reads Last 30 Days
Email Open Reads Last 30 Days
```

Rename them for the visual:

```text
Bloomberg Reads Last 30 Days → Bloomberg Library
Email Link Reads Last 30 Days → Email (Link)
Email Open Reads Last 30 Days → Email (Open)
```

---

## Make it display like the screenshot

### Turn off stepped layout

This is the key setting.

1. Select the Matrix.
    
2. Go to **Format visual**.
    
3. Open **Row headers**.
    
4. Turn **Stepped layout** to:
    

```text
Off
```

Now it will show separate columns:

```text
Contact Owner | Account | Reader Name | Title
```

instead of one indented hierarchy column.

### Expand all hierarchy levels

On the matrix visual, use the drill/expand icons at the top of the visual.

Click:

```text
Expand all down one level in the hierarchy
```

Keep clicking until it expands to:

```text
Contact Owner → Account → Reader Name → Title
```

This will make it look like the original screenshot.

---

## Fix repeating issue

If you still see the same reader/title repeated across many accounts, check this:

Use these fields only:

```text
ContactOwner from cib_tbl_bmcontacts
Account from cib_tbl_bmcontacts
ReaderName from cib_tbl_dim_reader
Title from cib_tbl_dim_doc
```

Do not use:

```text
cib_lookup_accountpropername_clienttype[AccountProperName]
```

inside this visual.

---

## Optional: hide empty rows

In the Matrix filters, add your combined ranking measure, for example:

```text
Reader Activity Total
```

Set filter:

```text
is greater than 0
```

If you do not have it, create:

```DAX
Reader Activity Total =
[Bloomberg Reads Last 30 Days]
+ [Email Link Reads Last 30 Days]
+ [Email Open Reads Last 30 Days]
```

Then apply visual filter:

```text
Reader Activity Total > 0
```

That will remove rows where there was no activity.

Final structure should be:

```text
Matrix

Rows:
ContactOwner
Account
ReaderName
Title

Values:
Bloomberg Library
Email (Link)
Email (Open)
```

This will give you the grouped “Top Readers” layout like the original.