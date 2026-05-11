Yes, that is a better approach for a clean dashboard. Instead of bringing all columns/tables into Mosaic, create a **Free-form SQL dataset** with only the fields needed for the dashboard.

Use `latest_equity_prices` as the main table and join dimensions around it.

### Example Free-form SQL

```sql
SELECT
    lep.ticker,
    lep.equity_name,
    lep.security_type,
    lep.exchange,
    lep.currency_code,
    lep.cusip,
    lep.snapshot_date,
    lep.snapshot_start_time,
    lep.snapshot_end_time,
    lep.source_file_name,

    di.instrument,
    di.instrument_type,
    di.asset_class,
    di.issuer,
    di.isin,
    di.is_active,

    da.asset_class AS asset_class_name,

    de.country_code AS exchange_country_code,
    de.time_zone AS exchange_time_zone,

    rc.currency,

    rco.country,
    rco.region,

    dis.credit_rating,
    dis.industry,
    dis.sector

FROM latest_equity_prices lep

LEFT JOIN dim_instrument di
    ON lep.cusip = di.cusip

LEFT JOIN dim_asset_class da
    ON di.asset_class = da.asset_class

LEFT JOIN dim_exchange de
    ON lep.exchange = de.exchange

LEFT JOIN ref_currency rc
    ON lep.currency_code = rc.currency_code

LEFT JOIN ref_country rco
    ON de.country_code = rco.country_code

LEFT JOIN dim_issuer dis
    ON di.issuer = dis.issuer;
```

If your tables are in Databricks Unity Catalog, use the fully qualified name:

```sql
SELECT
    lep.ticker,
    lep.equity_name,
    lep.security_type,
    lep.exchange,
    lep.currency_code,
    lep.cusip,
    lep.snapshot_date,
    lep.snapshot_start_time,
    lep.snapshot_end_time,
    lep.source_file_name,

    di.instrument,
    di.instrument_type,
    di.asset_class,
    di.issuer,
    di.isin,
    di.is_active,

    da.asset_class AS asset_class_name,

    de.country_code AS exchange_country_code,
    de.time_zone AS exchange_time_zone,

    rc.currency,

    rco.country,
    rco.region,

    dis.credit_rating,
    dis.industry,
    dis.sector

FROM `catalog_name`.`schema_name`.`latest_equity_prices` lep

LEFT JOIN `catalog_name`.`schema_name`.`dim_instrument` di
    ON lep.cusip = di.cusip

LEFT JOIN `catalog_name`.`schema_name`.`dim_asset_class` da
    ON di.asset_class = da.asset_class

LEFT JOIN `catalog_name`.`schema_name`.`dim_exchange` de
    ON lep.exchange = de.exchange

LEFT JOIN `catalog_name`.`schema_name`.`ref_currency` rc
    ON lep.currency_code = rc.currency_code

LEFT JOIN `catalog_name`.`schema_name`.`ref_country` rco
    ON de.country_code = rco.country_code

LEFT JOIN `catalog_name`.`schema_name`.`dim_issuer` dis
    ON di.issuer = dis.issuer;
```

### For dashboard metrics, create only these

Once this SQL dataset is created, create metrics like:

```text
Total Securities = CountDistinct(cusip)
Total Tickers = CountDistinct(ticker)
Total Exchanges = CountDistinct(exchange)
Total Currencies = CountDistinct(currency_code)
Total Issuers = CountDistinct(issuer)
Latest Snapshot Date = Max(snapshot_date)
```

### Suggested visuals from this SQL

|Visual|Attribute|Metric|
|---|---|---|
|KPI|—|Total Securities|
|Bar chart|Exchange|CountDistinct(ticker)|
|Bar chart|Asset Class|CountDistinct(cusip)|
|Donut chart|Currency|CountDistinct(cusip)|
|Map / bar|Region|CountDistinct(cusip)|
|Grid|Ticker, Equity Name, Exchange, Currency, Asset Class, Issuer, Region|—|

### Important join validation

Before using it in Mosaic, run this in Databricks to check whether joins are duplicating rows:

```sql
SELECT
    COUNT(*) AS row_count,
    COUNT(DISTINCT lep.cusip) AS distinct_cusip,
    COUNT(DISTINCT lep.ticker) AS distinct_ticker
FROM latest_equity_prices lep
LEFT JOIN dim_instrument di
    ON lep.cusip = di.cusip
LEFT JOIN dim_asset_class da
    ON di.asset_class = da.asset_class
LEFT JOIN dim_exchange de
    ON lep.exchange = de.exchange
LEFT JOIN ref_currency rc
    ON lep.currency_code = rc.currency_code
LEFT JOIN ref_country rco
    ON de.country_code = rco.country_code
LEFT JOIN dim_issuer dis
    ON di.issuer = dis.issuer;
```

If `row_count` becomes much larger than expected, one of the dimension tables has duplicate keys. Then you may need to deduplicate dimension tables before joining.

For your use case, this SQL-based approach is cleaner because the Mosaic dashboard will be based on one curated dataset instead of many tables with auto-detected relationships.