Here’s a full governed pattern version.

It does this:

- Spark reads the CSV from the Volume
    
- Spark builds a per-row request path
    
- Unity Catalog governance table controls allowed APIs/paths
    
- UC SQL function validates and calls the HTTP function
    
- Spark stores the result per row
    

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

## 2. Sample registry rows

```sql
INSERT INTO `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry
(api_name, base_url, path_url_type, path_value, approved, owner_team, notes)
VALUES
('repo_td', 'https://repo.td.com', 'EXACT', '/', true, 'TD', 'Root endpoint'),
('repo_td', 'https://repo.td.com', 'PREFIX', '/#browse/search/', true, 'TD', 'Allow search child paths'),
('postman_mock', 'https://f0ea5df4-0ce9-45dc-9a86-06fce04fdb58.mock.pstmn.io', 'PREFIX', '/udf-demo', true, 'TD', 'Mock endpoint');
```

---

## 3. Low-level HTTP function

```sql
CREATE OR REPLACE FUNCTION `d4001-centralus-tdvip-tdsbi_catalog`.bronze.http_call(
  full_url STRING
)
RETURNS STRING
LANGUAGE PYTHON
AS $$
import requests
import json

try:
    r = requests.get(full_url, timeout=10, verify=False)
    return json.dumps({
        "status": r.status_code,
        "response": r.text[:500]
    })
except Exception as e:
    return json.dumps({
        "error": str(e)
    })
$$;
```

---

## 4. Governed UC function

This validates `api_name` and `req_path` against the governance table before calling the HTTP function.

```sql
CREATE OR REPLACE FUNCTION `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
  api_name STRING,
  req_path STRING
)
RETURNS STRING
LANGUAGE SQL
RETURN
WITH params AS (
  SELECT
    api_name AS p_api_name,
    req_path AS p_req_path,
    split(req_path, '\\?')[0] AS p_base_path
),
api_rows AS (
  SELECT g.*
  FROM `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry g
  CROSS JOIN params p
  WHERE g.api_name = p.p_api_name
    AND g.approved = true
),
matched_rows AS (
  SELECT g.*
  FROM api_rows g
  CROSS JOIN params p
  WHERE
    (g.path_url_type = 'EXACT'  AND p.p_base_path = g.path_value)
    OR
    (g.path_url_type = 'PREFIX' AND p.p_base_path LIKE concat(g.path_value, '%'))
),
final_call AS (
  SELECT
    `d4001-centralus-tdvip-tdsbi_catalog`.bronze.http_call(
      concat(base_url, (SELECT p_req_path FROM params))
    ) AS response
  FROM matched_rows
  LIMIT 1
)
SELECT
  CASE
    WHEN (SELECT COUNT(*) FROM api_rows) = 0 THEN
      concat(
        'ERROR: API ''',
        api_name,
        ''' is not registered in api_governance_registry'
      )
    WHEN (SELECT COUNT(*) FROM matched_rows) = 0 THEN
      concat(
        'ERROR: Path ''',
        req_path,
        ''' is not approved for API ''',
        api_name,
        ''''
      )
    ELSE
      (SELECT response FROM final_call)
  END;
```

---

## 5. PySpark wrapper

This is the part that reads the CSV from the Volume and routes each row through the governed UC function.

```python
from pyspark.sql import functions as F

def process_csv_api_calls(
    csv_path: str,
    api_name: str,
    output_table: str | None = None,
    limit_rows: int | None = None
):
    # Read CSV from Volume
    df = spark.read.option("header", "true").csv(csv_path)

    # Optional row limit for testing
    if limit_rows is not None:
        df = df.limit(limit_rows)

    # Build dynamic request path per row
    # Adjust column names here if needed
    df = df.withColumn(
        "req_path",
        F.concat(
            F.lit("/udf-demo?limit_id="),
            F.col("Limit ID"),
            F.lit("&limit_name="),
            F.regexp_replace(F.col("Limit Name"), " ", "%20"),
            F.lit("&is_active="),
            F.col("Is Active")
        )
    )

    # Register temp view so SQL can call governed UC function row by row
    df.createOrReplaceTempView("tmp_api_input")

    result_df = spark.sql(f"""
        SELECT
            `Limit ID`,
            `Limit Name`,
            `Is Active`,
            req_path,
            `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
                '{api_name}',
                req_path
            ) AS api_result
        FROM tmp_api_input
    """)

    # Parse JSON result if returned
    result_df = (
        result_df
        .withColumn("status_code", F.get_json_object(F.col("api_result"), "$.status").cast("int"))
        .withColumn("response_body", F.get_json_object(F.col("api_result"), "$.response"))
        .withColumn("error", F.get_json_object(F.col("api_result"), "$.error"))
    )

    if output_table:
        result_df.write.mode("overwrite").saveAsTable(output_table)

    return result_df
```

---

## 6. Call the PySpark wrapper

```python
results = process_csv_api_calls(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="postman_mock",   # or repo_td
    output_table="d4001-centralus-tdvip-tdsbi_catalog.bronze.api_call_results_demo",
    limit_rows=3
)

results.show(truncate=False)
```

---

## 7. If you want to use `repo_td` instead

Replace the request-path builder in the PySpark wrapper with something simpler:

```python
df = df.withColumn("req_path", F.lit("/"))
```

or:

```python
df = df.withColumn("req_path", F.lit("/#browse/search/maven"))
```

Then call:

```python
results = process_csv_api_calls(
    csv_path="/Volumes/d4001-centralus-tdvip-tdsbi_catalog/bronze/idr_poc_limits/sample_limit_energy_updated.csv",
    api_name="repo_td",
    limit_rows=3
)
```

---

## 8. Optional audit table

```sql
CREATE TABLE IF NOT EXISTS `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_call_results_demo (
  `Limit ID` STRING,
  `Limit Name` STRING,
  `Is Active` STRING,
  req_path STRING,
  api_result STRING,
  status_code INT,
  response_body STRING,
  error STRING
);
```

---

## What this gives you

- file reading stays in Spark
    
- governance stays in Unity Catalog
    
- no raw HTTP calls directly from Spark business logic
    
- every row goes through the governed UC function
    

If you want, I can also give you a cleaner version with row filters like `Is Active = TRUE` already built in.