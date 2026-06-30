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

trade_ideas AS (
    SELECT
        d.Title AS Document_Title,

        SUM(CASE 
                WHEN r.Channel = 'Bloomberg' 
                THEN 1 ELSE 0 
            END) AS Bloomberg,

        SUM(CASE 
                WHEN r.Channel = 'Email (Click)' 
                THEN 1 ELSE 0 
            END) AS Email_Click,

        SUM(CASE 
                WHEN r.Channel = 'Email (Open)' 
                THEN 1 ELSE 0 
            END) AS Email_Open,

        SUM(CASE 
                WHEN r.Channel IN ('Bloomberg', 'Email (Click)', 'Email (Open)') 
                THEN 1 ELSE 0 
            END) AS Grand_Total_Last_30_Days

    FROM last_30 r
    INNER JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_dim_doc` d
        ON r.DocID = d.DocID

    WHERE d.DocumentType = 'Trade Idea'

    GROUP BY
        d.Title
)

SELECT
    Document_Title,
    Bloomberg,
    Email_Click,
    Email_Open,
    Grand_Total_Last_30_Days
FROM trade_ideas
WHERE Grand_Total_Last_30_Days > 0
ORDER BY
    Grand_Total_Last_30_Days DESC,
    Bloomberg DESC,
    Email_Click DESC,
    Email_Open DESC,
    Document_Title
LIMIT 10;