Yes, check it directly in Databricks first. Start with the same logic Strategy is using.

## 1. Check the max read date

```sql
SELECT
  MAX(ReadDateTime) AS Max_Read,
  dateadd(MAX(ReadDateTime), -7) AS Max_Read_7,
  dateadd(MAX(ReadDateTime), -30) AS Max_Read_30
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership;
```

This tells you the exact date range Strategy is using.

---

## 2. Check Top Documents - Last 7 Days

```sql
WITH custom_cib_tbl_readership AS (
  SELECT
    a.*,
    dateadd(MAX(a.ReadDateTime) OVER (), -7) AS Max_Read_7,
    dateadd(MAX(a.ReadDateTime) OVER (), -30) AS Max_Read_30,
    MAX(a.ReadDateTime) OVER () AS Max_Read,
    CASE
      WHEN a.ReadDateTime BETWEEN dateadd(MAX(a.ReadDateTime) OVER (), -7)
                              AND MAX(a.ReadDateTime) OVER ()
      THEN 1 ELSE 0
    END AS read_7,
    CASE
      WHEN a.ReadDateTime BETWEEN dateadd(MAX(a.ReadDateTime) OVER (), -30)
                              AND MAX(a.ReadDateTime) OVER ()
      THEN 1 ELSE 0
    END AS read_30
  FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership a
)
SELECT
  d.Title,
  SUM(CASE WHEN r.Channel = 'Email (Open)' THEN 1 ELSE 0 END) AS Email_Open,
  SUM(CASE WHEN r.Channel = 'Bloomberg' THEN 1 ELSE 0 END) AS Online,
  SUM(CASE WHEN r.Channel IN ('Email (Open)', 'Bloomberg') THEN 1 ELSE 0 END) AS Grand_Total
FROM custom_cib_tbl_readership r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_dim_doc d
  ON r.DocID = d.DocID
WHERE r.read_7 = 1
GROUP BY d.Title
HAVING Grand_Total > 0
ORDER BY Grand_Total DESC, Email_Open DESC, Online DESC, d.Title
LIMIT 50;
```

---

## 3. Check Top Documents - Last 30 Days

```sql
WITH custom_cib_tbl_readership AS (
  SELECT
    a.*,
    dateadd(MAX(a.ReadDateTime) OVER (), -7) AS Max_Read_7,
    dateadd(MAX(a.ReadDateTime) OVER (), -30) AS Max_Read_30,
    MAX(a.ReadDateTime) OVER () AS Max_Read,
    CASE
      WHEN a.ReadDateTime BETWEEN dateadd(MAX(a.ReadDateTime) OVER (), -7)
                              AND MAX(a.ReadDateTime) OVER ()
      THEN 1 ELSE 0
    END AS read_7,
    CASE
      WHEN a.ReadDateTime BETWEEN dateadd(MAX(a.ReadDateTime) OVER (), -30)
                              AND MAX(a.ReadDateTime) OVER ()
      THEN 1 ELSE 0
    END AS read_30
  FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership a
)
SELECT
  d.Title,
  SUM(CASE WHEN r.Channel = 'Email (Open)' THEN 1 ELSE 0 END) AS Email_Open,
  SUM(CASE WHEN r.Channel = 'Bloomberg' THEN 1 ELSE 0 END) AS Online,
  SUM(CASE WHEN r.Channel IN ('Email (Open)', 'Bloomberg') THEN 1 ELSE 0 END) AS Grand_Total
FROM custom_cib_tbl_readership r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_dim_doc d
  ON r.DocID = d.DocID
WHERE r.read_30 = 1
GROUP BY d.Title
HAVING Grand_Total > 0
ORDER BY Grand_Total DESC, Email_Open DESC, Online DESC, d.Title
LIMIT 50;
```

---

## 4. If Power BI still differs, check channel values

Run this:

```sql
SELECT
  Channel,
  COUNT(*) AS row_count
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.cib_tbl_readership
GROUP BY Channel
ORDER BY row_count DESC;
```

If Databricks shows `Bloomberg & Library` instead of `Bloomberg`, then your Power BI DAX should use that exact value.