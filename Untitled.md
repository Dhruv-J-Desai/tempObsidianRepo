Yes, your Power BI setup looks correct now:

```text
Rows:
Account
Reader Name
Title

Values:
Bloomberg & Library
Email Link
Email Open
```

So if you still see different rows than Strategy, then the difference is most likely due to one of these:

1. **Power BI is sorting Account alphabetically**, so `1832 Asset Management` appears first.
    
2. **Strategy and Power BI may not have the exact same filters applied**.
    
3. **The account field may be different** between tools.
    
4. **Power BI relationship path may be different**, especially between `account`, `reader`, `title`, and `readership`.
    
5. **Strategy may be using a custom/freeform SQL table**, while Power BI is using raw tables.
    

Use this Databricks SQL to validate the **Top Readers** output.

```sql
WITH max_read AS (
    SELECT
        MAX(ReadDateTime) AS max_read_datetime
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership`
),

last_30 AS (
    SELECT
        r.*
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership` r
    CROSS JOIN max_read m
    WHERE r.ReadDateTime BETWEEN m.max_read_datetime - INTERVAL 30 DAYS
                             AND m.max_read_datetime
),

top_readers AS (
    SELECT
        a.AccountProperName AS Account_Name,
        rd.ReaderName AS Reader_Name,
        d.Title AS Document_Title,

        SUM(
            CASE 
                WHEN r.Channel = 'Bloomberg' 
                THEN 1 ELSE 0 
            END
        ) AS Bloomberg_Library,

        SUM(
            CASE 
                WHEN r.Channel = 'Email (Click)' 
                THEN 1 ELSE 0 
            END
        ) AS Email_Link,

        SUM(
            CASE 
                WHEN r.Channel = 'Email (Open)' 
                THEN 1 ELSE 0 
            END
        ) AS Email_Open,

        SUM(
            CASE 
                WHEN r.Channel IN ('Bloomberg', 'Email (Click)', 'Email (Open)') 
                THEN 1 ELSE 0 
            END
        ) AS Grand_Total_Last_30_Days

    FROM last_30 r

    LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_dim_doc` d
        ON r.DocID = d.DocID

    LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_dim_reader` rd
        ON r.ReaderID = rd.ReaderID

    LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_lookup_accountpropername_clienttype` a
        ON r.AccountID = a.AccountID

    GROUP BY
        a.AccountProperName,
        rd.ReaderName,
        d.Title
)

SELECT
    Account_Name,
    Reader_Name,
    Document_Title,
    Bloomberg_Library,
    Email_Link,
    Email_Open,
    Grand_Total_Last_30_Days
FROM top_readers
WHERE Grand_Total_Last_30_Days > 0
ORDER BY
    Account_Name,
    Reader_Name,
    Document_Title;
```

If your actual account lookup table name is different, replace this part:

```sql
`d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_lookup_accountpropername_clienttype`
```

with the table you are using in Power BI.

Also, based on your Power BI model screenshot, your channel field values may be:

```text
Bloomberg
Email (Click)
Email (Open)
```

But your column label says:

```text
Bloomberg & Library
Email (Link)
Email (Open)
```

So if the source actually uses different names, first validate with:

```sql
SELECT
    Channel,
    COUNT(*) AS row_count
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership`
GROUP BY Channel
ORDER BY row_count DESC;
```

To check specifically why `1832 Asset Management` is appearing first:

```sql
WITH max_read AS (
    SELECT
        MAX(ReadDateTime) AS max_read_datetime
    FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership`
)

SELECT
    a.AccountProperName AS Account_Name,
    COUNT(*) AS Total_Reads_Last_30_Days
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership` r
CROSS JOIN max_read m
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_lookup_accountpropername_clienttype` a
    ON r.AccountID = a.AccountID
WHERE r.ReadDateTime BETWEEN m.max_read_datetime - INTERVAL 30 DAYS
                         AND m.max_read_datetime
GROUP BY
    a.AccountProperName
ORDER BY
    Account_Name;
```

If this returns `1832 Asset Management`, then Power BI is correct to show it. Strategy is probably either sorted differently or has a filter excluding it.