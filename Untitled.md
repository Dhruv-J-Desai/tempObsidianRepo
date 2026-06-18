Yes — the **overall logic is correct** if the requirement is:

```text
Last 7 Days = Max(ReadDateTime) - 7 days to Max(ReadDateTime)
Last 30 Days = Max(ReadDateTime) - 30 days to Max(ReadDateTime)
```

But there are a few things I would check because they can cause mismatch.

## 1. `BETWEEN` is inclusive

This part:

```sql
a.readdatetime BETWEEN Max_Read - 7 AND Max_Read
```

includes both start and end dates.

So it means:

```text
ReadDateTime >= Max_Read - 7
AND ReadDateTime <= Max_Read
```

That is usually okay, but if the data has time values, it may include slightly more/less depending on timestamp handling.

## 2. `dateadd()` may drop the time portion

This is the main thing I would be careful about.

You are doing:

```sql
dateadd(MAX(a.readdatetime) OVER (), -7)
```

Depending on SQL engine behavior, `dateadd()` may return a **date** and not a full timestamp. If `ReadDateTime` has time, this can create slight differences.

Safer Databricks-style logic would be:

```sql
MAX(a.readdatetime) OVER () - INTERVAL 7 DAYS
```

and:

```sql
MAX(a.readdatetime) OVER () - INTERVAL 30 DAYS
```

So I would write it like this:

```sql
SELECT
    a.*,

    MAX(a.readdatetime) OVER () - INTERVAL 7 DAYS AS Max_Read_7,
    MAX(a.readdatetime) OVER () - INTERVAL 30 DAYS AS Max_Read_30,
    MAX(a.readdatetime) OVER () AS Max_Read,

    CASE
        WHEN a.readdatetime BETWEEN MAX(a.readdatetime) OVER () - INTERVAL 7 DAYS
                                AND MAX(a.readdatetime) OVER ()
        THEN 1 ELSE 0
    END AS read_7,

    CASE
        WHEN a.readdatetime BETWEEN MAX(a.readdatetime) OVER () - INTERVAL 30 DAYS
                                AND MAX(a.readdatetime) OVER ()
        THEN 1 ELSE 0
    END AS read_30

FROM `d4001-centralus-tdvip-tdsbi_mstrt_catalog`.`raw`.`cib_tbl_readership` a;
```

## 3. It is calculating the max date across the entire table

This part:

```sql
MAX(a.readdatetime) OVER ()
```

means the max date is calculated across **all rows in the table**, not per document, not per reader, not per account.

That matches what we saw in Strategy. So this is correct if Strategy wants one global max read date for the whole dashboard.

## 4. This query only creates flags, not metrics

The query creates:

```text
read_7
read_30
Max_Read
Max_Read_7
Max_Read_30
```

But the actual `Email_Open` / `Online` metrics are calculated later in Strategy using formulas like:

```text
Email_Open = Sum(Case(Channel = "Email (Open)", 1, 0))
Online = Sum(Case(Channel = "Bloomberg", 1, 0))
```

Then the dashboard applies:

```text
read_7 = 1
```

or:

```text
read_30 = 1
```

So the query logic is fine, but Power BI must apply the same filter logic on top of the channel measures.

## Main answer

The query logic is mostly correct. The only thing I would change is using `INTERVAL 7 DAYS` / `INTERVAL 30 DAYS` instead of `dateadd(...)`, to avoid any timestamp/date conversion issue.