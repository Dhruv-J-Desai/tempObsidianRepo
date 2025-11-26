Exactly 👍 — the fact that **`trade` isn’t listed under `Query`** in the Docs pane is the smoking gun.

Now the job is: **teach DGS/GraphQL that `Query.trade` is a real field**, not just a DataFetcher.

Below is how to do that _programmatically_ in your existing `@DgsCodeRegistry` method.

---

## 1. What DGS is doing right now

Your method probably looks roughly like:

```java
@DgsCodeRegistry
public GraphQLCodeRegistry.Builder registry(GraphQLCodeRegistry.Builder codeRegistryBuilder,
                                            TypeDefinitionRegistry registry) {

    Collection<DatabricksTable> tables = databricksService.getAllTables(databricksCatalog, false);

    for (var table : tables) {
        String fieldName = table.getName();  // "trade"

        DataFetcher<List<Map<String, Object>>> getFromDatabricks = env -> {
            // ...
        };

        FieldCoordinates coords = FieldCoordinates.coordinates("Query", fieldName);
        codeRegistryBuilder.dataFetcher(coords, getFromDatabricks);
    }

    return codeRegistryBuilder;
}
```

So you’re only wiring **DataFetchers**.  
We need to **also mutate the SDL** (`TypeDefinitionRegistry`) to add the fields.

---

## 2. Add `trade` as a field on `Query` (programmatically)

Here’s one way to extend your method so, for _each_ table, we:

- add a `Query.<fieldName>` field definition
    
- (optionally) add a `Subscription.<fieldName>` field too
    
- then wire the DataFetcher
    

```java
@DgsComponent
@RequiredArgsConstructor
public class DynamicDataFetcher {

    private final DatabricksService databricksService;
    private final ConcurrentKafkaListenerContainerFactory<String, String> containerFactory;
    private final ObjectMapper objectMapper;

    @Value("${dataproductgen.catalog}")
    private String databricksCatalog;

    @DgsCodeRegistry
    public GraphQLCodeRegistry.Builder registry(GraphQLCodeRegistry.Builder codeRegistryBuilder,
                                                TypeDefinitionRegistry typeRegistry) {

        Collection<DatabricksTable> tables =
                databricksService.getAllTables(databricksCatalog, false);

        for (DatabricksTable table : tables) {

            String fieldName = table.getName(); // e.g. "trade"

            // 1) ----- mutate Query type SDL -----
            addFieldToRootType(typeRegistry, "Query", fieldName);

            // (optional) also add field to Subscription root:
            // addFieldToRootType(typeRegistry, "Subscription", fieldName);

            // 2) ----- create DataFetcher -----
            DataFetcher<List<Map<String, Object>>> getFromDatabricks =
                    env -> getAllDataFromTopic(fieldName); // or topicName

            FieldCoordinates queryCoords =
                    FieldCoordinates.coordinates("Query", fieldName);
            codeRegistryBuilder.dataFetcher(queryCoords, getFromDatabricks);

            // If you support subscription data fetchers you’d also
            // wire them on ("Subscription", fieldName) here.
        }

        return codeRegistryBuilder;
    }

    /**
     * Adds a field like:
     *
     *   type Query {
     *       <fieldName>: [Json]   // or your own type
     *   }
     *
     * to the given root type, if not already present.
     */
    private void addFieldToRootType(TypeDefinitionRegistry typeRegistry,
                                    String rootTypeName,
                                    String fieldName) {

        // Find the root type definition (Query or Subscription)
        typeRegistry.getType(rootTypeName).ifPresent(def -> {
            ObjectTypeDefinition root = (ObjectTypeDefinition) def;

            // Don’t add twice
            boolean exists = root.getFieldDefinitions().stream()
                    .anyMatch(f -> f.getName().equals(fieldName));
            if (exists) return;

            // Define field: change return type as you like
            FieldDefinition fieldDef = FieldDefinition
                    .newFieldDefinition()
                    .name(fieldName)
                    // here I use [Json]; you can use a specific type name instead
                    .type(new ListType(new TypeName("Json")))
                    .build();

            ObjectTypeDefinition newRoot = root.transform(builder ->
                    builder.fieldDefinition(fieldDef));

            typeRegistry.remove(root);
            typeRegistry.add(newRoot);
        });
    }

    private List<Map<String, Object>> getAllDataFromTopic(String topicName) {
        // your existing Kafka container logic here
        // return List<Map<String,Object>>;
        return List.of();
    }
}
```

### Notes

- `TypeDefinitionRegistry` is the SDL representation DGS builds from your `.graphqls` files.
    
- By transforming `ObjectTypeDefinition Query` we’re effectively doing:
    
    ```graphql
    extend type Query {
      trade: [Json]
    }
    ```
    
    but programmatically.
    
- I used `[Json]` as the return type just as an example; if you already have a `Trade` type and want `[Trade]`, change:
    
    ```java
    .type(new ListType(new TypeName("Trade")))
    ```
    

---

## 3. How to verify it worked

After you restart the app:

1. Open GraphiQL Docs.
    
2. Click on `Query`.
    
3. You should now see fields like:
    
    - `address`
        
    - `trade`
        
    - `customerEvents`
        
    - etc. — one per Databricks table you looped over.
        
4. Run:
    
    ```graphql
    {
      trade {
        payload
        topic
        key
        headers
        ingest_time
        event_time
        feed_name
        ingestion_id
        next_zone_status
        next_zone_error
        next_zone_processed_at
      }
    }
    ```
    
    Now validation should pass and your `getAllDataFromTopic("trade")` should be hit.
    

---

If you paste your current `DynamicDataFetcher` Java file, I can inline these changes into _your_ exact class so it’s a straight copy-paste for you.