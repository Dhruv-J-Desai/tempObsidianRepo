For **Strategy Mosaic Model**, use this SQL as a **Free-form SQL source** and make the output look like a clean curated dataset. The main idea is:

> Instead of importing all 6 tables separately and relying on auto-detected relationships, create one joined dataset from Databricks and then build the Mosaic Model/dashboard on top of that.

## Recommended Mosaic dataset name

Use something like:

```text
equity_price_instrument_analytics
```

or

```text
latest_equity_price_dashboard_model
```

## Use this SQL in Free-form SQL

This version is good for Mosaic because the aliases are dashboard-friendly and consistent:

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

## After importing into Mosaic Model

In Mosaic, classify fields like this:

### Attributes

Use these as attributes:

```text
ticker
equity_name
exchange_code
currency_code
security_type
isin
cusip
instrument_id
symbol
instrument_type
asset_class_code
issuer_id
asset_class_name
issuer_name
issuer_lei
credit_rating
issuer_country_code
sector_id
industry_id
exchange_id
exchange_name
mic_code
exchange_country_code
exchange_timezone
exchange_country
exchange_region
currency_name
currency_symbol
source_file_name
```

### Date/time attributes

Use these as date/time fields:

```text
snapshot_date
snapshot_start_time
snapshot_end_time
```

### Metrics

Create metrics on top of the imported dataset:

```text
Total Securities = CountDistinct(cusip)

Total Tickers = CountDistinct(ticker)

Average Latest Price = Avg(latest_price)

Maximum Latest Price = Max(latest_price)

Minimum Latest Price = Min(latest_price)

Total Exchanges = CountDistinct(exchange_code)

Total Currencies = CountDistinct(currency_code)

Total Issuers = CountDistinct(issuer_id)
```

For boolean `is_active`, avoid `Sum(is_active)` because Databricks/Spark does not allow summing boolean directly. Create a metric using a filter, or add this to SQL:

```sql
CASE WHEN di.is_active = true THEN 1 ELSE 0 END AS is_active_flag
```

Then in Mosaic:

```text
Active Instrument Count = Sum(is_active_flag)
```

## Suggested SQL improvement for Mosaic

I would add this field now to avoid the boolean issue:

```sql
CASE WHEN di.is_active = true THEN 1 ELSE 0 END AS is_active_flag
```

Place it near `di.is_active`.

Then your Mosaic model will have both:

```text
is_active
is_active_flag
```

Use `is_active` for filtering and `is_active_flag` for metrics.

## Dashboard ideas using this Mosaic dataset

### KPI cards

```text
Total Securities
Total Tickers
Average Latest Price
Total Exchanges
Total Currencies
Latest Snapshot Date
```

### Charts

```text
Securities by Exchange
Metric: CountDistinct(ticker)
Attribute: exchange_name or exchange_code
```

```text
Securities by Asset Class
Metric: CountDistinct(cusip)
Attribute: asset_class_name
```

```text
Average Price by Security Type
Metric: Avg(latest_price)
Attribute: security_type
```

```text
Securities by Currency
Metric: CountDistinct(cusip)
Attribute: currency_name
```

```text
Securities by Region
Metric: CountDistinct(cusip)
Attribute: exchange_region
```

### Detail grid

Use:

```text
ticker
equity_name
latest_price
exchange_name
currency_name
security_type
asset_class_name
issuer_name
credit_rating
exchange_country
exchange_region
snapshot_date
```

## Best Confluence note for this approach

```text
Instead of importing all related tables separately into the Mosaic Model, a curated Free-form SQL dataset was created using Databricks physical column names and explicit joins. This avoids confusion caused by Strategy renaming source columns during import and reduces dependency on auto-detected relationships. The resulting dataset includes only the required dashboard fields from latest equity prices, instrument, asset class, issuer, exchange, country, and currency tables. This approach provides a cleaner semantic layer for dashboard creation and makes validation against Databricks SQL easier.
```