OK — now I can clearly see **why your rows STILL show all `null` in DeepHaven / Angular.**

You **did NOT apply the real fix.**  
You only added `_parse_schema` (returns list of pairs), and then you still do:

```python
cols = _parse_schema(schema)
update_exprs = []
tbl = tbl.update(update_exprs)
```

🚨 **This is NOT how DeepHaven consumes Kafka JSON.**  
You are _manually_ trying to parse JSON inside Python using:

```python
f"{name} = (int) value['{name}']"
```

But your Kafka messages ARE like:

```json
{
  "clientId": "A101",
  "symbol": "TSLA",
  "direction": "BUY",
  "quantity": 10,
  "price": 200.50
}
```

Deephaven Kafka consumer does **NOT** expose a column named `"value"`, so this expression always fails silently → all columns become `null`.

---

# ✅ The REAL fix (you must replace your current logic)

You must **NOT** parse JSON manually.

You must call:

```python
value_spec = kc.json_spec(schema_dict)
```

and **schema_dict must be a real dict**, not a list, not a string.

Right now you have:

```python
v_spec = kc.json_spec()
```

and `schema` is a string, not a dict.

This is why everything stays null.

---

# ✅ Correct version of create_live_table (copy–paste EXACTLY this)

Replace your entire function body with this:

```python
def _schema_to_dict(schema):
    if isinstance(schema, dict):
        return schema

    if isinstance(schema, str):
        result = {}
        for line in schema.strip().splitlines():
            line = line.strip()
            if not line:
                continue

            name, dtype_expr = [p.strip() for p in line.split(":", 1)]

            # dtype_expr is "dht.string" etc
            dtype = eval(dtype_expr, {"dht": dht})
            result[name] = dtype

        return result

    raise TypeError(f"Invalid schema type: {type(schema)}")


def create_live_table(topic: str, schema, alias: str | None = None,
                      bootstrap: str | None = None, table_type: str = "append",
                      ignore_key: bool = True):
    if not topic:
        raise ValueError("topic is required")

    name = alias or topic.replace("-", "_")

    # -------------------------
    # FIX: schema must be dict
    # -------------------------
    schema_dict = _schema_to_dict(schema)

    config = {
        "bootstrap.servers": "pkc-k13op.canadacentral.azure.confluent.cloud:9092",
        "auto.offset.reset": "latest",
        "security.protocol": "SASL_SSL",
        "sasl.mechanism": "OAUTHBEARER",
        "sasl.login.callback.handler.class": "org.apache.kafka.common.security.oauthbearer.secured.OAuthBearerLoginCallbackHandler",
        "sasl.jaas.config":
            "org.apache.kafka.common.security.oauthbearer.OAuthBearerLoginModule required "
            'clientId="TestScopeClient" '
            'clientSecret="2Federate" '
            'scope="" '
            'extension_logicalCluster="lkc-ygyvwp" '
            'extension_identityPoolId="pool-NRk1";',
        "sasl.oauthbearer.token.endpoint.url": "https://fedsit.rastest.tdbank.ca/as/token.oauth2",
        "sasl.oauthbearer.sub.claim.name": "client_id",
        "sasl.oauthbearer.client.id": "TestScopeClient",
        "sasl.oauthbearer.client.secret": "2Federate",
        "sasl.oauthbearer.extensions.logicalCluster": "lkc-ygyvwp",
        "sasl.oauthbearer.extensions.identityPoolId": "pool-NRk1",
        "enable.auto.commit": "false",
        "deephaven.kafka.skip.partition.column": "true",
        "deephaven.kafka.skip.offset.column": "true",
        "deephaven.kafka.skip.timestamp.column": "true",
        "deephaven.kafka.skip.key.column": "true",
    }

    # -------------------------
    # FIX: correct json_spec usage
    # -------------------------
    value_spec = kc.json_spec(schema_dict)
    key_spec = kc.KeyValueSpec.IGNORE if ignore_key else None

    ttype = kc.TableType.Append if table_type.lower() == "append" else kc.TableType.Blink

    tbl = kc.consume(
        config,
        topic,
        key_spec=key_spec,
        value_spec=value_spec,
        table_type=ttype,
    )

    # cleanup metadata
    for col in ("KafkaPartition", "KafkaOffset", "KafkaTimestamp", "KafkaKey"):
        if col in tbl.columns:
            tbl = tbl.dropColumns(col)

    globals()[name] = tbl
    setattr(__main__, name, tbl)

    print(f"[DH] Created {name} from topic '{topic}'")
    return name
```

---

# 🚀 Why this works

Because now:

- `schema` string → `schema_dict = {"clientId": dht.string, ...}`
    
- `kc.json_spec(schema_dict)` tells Deephaven how to decode JSON
    
- Kafka consumer returns columns directly mapped to your JSON fields
    
- NO need for `tbl.update()` / `value['field']` parsing
    
- Angular receives clean objects:
    

```json
{
  "clientId": "A101",
  "symbol": "TSLA",
  "direction": "BUY",
  "quantity": 10,
  "price": 200.5
}
```

---

# 🔥 NEXT — Do this test

Inside Deephaven console run:

```python
trade.head(5)
```

If the values appear, Angular will immediately show non-null rows.

---

If you want, show me your **actual schema string** and **one actual Kafka JSON message** — I will generate the exact schema dict for you too.