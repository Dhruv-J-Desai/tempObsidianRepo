For a **GET** request, you generally should **not use request body**.

Use **query parameters** instead.

So for your Postman mock:

### Better approach

Endpoint:

```text
GET /udf-demo/echo
```

Parameters go in the URL, like:

```text
/udf-demo/echo?limit_id=80986&limit_name=Energy%20Delta&is_active=TRUE
```

## Why

Because GET requests usually pass inputs through:

- query params
    
- path params
    

not request body.

## In Postman mock server

You can define the request as:

- Method: `GET`
    
- URL: `{{url}}/udf-demo/echo`
    

Then the response body can still use variables like:

```json
{
  "status": "success",
  "message": "Mock API called successfully",
  "limit_id": "{{limit_id}}",
  "limit_name": "{{limit_name}}",
  "is_active": "{{is_active}}"
}
```

## So what to do in that screen

For the request body column:

- leave it empty for GET
    

Instead, later when calling the mock URL, pass parameters in the URL itself.

## If you really want request body

Then use **POST**, not GET.

Example:

### Endpoint

```text
POST /udf-demo/process-row
```

### Request body

```json
{
  "limit_id": "80986",
  "limit_name": "Energy Delta",
  "is_active": "TRUE"
}
```

### Response body

```json
{
  "status": "success",
  "message": "Row processed successfully",
  "limit_id": "{{limit_id}}",
  "limit_name": "{{limit_name}}",
  "is_active": "{{is_active}}"
}
```

But for your Databricks demo, **GET with query params is simpler**.

## Recommendation

Use:

- **GET**
    
- no request body
    
- pass row values as query params
    

So the answer is:

**For GET, leave request body empty. Use query parameters instead.**

If you want, I can next give you the exact mock URL pattern and the Databricks function code to call it per row.