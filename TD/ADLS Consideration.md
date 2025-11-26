Yes, 100% – **that’s exactly what you should do** and it will get rid of the “tried to redefine field …” errors.

You already have `TypeDefinitionRegistry` injected into your `@DgsCodeRegistry` method, so you can ask it:

> “Does `Query` already have a field called `X`? If yes → skip. If no → add it.”

Here’s a concrete pattern you can drop into your `DynamicDataFetcher`.

```java
@DgsCodeRegistry
public GraphQLCodeRegistry.Builder registry(GraphQLCodeRegistry.Builder codeRegistryBuilder,
                                            TypeDefinitionRegistry typeRegistry) {

    Collection<DatabricksTable> tables =
            databricksService.getAllTables(databricksCatalog, false);

    for (DatabricksTable table : tables) {
        String fieldName = table.getName();  // e.g. "trade", "accounts", "address"

        // 1) Safely extend Query / Subscription
        ensureRootField(typeRegistry, "Query", fieldName, "Json");          // or Trade, etc.
        ensureRootField(typeRegistry, "Subscription", fieldName, "Json");   // optional

        // 2) Wire data fetchers as you already do
        FieldCoordinates queryCoords = FieldCoordinates.coordinates("Query", fieldName);
        codeRegistryBuilder.dataFetcher(queryCoords, env -> getFromDatabricks(table));

        FieldCoordinates subCoords = FieldCoordinates.coordinates("Subscription", fieldName);
        codeRegistryBuilder.dataFetcher(subCoords, env -> getAllDataFromTopic(table));
    }

    return codeRegistryBuilder;
}

/**
 * Add "fieldName: [elementTypeName]" to the given root type (Query/Subscription)
 * only if it does NOT already exist.
 */
private void ensureRootField(TypeDefinitionRegistry registry,
                             String rootTypeName,
                             String fieldName,
                             String elementTypeName) {

    registry.getType(rootTypeName).ifPresent(def -> {
        ObjectTypeDefinition root = (ObjectTypeDefinition) def;

        // --- check first ---
        boolean alreadyExists = root.getFieldDefinitions().stream()
                .anyMatch(fd -> fd.getName().equals(fieldName));

        if (alreadyExists) {
            // e.g. accounts, address, users – already defined in .graphqls
            return;   // do NOT add again → no "redefine field" error
        }

        // build field definition: fieldName: [elementTypeName]
        Type elementType = new TypeName(elementTypeName);   // "Json", "Trade", etc.
        Type listType   = new ListType(elementType);

        FieldDefinition newField = FieldDefinition.newFieldDefinition()
                .name(fieldName)
                .type(listType)
                .build();

        // transform existing root type to include the new field
        ObjectTypeDefinition updatedRoot = root.transform(builder ->
                builder.fieldDefinition(newField));

        registry.remove(root);
        registry.add(updatedRoot);
    });
}
```

### What this gives you

- If `accounts`, `address`, `users`, etc. are **already defined** in your static `.graphqls`, this helper sees them and **skips** adding them again → no redefinition errors.
    
- New things like `trade` (which are not in SDL) **do get added** and wired to your DataFetchers.
    
- Bronze can be turned on without crashing, because you’re no longer blindly redefining existing fields.
    

So yes: **“check before adding” is the right fix**, and the snippet above is the GraphQL-java/DGS way to do it.