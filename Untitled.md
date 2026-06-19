Use this in Databricks SQL:

```sql
WITH max_read AS (
    SELECT
        MAX(ReadDateTime) AS Max_Read
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership`
),

last_7 AS (
    SELECT
        r.*
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership` r
    CROSS JOIN max_read m
    WHERE r.ReadDateTime BETWEEN m.Max_Read - INTERVAL 7 DAYS
                             AND m.Max_Read
)

SELECT
    d.Title,

    SUM(
        CASE 
            WHEN r.Channel = 'Email (Open)' 
            THEN 1 ELSE 0 
        END
    ) AS Email_Open,

    SUM(
        CASE 
            WHEN r.Channel = 'Bloomberg' 
            THEN 1 ELSE 0 
        END
    ) AS Online,

    SUM(
        CASE 
            WHEN r.Channel IN ('Email (Open)', 'Bloomberg') 
            THEN 1 ELSE 0 
        END
    ) AS Grand_Total

FROM last_7 r
JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_dim_doc` d
    ON r.DocID = d.DocID

GROUP BY
    d.Title

HAVING
    Grand_Total > 0

ORDER BY
    Grand_Total DESC,
    Email_Open DESC,
    Online DESC,
    d.Title;
```

This is doing:

```text
Max_Read = MAX(ReadDateTime) from cib_tbl_readership

Last 7 Days = ReadDateTime between Max_Read - 7 days and Max_Read

Email_Open = count rows where Channel = 'Email (Open)'

Online = count rows where Channel = 'Bloomberg'
```

Before comparing with Power BI/Strategy, also check the exact channel values:

```sql
SELECT
    Channel,
    COUNT(*) AS row_count
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership`
GROUP BY Channel
ORDER BY row_count DESC;
```

If Databricks shows `Bloomberg & Library` instead of `Bloomberg`, change the query condition to that exact value.