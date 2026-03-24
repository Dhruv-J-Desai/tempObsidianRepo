Exactly — they **should not** have to write that whole `WITH requested ... JOIN ...` query.

That SQL is good for the **PoC**, but for actual use you should wrap it behind a **single callable function** so callers only pass:

- `api_name`
    
- `path`
    

and nothing else.

Databricks supports Unity Catalog **SQL scalar functions** and **table functions**, so you can encapsulate logic in a function instead of exposing the join/query to users. ([docs.databricks.com](https://docs.databricks.com/aws/en/sql/language-manual/sql-ref-syntax-ddl-create-sql-function?utm_source=chatgpt.com "CREATE FUNCTION (SQL and Python) | Databricks on AWS"))

## Better design

Keep:

- the UC table as the registry
    
- the Python UDF as the low-level HTTP caller
    

Then add:

- a **SQL wrapper function** that does the lookup and calls the HTTP UDF
    

So the caller uses only:

```sql
SELECT `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api('repo_td', '/repository/tds-npm-all-release/');
```

## Full code

### 1. UC governance table

```sql
CREATE TABLE IF NOT EXISTS `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry (
  api_name STRING NOT NULL,
  base_url STRING NOT NULL,
  path_rule_type STRING NOT NULL,   -- EXACT or PREFIX
  path_value STRING NOT NULL,
  approved BOOLEAN NOT NULL,
  owner_team STRING,
  notes STRING,
  created_ts TIMESTAMP DEFAULT current_timestamp(),
  updated_ts TIMESTAMP DEFAULT current_timestamp()
);
```

### 2. Seed rows

```sql
INSERT INTO `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry
(api_name, base_url, path_rule_type, path_value, approved, owner_team, notes)
VALUES
('repo_td', 'https://repo.td.com', 'EXACT', '/', true, 'TDSBI', 'Root endpoint'),
('repo_td', 'https://repo.td.com', 'PREFIX', '/repository/', true, 'TDSBI', 'Allow repository child paths'),
('repo_td', 'https://repo.td.com', 'PREFIX', '/#browse/search/', true, 'TDSBI', 'Allow browse/search child paths'),
('strategy_td', 'https://td-dev.cloud.strategy.com', 'EXACT', '/MicroStrategy/servlet/mstrWeb', true, 'TDSBI', 'Strategy endpoint');
```

### 3. Low-level HTTP UDF

```sql
CREATE OR REPLACE FUNCTION `d4001-centralus-tdvip-tdsbi_catalog`.bronze.http_get(full_url STRING)
RETURNS STRING
LANGUAGE PYTHON
AS $$
import requests

try:
    r = requests.get(full_url, timeout=10)
    return f"{r.status_code}"
except Exception as e:
    return f"ERROR: {str(e)}"
$$;
```

### 4. Wrapper function callers will use

This is the main piece you were missing.

```sql
CREATE OR REPLACE FUNCTION `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
  api_name STRING,
  req_path STRING
)
RETURNS STRING
LANGUAGE SQL
RETURN
  SELECT COALESCE(
    (
      SELECT `d4001-centralus-tdvip-tdsbi_catalog`.bronze.http_get(
        concat(g.base_url, req_path)
      )
      FROM `d4001-centralus-tdvip-tdsbi_catalog`.bronze.api_governance_registry g
      WHERE g.api_name = call_governed_api.api_name
        AND g.approved = true
        AND (
          (g.path_rule_type = 'EXACT'  AND req_path = g.path_value)
          OR
          (g.path_rule_type = 'PREFIX' AND req_path LIKE concat(g.path_value, '%'))
        )
      LIMIT 1
    ),
    'ERROR: endpoint/path is not approved in api_governance_registry'
  );
```

## How users call it

### Approved

```sql
SELECT `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
  'repo_td',
  '/repository/tds-npm-all-release/'
);
```

### Another approved one

```sql
SELECT `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
  'repo_td',
  '/#browse/search/maven'
);
```

### Unapproved

```sql
SELECT `d4001-centralus-tdvip-tdsbi_catalog`.bronze.call_governed_api(
  'repo_td',
  '/not-approved-path/'
);
```

That should return:

```text
ERROR: endpoint/path is not approved in api_governance_registry
```

## Why this is better

Now the caller:

- does **not** need to know table structure
    
- does **not** write joins
    
- does **not** know about `path_rule_type`
    
- only uses a single governed function
    

So this gives you a proper abstraction:

- **registry table** for governance
    
- **wrapper SQL function** for business use
    
- **HTTP Python UDF** for execution
    

## Important note

This is a clean governed interface, but it still does **not** stop someone from writing their own notebook code or a different UDF if they have permission. That part is the separate “inherit cluster policy / platform enforcement” discussion.

## What to tell Eilam

You can say:

> I agree callers should not need to know the governance table structure or write joins. I can wrap the table lookup behind a single UC SQL function, so consumers only call something like `call_governed_api(api_name, path)` while the approval check stays hidden behind the function.

If you want, I can also give you a version that returns the response body instead of just HTTP status.