Yes, good point. Ignoring **Email Click**, the **Email Open** and **Online** numbers can still be different for a few reasons.

The most likely reason is this:

> Your Power BI visual is counting rows from `cib_tbl_readership`, but the original visual may be using pre-aggregated metric columns or a different date/window/filter logic.

## 1. Your Power BI logic is row-count based

Your measure is probably doing this:

```DAX
COUNTROWS(cib_tbl_readership)
```

So for each document:

```text
Email_Open = count of rows where Channel = Email (Open)
Online = count of rows where Channel = Bloomberg
```

But the original dashboard may be using something like:

```text
Email_Open metric column
Online metric column
Grand_Total metric column
```

Those may already be aggregated differently.

---

## 2. Date range may not be the same

Your Power BI “Last 7 Days” is based on:

```text
Max ReadDateTime in your dataset - 7 days
```

The original may be using:

```text
Report run date - 7 calendar days
Last updated date - 7 days
Publish date
Read date
Business date
```

Even a one-day difference can change counts.

To check, create a temporary card/table with:

```DAX
Max Read Date =
MAX(cib_tbl_readership[ReadDateTime])
```

Then confirm whether that matches the original report’s last updated date/window.

---

## 3. Online may not equal Bloomberg exactly

In your dataset, we mapped:

```text
Online = Bloomberg
```

But in the original dashboard, **Online** may mean a different source, for example:

```text
Online = Bloomberg + Library
Online = website reads
Online = non-email reads
Online = channel other than email
```

So if original `Online` is not exactly `Channel = Bloomberg`, numbers will differ.

---

## 4. The visual may be grouped differently

Your Power BI visual groups by:

```text
Title
```

But the original may group by:

```text
DocID + Title
```

This matters because the same title can appear for multiple documents. If Power BI groups only by `Title`, it may combine or split differently than the original.

For closer matching, add `DocID` temporarily:

```text
DocID
Title
Email_Open Last 7 Days
Online Last 7 Days
Grand Total Last 7 Days
```

Then check if counts make more sense.

---

## 5. Your sample data may not match the original data

From your screenshot, Power BI totals are:

```text
Email_Open = 20
Online = 23
```

But the original visual has much higher row-level counts like:

```text
Email_Open = 9
Online = 5
```

That suggests your current Databricks sample data may not be the same as the original source data. The model structure can be correct, but the numbers will not match if the loaded data is different.

---

## Best way to prove the difference

Run this in Databricks for one document from the Power BI visual:

```sql
SELECT
    d.DocID,
    d.Title,
    r.Channel,
    COUNT(*) AS cnt
FROM cib_tbl_readership r
JOIN cib_tbl_dim_doc d
    ON r.DocID = d.DocID
WHERE d.Title = 'US Leveraged Finance: Covenant Trends'
GROUP BY
    d.DocID,
    d.Title,
    r.Channel
ORDER BY
    r.Channel;
```

If Databricks returns the same counts as Power BI, then Power BI is correct based on your current data.

Then the reason it differs from the original dashboard is one of these:

```text
Different source metric
Different date filter
Different channel definition
Different grouping grain
Different sample data
```

## Simple explanation you can give

> The Email Open and Online numbers are different because our Power BI version is currently counting readership rows by channel from `cib_tbl_readership`. The original dashboard may be using different metric definitions, a different last-7-days date window, or pre-aggregated fields. To match it exactly, we need to confirm whether Email Open and Online in the source dashboard are calculated from `Channel`, from separate metric columns, or from another aggregation table.