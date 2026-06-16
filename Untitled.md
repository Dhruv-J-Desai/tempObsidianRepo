Yes — first validate in **Databricks SQL** before adjusting Power BI. Use this query to reproduce the **Top Documents - Last 7 Days** logic.

Based on what we confirmed, the original visual seems to calculate:

```text
Email_Open = Channel = 'Email (Open)'
Online = Channel = 'Bloomberg'
Grand_Total = Email_Open + Online
```

It does **not** include `Email (Click)` in `Grand_Total`.

## 1. Databricks query without date filter first

Run this first because it should match the numbers you saw for:

```text
US Leveraged Finance: Covenant Trends
Email_Open = 9
Online = 5
Grand_Total = 14
```

```sql
SELECT
    d.Title,

    SUM(CASE WHEN r.Channel = 'Email (Open)' THEN 1 ELSE 0 END) AS Email_Open,

    SUM(CASE WHEN r.Channel = 'Bloomberg' THEN 1 ELSE 0 END) AS Online,

    SUM(CASE 
            WHEN r.Channel IN ('Email (Open)', 'Bloomberg') 
            THEN 1 
            ELSE 0 
        END) AS Grand_Total

FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_dim_doc d
    ON r.DocID = d.DocID

GROUP BY
    d.Title

HAVING Grand_Total > 0

ORDER BY
    Grand_Total DESC,
    Email_Open DESC,
    Online DESC;
```

This should produce the same style as:

```text
Title | Email_Open | Online | Grand_Total
```

---

## 2. Check one document specifically

Use this to confirm the exact values for the document from the screenshot:

```sql
SELECT
    d.DocID,
    d.Title,

    SUM(CASE WHEN r.Channel = 'Email (Open)' THEN 1 ELSE 0 END) AS Email_Open,

    SUM(CASE WHEN r.Channel = 'Bloomberg' THEN 1 ELSE 0 END) AS Online,

    SUM(CASE 
            WHEN r.Channel IN ('Email (Open)', 'Bloomberg') 
            THEN 1 
            ELSE 0 
        END) AS Grand_Total,

    SUM(CASE WHEN r.Channel = 'Email (Click)' THEN 1 ELSE 0 END) AS Email_Click

FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_dim_doc d
    ON r.DocID = d.DocID

WHERE d.Title = 'US Leveraged Finance: Covenant Trends'

GROUP BY
    d.DocID,
    d.Title;
```

Expected based on your previous SQL result:

```text
Email_Open = 9
Online = 5
Grand_Total = 14
Email_Click = 2
```

---

## 3. Now test real “Last 7 Days” using max date in data

After the above matches, then apply the date window:

```sql
WITH max_date AS (
    SELECT MAX(ReadDateTime) AS max_read_datetime
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership
),

filtered AS (
    SELECT
        r.*
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership r
    CROSS JOIN max_date m
    WHERE r.ReadDateTime >= m.max_read_datetime - INTERVAL 7 DAYS
      AND r.ReadDateTime <= m.max_read_datetime
)

SELECT
    d.Title,

    SUM(CASE WHEN r.Channel = 'Email (Open)' THEN 1 ELSE 0 END) AS Email_Open,

    SUM(CASE WHEN r.Channel = 'Bloomberg' THEN 1 ELSE 0 END) AS Online,

    SUM(CASE 
            WHEN r.Channel IN ('Email (Open)', 'Bloomberg') 
            THEN 1 
            ELSE 0 
        END) AS Grand_Total

FROM filtered r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_dim_doc d
    ON r.DocID = d.DocID

GROUP BY
    d.Title

HAVING Grand_Total > 0

ORDER BY
    Grand_Total DESC,
    Email_Open DESC,
    Online DESC;
```

---

## 4. Last 30 Days version

```sql
WITH max_date AS (
    SELECT MAX(ReadDateTime) AS max_read_datetime
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership
),

filtered AS (
    SELECT
        r.*
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership r
    CROSS JOIN max_date m
    WHERE r.ReadDateTime >= m.max_read_datetime - INTERVAL 30 DAYS
      AND r.ReadDateTime <= m.max_read_datetime
)

SELECT
    d.Title,

    SUM(CASE WHEN r.Channel = 'Email (Open)' THEN 1 ELSE 0 END) AS Email_Open,

    SUM(CASE WHEN r.Channel = 'Bloomberg' THEN 1 ELSE 0 END) AS Online,

    SUM(CASE 
            WHEN r.Channel IN ('Email (Open)', 'Bloomberg') 
            THEN 1 
            ELSE 0 
        END) AS Grand_Total

FROM filtered r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_dim_doc d
    ON r.DocID = d.DocID

GROUP BY
    d.Title

HAVING Grand_Total > 0

ORDER BY
    Grand_Total DESC,
    Email_Open DESC,
    Online DESC;
```

## What to compare

Compare Databricks output against the original dashboard first.

If **query 1** matches the original, but **query 3** does not, then the original “Last 7 Days” is probably not using `ReadDateTime >= max date - 7`.

It may be using a pre-filtered table, upload batch, or another date column.