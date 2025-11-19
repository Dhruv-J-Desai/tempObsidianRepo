### 🔥 **This is 100% the root cause.**

Your screenshot shows the exact JsonPath exception:

```
Expected to find an object with property [clientId] in path but found: 
'java.lang.String' that does not match the expected type. 
Invalid JSON object according to the jsonProvider
```

This error message tells us **everything**:

---

# ✅ **Root Cause**

`doc` is **NOT a parsed JSON object**.  
It is a **String**, and JsonPath thinks that the entire string is the value of `clientId` or `validFrom`.

Meaning:

Instead of:

```json
{
  "clientId": "C004",
  "symbol": "AVGO",
  ...
}
```

You are passing this _literal string_ into JsonPath:

```
"{\"clientId\":\"C004\",\"symbol\":\"AVGO\", ... }"
```

JsonPath sees it as:

```
java.lang.String = "{ \"clientId\": \"C004\", ... }"
```

and therefore:

```
JsonPath.using(cfg).parse(doc).read("$.validFrom")
```

= ERROR  
because `"doc"` is not JSON, it’s a plain string.

---

# ✅ **Why is this happening?**

You are calling:

```java
eval(doc, expr)
```

with **doc = the raw JSON string** from Kafka.

Inside `apply()` you do:

```java
out.put(e.getKey(), eval(doc, e.getValue()));  
```

But **doc has not been parsed yet**.

JsonPath **can** parse raw JSON strings, but **only if they are valid JSON strings**, not Java-escaped strings.

Your JSON string likely contains either:

- escaped quotes,
    
- or double-escaped quotes,
    
- or has newline/whitespace issues.
    

JsonPath rejects it as _not valid JSON_.

---

# ✅ **The correct fix**

Parse JSON **once** before calling `eval()`.

### 🔧 Change your code to:

```java
public Map<String, Object> apply(Object doc, Map<String, String> mapping) {
    // Always parse once
    Object parsed = cfg.jsonProvider().parse(doc.toString());

    Map<String, Object> out = new LinkedHashMap<>();
    for (var e : mapping.entrySet()) {
        out.put(e.getKey(), eval(parsed, e.getValue()));
    }
    return out;
}
```

Then inside `eval()`:

```java
if (first.startsWith("$")) {
    cur = JsonPath.read(doc, first);   // doc is already parsed
}
```

---

# ⭐ **Final Working Pattern**

### Step 1 — Parse once

```java
Object parsedDoc = cfg.jsonProvider().parse(jsonString);
```

### Step 2 — Evaluate mappings

```java
Object value = JsonPath.read(parsedDoc, "$.validFrom");
```

### Step 3 — Apply transforms (`trim`, `upper`, `parse_ts`, etc.)

---

# 🎯 **Why this works**

JsonPath internally expects:

- a `Map`
    
- or a `List`
    
- or a JSON-parse tree
    

NOT a Java string.

By parsing it first, you give JsonPath a proper JSON tree, so:

```java
JsonPath.read(parsedDoc, "$.validFrom")
```

returns `"2025-11-19T..."` instead of throwing.

---

# 🚀 Final Note

Nothing is wrong with your mapping.  
Nothing wrong with JsonPath.

**The only issue was that `doc` was not parsed JSON in `eval()`.**

Fix the `apply()` method so that `eval()` receives a parsed JSON, and everything will work.

---

If you want, send me your current `apply()` + `eval()` code and I’ll give you the exact corrected version.