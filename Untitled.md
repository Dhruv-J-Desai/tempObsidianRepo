Yes, if you **do not want Contact Owner** and you are using a **simple Table**, then the correct grain should be:

```text
Account + Reader Name + Title
```

Right now it is repeating because **Account is not correctly tied to the actual readership event**. It is showing the same reader/title against many accounts.

## What you should use

For Top Readers table, use account based on the `AccountID` in `cib_tbl_readership`.

Your relationship should be:

```text
cib_lookup_accountpropername_clienttype[AccountID]
1 → *
cib_tbl_readership[AccountID]
```

Then in the table, use:

```text
Account:     cib_lookup_accountpropername_clienttype[AccountProperName]
Reader Name: cib_tbl_dim_reader[ReaderName]
Title:       cib_tbl_dim_doc[Title]
Measures:    Online / Email Click / Email Open / Grand Total
```

Do **not** use `cib_tbl_bmcontacts[Account]` for this version if you do not want Contact Owner, because `BMContacts` can have multiple account mappings for the same reader.

---

## Table visual setup

Use a normal **Table** visual with:

```text
cib_lookup_accountpropername_clienttype[AccountProperName]
cib_tbl_dim_reader[ReaderName]
cib_tbl_dim_doc[Title]
Online Last 30 Days
Email Click Last 30 Days
Email Open Last 30 Days
Grand Total Last 30 Days
```

Rename `AccountProperName` to:

```text
Account
```

---

## Add a filter to remove fake/repeated rows

Create this measure:

```DAX
Reader Activity Total Last 30 Days =
[Online Last 30 Days]
+ [Email Click Last 30 Days]
+ [Email Open Last 30 Days]
```

Then select the Top Readers table and add this to **Filters on this visual**:

```text
Reader Activity Total Last 30 Days is greater than 0
```

This is important. It removes account/reader/title combinations that do not actually have readership activity.

---

## Why your current table repeats

This row:

```text
Elena Volkov | Chart Logic: Equity Risk Premium vs Credit
```

is being shown for many accounts because Power BI is not using the account from the same event grain as `cib_tbl_readership`.

The actual event grain is closer to:

```text
Readership row = ReaderID + DocID + AccountID + Channel + ReadDateTime
```

So the Account should come through `Readership.AccountID`, not from a loose contact/account mapping.

---

## Quick check

Go to **Model view** and confirm these active relationships:

```text
cib_tbl_dim_reader[ReaderID] 1 → * cib_tbl_readership[ReaderID]

cib_tbl_dim_doc[DocID] 1 → * cib_tbl_readership[DocID]

cib_lookup_accountpropername_clienttype[AccountID] 1 → * cib_tbl_readership[AccountID]
```

Once those are active, the simple table should stop repeating the same reader/title across unrelated accounts.