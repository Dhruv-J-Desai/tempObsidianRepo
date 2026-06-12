Below is a **Databricks SQL Notebook script** you can use to create the tables shown in your screenshot.

I used simple data types because the screenshot shows the table/column names but not exact source data types. You can later change `STRING`, `DATE`, `TIMESTAMP`, `INT`, or `DOUBLE` based on the actual Aspen source schema.

```sql
-- Optional: set your catalog and schema first
-- USE CATALOG your_catalog_name;
-- USE SCHEMA your_schema_name;

CREATE TABLE IF NOT EXISTS tbl_dim_doc_aspen (
    DocID STRING,
    Title STRING,
    PublishDate DATE,
    DocumentType STRING,
    StrategistID STRING,
    Contribute_Group STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS tbl_readership_aspen (
    DataUploadDatetime TIMESTAMP,
    DocID STRING,
    ReaderID STRING,
    AccountID STRING,
    Channel STRING,
    ReadDateTime TIMESTAMP,
    IPAddress STRING,
    LinkClick INT,
    CountryCode STRING,
    system_source STRING,
    ProspectTradingClient STRING,
    Hit_BM_ID STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS lookup_countrytoregion_aspen (
    CountryCode STRING,
    Country STRING,
    Region STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS tbl_dim_reader_aspen (
    ReaderID STRING,
    ReaderEmail STRING,
    ReaderName STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS tbl_producttag_rpt_aspen (
    DataUploadDatetime TIMESTAMP,
    DocID STRING,
    Hits INT,
    SentEmails INT,
    LinkViews INT,
    EmailViews INT,
    HitShare DOUBLE,
    EmailShare DOUBLE,
    EmailReads INT,
    NonEmailReads INT,
    Rates INT,
    Rates_Sovereigns INT,
    Rates_Derivatives INT,
    Rates_Regulatory INT,
    Rates_Inflation INT,
    Rates_Short_End INT,
    Rates_Spread_Products INT,
    Rates_ESG INT,
    Macro INT,
    Macro_Politics INT,
    Macro_Central_Bank INT,
    Macro_Inflation INT,
    Macro_Growth INT,
    Macro_Trade INT,
    Macro_ESG INT,
    EM INT
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS lookup_strategist_aspen (
    StrategistID STRING,
    Strategists STRING,
    StrategistsNoComma STRING,
    Position STRING,
    StrategistCountry STRING,
    StrategistTeam STRING,
    ProductCovered STRING,
    Region STRING,
    Cty STRING,
    ActiveInactive STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS tbl_bmcontacts_aspen (
    DataUploadDatetime TIMESTAMP,
    ID STRING,
    FirstName STRING,
    LastName STRING,
    ReaderID STRING,
    Account STRING,
    ContactOwner STRING,
    Status STRING,
    Tier STRING,
    IPRestriction STRING,
    Address1 STRING,
    Email_BBG STRING,
    Email_Corp STRING,
    Opted_In_Out STRING,
    Region STRING,
    Client_NonClient STRING,
    Email STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS lookup_regionbyclientcountry_aspen (
    ClientCountryStateProv STRING,
    RegionBroad1 STRING,
    RegionNarrow1 STRING,
    StateProvinceCountry STRING,
    RegionBroad2 STRING,
    RegionNarrow2 STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS tbl_meetingtracker_aspen (
    DataUploadDatetime TIMESTAMP,
    MeetingID STRING,
    Date DATE,
    StrategistID STRING,
    AccountID STRING,
    MeetingType STRING,
    PrimarySales STRING,
    OtherSales STRING,
    OtherStrat STRING,
    TopicDiscussed STRING,
    Desk STRING,
    ClientCountryStateProv STRING,
    Notes STRING,
    ClientArea STRING,
    ClientRegion STRING,
    ClientLocation STRING
)
USING DELTA;
```

```sql
CREATE TABLE IF NOT EXISTS lookup_accountpropername_clienttype_aspen (
    AccountID STRING,
    AccountNameRaw STRING,
    AccountProperName STRING,
    ClientType STRING,
    SFClientCode STRING,
    ParentEntity STRING,
    MurexID STRING
)
USING DELTA;
```

### Optional: add constraints / primary-key style metadata

Databricks does not enforce primary keys like a traditional database in most Delta table scenarios, but you can still document them using constraints if your workspace supports informational constraints.

```sql
ALTER TABLE tbl_dim_doc_aspen
ADD CONSTRAINT pk_tbl_dim_doc_aspen PRIMARY KEY (DocID) NOT ENFORCED;

ALTER TABLE tbl_dim_reader_aspen
ADD CONSTRAINT pk_tbl_dim_reader_aspen PRIMARY KEY (ReaderID) NOT ENFORCED;

ALTER TABLE lookup_countrytoregion_aspen
ADD CONSTRAINT pk_lookup_countrytoregion_aspen PRIMARY KEY (CountryCode) NOT ENFORCED;

ALTER TABLE lookup_strategist_aspen
ADD CONSTRAINT pk_lookup_strategist_aspen PRIMARY KEY (StrategistID) NOT ENFORCED;

ALTER TABLE lookup_accountpropername_clienttype_aspen
ADD CONSTRAINT pk_lookup_accountpropername_clienttype_aspen PRIMARY KEY (AccountID) NOT ENFORCED;

ALTER TABLE lookup_regionbyclientcountry_aspen
ADD CONSTRAINT pk_lookup_regionbyclientcountry_aspen PRIMARY KEY (ClientCountryStateProv) NOT ENFORCED;
```

### Suggested relationship mapping for Mosaic / BI model

After creating these tables, the main relationships appear to be:

```text
tbl_readership_aspen.DocID          -> tbl_dim_doc_aspen.DocID
tbl_readership_aspen.ReaderID       -> tbl_dim_reader_aspen.ReaderID
tbl_readership_aspen.CountryCode    -> lookup_countrytoregion_aspen.CountryCode
tbl_readership_aspen.DocID          -> tbl_producttag_rpt_aspen.DocID

tbl_dim_doc_aspen.StrategistID      -> lookup_strategist_aspen.StrategistID

tbl_bmcontacts_aspen.ReaderID       -> tbl_dim_reader_aspen.ReaderID
tbl_bmcontacts_aspen.Account        -> lookup_accountpropername_clienttype_aspen.AccountID

tbl_meetingtracker_aspen.StrategistID -> lookup_strategist_aspen.StrategistID
tbl_meetingtracker_aspen.AccountID    -> lookup_accountpropername_clienttype_aspen.AccountID
tbl_meetingtracker_aspen.ClientCountryStateProv
    -> lookup_regionbyclientcountry_aspen.ClientCountryStateProv
```

For quick initial testing in Databricks, start with the `CREATE TABLE` statements only. Once the tables are created and sample data is loaded, then validate the joins before building the Mosaic Model.