Got it. Since Databricks physical column names are different from Strategy/Mosaic display names, use the **Databricks column names** directly.

Based on your screenshots, use this query:

```sql
SELECT
    -- Latest equity price fields
    lep.TICKER AS ticker,
    lep.NAME AS equity_name,
    lep.PRICE AS latest_price,
    lep.EXCH_CODE AS exchange_code,
    lep.CRNCY AS currency_code,
    lep.SECURITY_TYP AS security_type,
    lep.ID_ISIN AS isin,
    lep.ID_CUSIP AS cusip,
    lep.DL_SNAPSHOT_DATE AS snapshot_date,
    lep.DL_SNAPSHOT_START_TIME AS snapshot_start_time,
    lep.DL_SNAPSHOT_END_TIME AS snapshot_end_time,
    lep.`$file` AS source_file_name,

    -- Instrument details
    di.instrument_id,
    di.symbol,
    di.instrument_type,
    di.asset_class_code,
    di.issuer_id,
    di.primary_exchange_id,
    di.currency_code AS instrument_currency_code,
    di.lot_size,
    di.tick_size,
    di.is_active,

    -- Asset class details
    da.asset_class_name,

    -- Issuer details
    dis.issuer_name,
    dis.issuer_lei,
    dis.credit_rating,
    dis.country_code AS issuer_country_code,
    dis.sector_id,
    dis.industry_id,

    -- Exchange details
    de.exchange_id,
    de.exchange_name,
    de.mic_code,
    de.country_code AS exchange_country_code,
    de.timezone AS exchange_timezone,

    -- Country / region details
    rco.country_name AS exchange_country,
    rco.region AS exchange_region,

    -- Currency details
    rc.currency_name,
    rc.symbol AS currency_symbol

FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.latest_equity_prices lep

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_instrument di
    ON lep.ID_CUSIP = di.cusip

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_asset_class da
    ON di.asset_class_code = da.asset_class_code

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_issuer dis
    ON di.issuer_id = dis.issuer_id

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_exchange de
    ON lep.EXCH_CODE = de.mic_code

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_country rco
    ON de.country_code = rco.country_code

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_currency rc
    ON lep.CRNCY = rc.currency_code;
```

If `lep.EXCH_CODE = de.mic_code` does not match well, use this alternate exchange join:

```sql
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_exchange de
    ON di.primary_exchange_id = de.exchange_id
```

For testing the join quality, run this first:

```sql
SELECT
    COUNT(*) AS total_rows,
    COUNT(di.instrument_id) AS matched_instruments,
    COUNT(de.exchange_id) AS matched_exchanges,
    COUNT(rc.currency_code) AS matched_currencies,
    COUNT(da.asset_class_code) AS matched_asset_classes,
    COUNT(dis.issuer_id) AS matched_issuers
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.latest_equity_prices lep
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_instrument di
    ON lep.ID_CUSIP = di.cusip
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_asset_class da
    ON di.asset_class_code = da.asset_class_code
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_issuer dis
    ON di.issuer_id = dis.issuer_id
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_exchange de
    ON lep.EXCH_CODE = de.mic_code
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_currency rc
    ON lep.CRNCY = rc.currency_code;
```

This will tell you whether the joins are actually matching or returning nulls.