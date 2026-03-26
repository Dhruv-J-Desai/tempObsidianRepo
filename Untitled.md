You’re right to call that out.

In the PySpark helper, the **governed part is only present if** this function already exists and is used as the enforcement layer:

```sql
`d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(api_name, req_path)
```

So the full governed solution should include **both**:

1. the governed UC SQL function
    
2. the PySpark wrapper that routes every row through it
    

Below is the full version.

---

## 1. Governed API function in UC

This function:

- checks whether the API is registered
    
- checks whether the path is approved
    
- if approved, calls the low-level HTTP function
    
- otherwise returns a governed error
    

```sql
CREATE OR REPLACE FUNCTION `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
    api_name STRING,
    req_path STRING
)
RETURNS STRING
LANGUAGE SQL
RETURN
    SELECT
        CASE
            WHEN (
                SELECT COUNT(*)
                FROM `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry g
                CROSS JOIN (SELECT api_name AS p_api_name) params
                WHERE g.api_name = params.p_api_name
                  AND g.approved = true
            ) = 0
            THEN concat(
                'ERROR: API ''',
                api_name,
                ''' is not registered in api_governance_registry'
            )

            WHEN (
                SELECT COUNT(*)
                FROM `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry g
                CROSS JOIN (SELECT api_name AS p_api_name, req_path AS p_req_path) params
                WHERE g.api_name = params.p_api_name
                  AND g.approved = true
                  AND (
                        (g.path_url_type = 'EXACT'  AND params.p_req_path = g.path_value)
                     OR (g.path_url_type = 'PREFIX' AND params.p_req_path LIKE concat(g.path_value, '%'))
                  )
            ) = 0
            THEN concat(
                'ERROR: Path ''',
                req_path,
                ''' is not approved for API ''',
                api_name,
                ''''
            )

            ELSE (
                SELECT `d4001-centralus-tdvip-tdsbi_catalog`.bronze.http_call(
                    concat(g.base_url, req_path)
                )
                FROM `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry g
                CROSS JOIN (SELECT api_name AS p_api_name, req_path AS p_req_path) params
                WHERE g.api_name = params.p_api_name
                  AND g.approved = true
                  AND (
                        (g.path_url_type = 'EXACT'  AND params.p_req_path = g.path_value)
                     OR (g.path_url_type = 'PREFIX' AND params.p_req_path LIKE concat(g.path_value, '%'))
                  )
                LIMIT 1
            )
        END;
```

---

## 2. PySpark wrapper that uses the governed function

This version:

- reads the CSV from Volume
    
- lets user pass `api_name`, `endpoint`, and optional params
    
- builds `req_path`
    
- calls the **governed** UC function for each row
    
- never takes raw `base_url` from the user
    

```python
from pyspark.sql import functions as F

def process_csv_api_calls_governed(
    csv_path: str,
    api_name: str,
    endpoint: str,
    params: dict | None = None,
    output_table: str | None = None
):
    """
    Reads a CSV from Volume storage, builds a governed request path per row,
    and invokes the UC governed function call_governed_api(api_name, req_path).

    params format:
    {
        "param_name": {"type": "column", "value": "Column Name"},
        "param2": {"type": "literal", "value": "fixed_value"}
    }
    """

    # Read CSV
    df = spark.read.csv(csv_path, header=True, inferSchema=True)

    # Build req_path dynamically
    req_path = F.lit(endpoint)

    if params:
        param_parts = []

        for param_name, config in params.items():
            if config["type"] == "column":
                value_expr = F.col(config["value"]).cast("string")
                value_expr = F.regexp_replace(value_expr, " ", "%20")
            elif config["type"] == "literal":
                value_expr = F.lit(str(config["value"]))
            else:
                raise ValueError(
                    f"Unsupported param type for {param_name}: {config['type']}"
                )

            param_parts.append(F.concat(F.lit(f"{param_name}="), value_expr))

        if param_parts:
            query_expr = param_parts[0]
            for p in param_parts[1:]:
                query_expr = F.concat(query_expr, F.lit("&"), p)

            req_path = F.concat(F.lit(endpoint), F.lit("?"), query_expr)

    df = df.withColumn("req_path", req_path)

    # Register temp view
    df.createOrReplaceTempView("csv_input_tmp")

    # Route every row through the governed UC function
    result_df = spark.sql(f"""
        SELECT
            *,
            `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
                '{api_name}',
                req_path
            ) AS api_result
        FROM csv_input_tmp
    """)

    if output_table:
        result_df.write.mode("overwrite").saveAsTable(output_table)

    return result_df
```

---

## 3. Example usage

### Example with CSV columns plus literal params

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",
    endpoint="/udf-demo",
    params={
        "limit_id": {"type": "column", "value": "Limit ID"},
        "limit_name": {"type": "column", "value": "Limit Name"},
        "is_active": {"type": "column", "value": "Is Active"},
        "env": {"type": "literal", "value": "prod"}
    }
)

display(result_df)
```

### Example with no params

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="repo_td",
    endpoint="/"
)

display(result_df)
```

---

## 4. Why this is governed

This is now governed because:

- user passes only `api_name`
    
- `base_url` is resolved from `api_governance_registry`
    
- path approval is checked in `call_governed_api`
    
- unapproved paths return governed error messages
    
- Spark only orchestrates row processing
    

So yes, the governed part is not in the Python helper itself — it is intentionally centralized in the UC SQL function.

If you want, I can also give you a cleaner version where the wrapper does not use a temp view and instead uses `selectExpr`.