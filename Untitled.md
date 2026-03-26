Got it.

You want:

- user passes `csv_path`
    
- user passes `api_name`
    
- user passes `endpoint`
    
- user can optionally pass extra parameters
    
- **not hardcode column names** like `Limit ID`, `Limit Name`, `Is Active`
    
- base_url should come automatically from governance table
    

That’s the right design.

## How base_url is identified

`base_url` should **not** be passed by the user.

It should come from `api_governance_registry` using `api_name`.

So the flow is:

1. user passes `api_name = 'postman_mock'`
    
2. code looks up governance table
    
3. finds:
    
    - `api_name`
        
    - `base_url`
        
    - allowed path rules
        
4. code builds final URL using:
    
    - `base_url` from registry
        
    - `req_path` from endpoint + params
        

So the user never passes raw base URL.

---

## Better design

Let user pass:

- `csv_path`
    
- `api_name`
    
- `endpoint`
    
- optional `params`
    

Where `params` can be:

- fixed values
    
- or column references
    

Example idea:

```python
params = {
    "limit_id": {"type": "column", "value": "Limit ID"},
    "limit_name": {"type": "column", "value": "Limit Name"},
    "is_active": {"type": "column", "value": "Is Active"},
    "env": {"type": "literal", "value": "prod"}
}
```

That way the function is generic.

---

## Generic governed version

```python
from pyspark.sql import functions as F

def process_csv_api_calls_governed(
    csv_path: str,
    api_name: str,
    endpoint: str,
    params: dict | None = None,
    output_table: str | None = None
):
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
                raise ValueError(f"Unsupported param type for {param_name}: {config['type']}")

            param_parts.append(F.concat(F.lit(f"{param_name}="), value_expr))

        if param_parts:
            query_expr = param_parts[0]
            for p in param_parts[1:]:
                query_expr = F.concat(query_expr, F.lit("&"), p)

            req_path = F.concat(F.lit(endpoint), F.lit("?"), query_expr)

    df = df.withColumn("req_path", req_path)

    # Use governed function
    df.createOrReplaceTempView("csv_input_tmp")

    result_df = spark.sql(f"""
        SELECT
            *,
            `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api('{api_name}', req_path) AS api_result
        FROM csv_input_tmp
    """)

    if output_table:
        result_df.write.mode("overwrite").saveAsTable(output_table)

    return result_df
```

---

## Example usage

### Example 1: all params from columns

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",
    endpoint="/udf-demo",
    params={
        "limit_id": {"type": "column", "value": "Limit ID"},
        "limit_name": {"type": "column", "value": "Limit Name"},
        "is_active": {"type": "column", "value": "Is Active"},
    }
)

display(result_df)
```

### Example 2: mix of column + literal

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",
    endpoint="/udf-demo",
    params={
        "limit_id": {"type": "column", "value": "Limit ID"},
        "region": {"type": "literal", "value": "us-east"},
        "env": {"type": "literal", "value": "prod"},
    }
)

display(result_df)
```

### Example 3: no params

```python
result_df = process_csv_api_calls_governed(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="repo_td",
    endpoint="/"
)

display(result_df)
```

---

## Why this is better

Now:

- no hardcoded CSV column names in the function
    
- user chooses what to pass
    
- base URL stays governed in registry
    
- endpoint approval still happens through `call_governed_api`
    

---

## Important architecture point

This function does **not** identify `base_url` in Spark directly.

It identifies it indirectly because this line:

```python
call_governed_api('{api_name}', req_path)
```

calls your governed UC SQL function, and **that function** looks up `base_url` from the governance table.

That is the cleanest design.

---

## Cleaner mental model

User provides:

- file
    
- approved API name
    
- endpoint
    
- optional params
    

Governance function provides:

- base_url lookup
    
- allowed path validation
    
- HTTP execution
    

Spark provides:

- CSV reading
    
- row shaping
    
- parameter mapping
    

---

If you want, I can make this even cleaner by allowing `params` to be passed as a simple shorter format like:

```python
params = {
    "limit_id": "Limit ID",
    "limit_name": "Limit Name",
    "env": ("literal", "prod")
}
```