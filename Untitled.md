Total Reads Last 30 Days =
VAR MaxReadDate =
    CALCULATE(
        MAX(cib_tbl_readership[ReadDateTime]),
        ALL(cib_tbl_readership)
    )
RETURN
CALCULATE(
    COUNTROWS(cib_tbl_readership),
    cib_tbl_readership[ReadDateTime] >= MaxReadDate - 30,
    cib_tbl_readership[ReadDateTime] <= MaxReadDate,
    cib_tbl_readership[Channel] IN {
        "Bloomberg",
        "Email (Click)",
        "Email (Open)"
    }
)