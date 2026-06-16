Yes — for **Last 30 Days from the present/current date**, use `TODAY()` in Power BI instead of using `MaxDate` from the data.

This will answer:

> Do we have any readership records in the last 30 days from today?

## Power BI measures using present date

### Email_Open Last 30 Days from Today

```DAX
Email_Open Last 30 Days Today =
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30,
    cib_tbl_readership[ReadDateTime] <= TODAY(),
    cib_tbl_readership[Channel] = "Email (Open)"
)
```

### Online Last 30 Days from Today

```DAX
Online Last 30 Days Today =
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= TODAY() - 30,
    cib_tbl_readership[ReadDateTime] <= TODAY(),
    cib_tbl_readership[Channel] = "Bloomberg"
)
```

### Grand Total Last 30 Days from Today

```DAX
Grand Total Last 30 Days Today =
[Email_Open Last 30 Days Today] + [Online Last 30 Days Today]
```

Use these in the table visual:

```text
Title
Email_Open Last 30 Days Today
Online Last 30 Days Today
Grand Total Last 30 Days Today
```

If the table becomes blank or shows 0, that means your sample data does **not** have records in the last 30 calendar days from today.

## Databricks SQL check

Run this first to see your date range:

```sql
SELECT
  MIN(ReadDateTime) AS min_read_datetime,
  MAX(ReadDateTime) AS max_read_datetime,
  COUNT(*) AS total_rows
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership;
```

Then check last 30 days from current date:

```sql
SELECT
  COUNT(*) AS rows_last_30_days_from_today
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership
WHERE ReadDateTime >= current_date() - INTERVAL 30 DAYS
  AND ReadDateTime <= current_date();
```

And for Top Documents:

```sql
SELECT
    d.Title,
    SUM(CASE WHEN r.Channel = 'Email (Open)' THEN 1 ELSE 0 END) AS Email_Open,
    SUM(CASE WHEN r.Channel = 'Bloomberg' THEN 1 ELSE 0 END) AS Online,
    SUM(CASE WHEN r.Channel IN ('Email (Open)', 'Bloomberg') THEN 1 ELSE 0 END) AS Grand_Total
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_dim_doc d
    ON r.DocID = d.DocID
WHERE r.ReadDateTime >= current_date() - INTERVAL 30 DAYS
  AND r.ReadDateTime <= current_date()
GROUP BY d.Title
HAVING Grand_Total > 0
ORDER BY Grand_Total DESC, Email_Open DESC, Online DESC;
```

If this returns no rows, use the **MaxDate-based** logic for testing/demo data. That means the data is historical and not within the real current 30-day window.