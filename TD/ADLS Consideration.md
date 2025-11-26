Great — now your **duplicate-field errors are gone**, which means the `ensureRootField()` fix is working.

Now you’re seeing a **new error**, which is:

> **The type `Json` is not present when resolving type `Subscription`**

This is very easy to fix — GraphQL is telling you:

> “You are adding a field whose return type is `Json`, but you have _not defined_ what `Json` means anywhere in the schema.”

---

# ✅ Why this error now?

In your dynamic code, you are doing:

```java
ensureRootField(registry, "Query", table.getName(), "Json");
ensureRootField(registry, "Subscription", table.getName(), "Json");
```

So you are adding schema like:

```graphql
type Query {
  trade: [Json]
}

type Subscription {
  trade: [Json]
}
```

But **GraphQL does not know what `Json` is.**

There is:

- no scalar named `Json`
    
- no object type named `Json`
    
- no custom scalar registered in your runtime wiring
    

So the schema becomes invalid.

That’s why the error is:

> **The type 'Json' is not present**

---

# ✅ How to fix (3 options)

---

## **Option 1 (Recommended): use DGS built-in `@DgsScalar(UUID: JSON)`**

DGS provides a JSON scalar named **`_Any`**, but not plain `Json`.

Spring Boot GraphQL has `GraphQLScalarType` for JSON, but not DGS by default.

Instead, DGS suggests using:

```java
import com.netflix.graphql.dgs.scalars.DgsScalar;
import com.netflix.graphql.dgs.internal.types.DgsScalarTypes;
```

They provide:

```graphql
scalar _Any
```

But better: **Define your own JSON scalar once**, and reuse it.

---

### 👉 Add this class:

```java
@DgsScalar(name = "Json")
public class JsonScalar implements Coercing<Object, Object> {
    @Override
    public Object serialize(Object dataFetcherResult) {
        return dataFetcherResult;
    }

    @Override
    public Object parseValue(Object input) {
        return input;
    }

    @Override
    public Object parseLiteral(Object input) {
        if (input instanceof Value) {
            return ((Value) input).toString();
        }
        return null;
    }
}
```

Add this file anywhere in your project (e.g. `scalars/JsonScalar.java`).

### Then your dynamic field:

```java
ensureRootField(registry, "Query", tableName, "Json");
```

starts working, because `Json` now exists.

---

## **Option 2 — Don’t use Json at all (use String)**

If your table returns a raw Map and you want GraphQL to return a JSON string:

Do:

```java
ensureRootField(registry, "Query", tableName, "String");
```

This completely avoids custom scalars.

---

## **Option 3 — Use a proper object type instead of Json**

For example:

```graphql
type Trade {
  payload: String
  topic: String
  ingest_time: String
}
```

Then:

```java
ensureRootField(registry, "Query", tableName, "Trade");
```

But this requires generating a GraphQLObjectType dynamically — more work.

---

# 🔥 What you should do right now

**Fastest working fix** = Option 1: add a JSON scalar.

---

# 📌 Steps to fix your current error

### 1. Create a JSON scalar class:

```java
package com.example.scalars;

import com.netflix.graphql.dgs.DgsScalar;
import graphql.language.Value;
import graphql.schema.Coercing;

@DgsScalar(name = "Json")
public class JsonScalar implements Coercing<Object, Object> {
    @Override
    public Object serialize(Object dataFetcherResult) {
        return dataFetcherResult;
    }

    @Override
    public Object parseValue(Object input) {
        return input;
    }

    @Override
    public Object parseLiteral(Object input) {
        if (input instanceof Value) {
            return input.toString();
        }
        return null;
    }
}
```

### 2. Restart the app.

### 3. Run GraphQL again.

The Subscription + Query fields will now be valid, because `Json` is a known GraphQL type.

---

# ⭐ Confirming why Subscription specifically failed

The error message says:

> “when resolving type `Subscription`”

Because both:

```java
ensureRootField(registry, "Subscription", table.getName(), "Json")
```

AND your DataFetcher return type use `"Json"`.

GraphQL validates subscription schema strictly.

---

# If you want, I can rewrite your whole `DynamicDataFetcher` into a clean, production-ready class that works with:

- dynamic schema merging
    
- JSON scalar
    
- table schema detection
    
- bronze/silver/gold separation
    
- full wiring for Query + Subscription
    
- no duplicate fields
    
- no missing types
    

Just say **“Rewrite DynamicDataFetcher completely”**.