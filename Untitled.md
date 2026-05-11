Based on your Mosaic relationship diagram, `latest_equity_prices` **does have `cusip`**, so the main join to `dim_instrument` should be on:

```sql
lep.cusip = di.cusip
```

Use this query in Databricks / Free-form SQL:

```sql
SELECT
    -- Main latest equity price fields
    lep.ticker,
    lep.equity_name,
    lep.security_type,
    lep.exchange AS price_exchange,
    lep.currency_code AS price_currency_code,
    lep.cusip,
    lep.snapshot_date,
    lep.snapshot_start_time,
    lep.snapshot_end_time,
    lep.source_file_name,

    -- Instrument details
    di.instrument,
    di.instrument_type,
    di.asset_class AS instrument_asset_class,
    di.issuer,
    di.isin,
    di.is_active,
    di.lot_size,
    di.exchange AS instrument_exchange,
    di.currency_code AS instrument_currency_code,

    -- Asset class details
    da.asset_class AS asset_class_name,
    da.`Asset Class (2)` AS asset_class_group,

    -- Issuer details
    dis.credit_rating,
    dis.industry,
    dis.sector,
    dis.country_code AS issuer_country_code,

    -- Exchange details
    de.country_code AS exchange_country_code,
    de.time_zone AS exchange_time_zone,

    -- Currency details
    rc.currency AS currency_name,

    -- Country / region details
    rco.country AS exchange_country,
    rco.region AS exchange_region

FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.latest_equity_prices lep

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_instrument di
    ON lep.cusip = di.cusip

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_asset_class da
    ON di.asset_class = da.asset_class

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_issuer dis
    ON di.issuer = dis.issuer

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_exchange de
    ON lep.exchange = de.exchange

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_currency rc
    ON lep.currency_code = rc.currency_code

LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_country rco
    ON de.country_code = rco.country_code;
```

One small thing: if `Asset Class (2)` fails because of the space/parentheses, first run:

```sql
DESCRIBE `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_asset_class;
```

Then use the actual Databricks column name. It may be something like:

```sql
asset_class_2
```

Before using this in Mosaic, validate row duplication:

```sql
SELECT
    COUNT(*) AS joined_row_count,
    COUNT(DISTINCT lep.cusip) AS distinct_cusip_count,
    COUNT(DISTINCT lep.ticker) AS distinct_ticker_count
FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.latest_equity_prices lep
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_instrument di
    ON lep.cusip = di.cusip
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_asset_class da
    ON di.asset_class = da.asset_class
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_issuer dis
    ON di.issuer = dis.issuer
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_exchange de
    ON lep.exchange = de.exchange
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_currency rc
    ON lep.currency_code = rc.currency_code
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_country rco
    ON de.country_code = rco.country_code;
```

For dashboard testing, you can also create a lighter version:

```sql
SELECT
    lep.ticker,
    lep.equity_name,
    lep.security_type,
    lep.exchange,
    lep.currency_code,
    lep.cusip,
    lep.snapshot_date,

    di.instrument,
    di.instrument_type,
    di.asset_class,
    di.issuer,
    di.isin,

    dis.industry,
    dis.sector,
    dis.credit_rating,

    rc.currency,
    rco.country,
    rco.region

FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.latest_equity_prices lep
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_instrument di
    ON lep.cusip = di.cusip
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_issuer dis
    ON di.issuer = dis.issuer
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_currency rc
    ON lep.currency_code = rc.currency_code
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.dim_exchange de
    ON lep.exchange = de.exchange
LEFT JOIN `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.raw.ref_country rco
    ON de.country_code = rco.country_code;
```

This lighter query is probably better for your dashboard because it avoids pulling too many columns.