Yes — you can keep the same governance idea in PySpark.

Meaning:

- **Spark** reads the CSV
    
- **PySpark** checks the governance registry
    
- **PySpark** builds the approved full URL
    
- **PySpark** calls the HTTP function only for approved rows
    
- otherwise returns governed error messages
    

So it becomes the PySpark equivalent of your SQL `call_governed_api`.

## Recommended approach

### Keep these 2 things

- `api_governance_registry` table
    
- low-level HTTP caller function or direct requests call in Spark logic
    

### Move this logic into PySpark

Like your SQL function, do these checks in order:

1. Is `api_name` registered and approved?
    
2. Is `req_path` allowed for that API?
    
3. If yes, make the call
    
4. Else return clear error
    

---

## Better PySpark governed pattern

```python
from pyspark.sql import functions as F

def process_csv_api_calls_governed(
    csv_path: str,
    api_name: str,
    endpoint_col_expr,
    output_table: str | None = None
):
    # 1. Read CSV from Volume
    df = spark.read.csv(csv_path, header=True, inferSchema=True)

    # 2. Build req_path per row
    df = df.withColumn("req_path", endpoint_col_expr)

    # 3. Load governance registry for this API only
    registry_df = (
        spark.table("d4001-centralus-tdvip-tdsbi_catalog.bronze.api_governance_registry")
        .filter((F.col("api_name") == api_name) & (F.col("approved") == True))
        .select("api_name", "base_url", "path_url_type", "path_value")
    )

    # 4. Check whether API exists at all
    if registry_df.limit(1).count() == 0:
        return df.withColumn(
            "api_result",
            F.lit(f"ERROR: API '{api_name}' is not registered in api_governance_registry")
        )

    # 5. Join each row with registry rows for same API
    joined = df.crossJoin(registry_df)

    # 6. Match EXACT or PREFIX rules
    matched = joined.filter(
        (
            (F.col("path_url_type") == "EXACT") &
            (F.col("req_path") == F.col("path_value"))
        ) |
        (
            (F.col("path_url_type") == "PREFIX") &
            (F.col("req_path").startswith(F.col("path_value")))
        )
    )

    # 7. Rows that matched governance
    approved_rows = matched.withColumn(
        "full_url",
        F.concat(F.col("base_url"), F.col("req_path"))
    )

    # 8. Rows that did not match governance
    unapproved_rows = df.join(
        matched.select(*df.columns).distinct(),
        on=df.columns,
        how="left_anti"
    ).withColumn(
        "api_result",
        F.concat(
            F.lit("ERROR: Path '"),
            F.col("req_path"),
            F.lit(f"' is not approved for API '{api_name}'")
        )
    )

    # 9. Call governed SQL function per approved row
    approved_rows.createOrReplaceTempView("approved_rows_tmp")

    result_approved = spark.sql(f"""
        SELECT
            *,
            `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api('{api_name}', req_path) AS api_result
        FROM approved_rows_tmp
    """)

    # 10. Union both
    final_df = result_approved.select(*df.columns, "api_result").unionByName(
        unapproved_rows.select(*df.columns, "api_result")
    )

    if output_table:
        final_df.write.mode("overwrite").saveAsTable(output_table)

    return final_df
```

---

## Example usage

For your limits CSV:

```python
from pyspark.sql import functions as F

result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",
    endpoint_col_expr=F.concat(
        F.lit("/udf-demo?limit_id="),
        F.col("Limit ID").cast("string"),
        F.lit("&limit_name="),
        F.regexp_replace(F.col("Limit Name"), " ", "%20"),
        F.lit("&is_active="),
        F.col("Is Active").cast("string")
    )
)

display(result_df)
```

---

## Cleaner version

If you do **not** want Spark to do the approval join itself, then the simplest governed PySpark design is:

- Spark reads CSV
    
- Spark builds `req_path`
    
- Spark calls **only** `call_governed_api(api_name, req_path)` per row
    

That is actually closer to your current SQL governance.

### Simpler PySpark version

```python
from pyspark.sql import functions as F

def process_csv_via_governed_function(csv_path: str, api_name: str):
    df = spark.read.csv(csv_path, header=True, inferSchema=True)

    df = df.withColumn(
        "req_path",
        F.concat(
            F.lit("/udf-demo?limit_id="),
            F.col("Limit ID").cast("string"),
            F.lit("&limit_name="),
            F.regexp_replace(F.col("Limit Name"), " ", "%20"),
            F.lit("&is_active="),
            F.col("Is Active").cast("string")
        )
    )

    df.createOrReplaceTempView("csv_input_tmp")

    return spark.sql(f"""
        SELECT
            *,
            `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api('{api_name}', req_path) AS api_result
        FROM csv_input_tmp
    """)
```

This is probably the **best** option for you.

Because:

- governance stays centralized in UC SQL function
    
- PySpark just orchestrates file reading and row shaping
    
- it is easier to explain and maintain
    

---

## Recommendation

Use this model:

### PySpark responsibility

- read CSV
    
- filter rows
    
- build dynamic request path
    

### UC governed function responsibility

- validate API registration
    
- validate allowed path
    
- execute HTTP call
    
- return error if not approved
    

That gives you the same governance model as before, just with Spark handling file processing.

## What to tell Eilam

You can say:

> I can keep the same governance model in PySpark by letting Spark handle the CSV read and row-level request-path construction, while still routing each row through the UC governed function for validation and execution. So Spark would orchestrate the batch/file processing, but the actual API approval logic would remain centralized in the governance registry and governed function.

If you want, I can rewrite your exact notebook code into this simpler governed PySpark version.