Yes — here is the full code using **PREFIX** for dynamic params.

This includes:

1. governance table
    
2. sample registration with `PREFIX`
    
3. low-level HTTP function
    
4. governed UC function
    
5. PySpark wrapper
    

---

## 1. Governance table

```sql
CREATE TABLE IF NOT EXISTS `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry (
    api_name STRING,
    base_url STRING,
    path_url_type STRING,   -- EXACT or PREFIX
    path_value STRING,
    approved BOOLEAN,
    owner_team STRING,
    notes STRING
);
```

---

## 2. Register endpoint using PREFIX

Use `PREFIX` when query params will be appended.

```sql
INSERT INTO `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry
(api_name, base_url, path_url_type, path_value, approved, owner_team, notes)
VALUES
(
  'postman_mock',
  'https://f0ea5df4-0ce9-45dc-9a86-06fce04fdb58.mock.pstmn.io',
  'PREFIX',
  '/udf-demo',
  true,
  'TDSBI',
  'Allows dynamic query params under /udf-demo'
);
```

You can verify:

```sql
SELECT *
FROM `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry;
```

---

## 3. Low-level HTTP caller

```sql
CREATE OR REPLACE FUNCTION `d4001-centralus-tdvip-tdsbi_catalog`.bronze.http_call(
    full_url STRING
)
RETURNS STRING
LANGUAGE PYTHON
AS $$
import requests

try:
    r = requests.get(full_url, timeout=10)
    return str(r.status_code)
except Exception as e:
    return f"ERROR: {str(e)}"
$$;
```

---

## 4. Governed API function

This is the governed layer.

- checks API registration
    
- checks approval
    
- checks path against registry
    
- uses `PREFIX` when query params are present
    

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
            ) = 0
            THEN concat(
                'ERROR: API ''',
                api_name,
                ''' is not registered in api_governance_registry'
            )

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
                ''' is registered but not approved'
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

## 5. PySpark wrapper

This lets the user pass:

- `csv_path`
    
- `api_name`
    
- `endpoint`
    
- optional params
    

No hardcoded CSV columns inside the function.

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
    Reads a CSV from Volume storage, builds request paths per row,
    and routes each row through the governed UC function.

    params example:
    {
        "limit_id": {"type": "column", "value": "Limit ID"},
        "env": {"type": "literal", "value": "prod"}
    }
    """

    # Read CSV
    df = spark.read.csv(csv_path, header=True, inferSchema=True)

    # Build req_path
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

    # Call governed UC function per row
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

## 6. Example usage

### Example with dynamic params from CSV columns

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",
    endpoint="/udf-demo",
    params={
        "limit_id": {"type": "column", "value": "Limit ID"},
        "limit_name": {"type": "column", "value": "Limit Name"},
        "is_active": {"type": "column", "value": "Is Active"}
    }
)

display(result_df)
```

### Example with mixed column and literal values

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",
    endpoint="/udf-demo",
    params={
        "limit_id": {"type": "column", "value": "Limit ID"},
        "region": {"type": "literal", "value": "us-east"},
        "env": {"type": "literal", "value": "prod"}
    }
)

display(result_df)
```

### Example with no params

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",
    endpoint="/udf-demo"
)

display(result_df)
```

---

## 7. Why PREFIX is needed here

If a row builds:

```text
/udf-demo?limit_id=80986&limit_name=Energy%20Delta
```

then this will match registry row:

```text
path_url_type = PREFIX
path_value = /udf-demo
```

but it would **not** match:

```text
path_url_type = EXACT
path_value = /udf-demo
```

So for dynamic params, `PREFIX` is the right registration with your current logic.

If you want, I can also give you a cleaner version that strips query params first, so you can still register `/udf-demo` as `EXACT`.