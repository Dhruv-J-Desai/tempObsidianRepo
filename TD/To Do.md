- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Perfect — thanks for the screenshots. Now I can be **very precise** and tie this directly to _your_ codebase.

### Key observation from your screenshots

You are **NOT** manually building Kafka configs.  
You are relying on **Spring Boot auto-configuration** via:

```yaml
spring:
  kafka:
    consumer:
      properties:
        ssl.truststore.location: classpath:certs/cacerts-updated
```

and you only **log** properties using:

- `KafkaPropertiesLogger`
    
- `ActivePropertiesLogger`
    

➡️ That means **Kafka client properties are finalized internally by Spring**, not by you.

So the fix must happen **before Kafka client is created**, without introducing a `Map<String,Object>` in your app code.

---

## ✅ The CORRECT hook for your setup (this is the missing piece)

### Use `KafkaPropertiesCustomizer`

This is _exactly_ what Spring Boot provides for this scenario.

You add **one bean**, and Spring will:

1. Load `application.yml`
    
2. Build Kafka properties
    
3. Call your customizer
    
4. Then create Kafka consumers/producers
    

---

## 🔧 Drop-in Fix (minimal, production-grade)

Create **one new class** (or put in existing config package):

```java
@Configuration
public class KafkaTruststoreClasspathResolver {

    @Bean
    KafkaPropertiesCustomizer truststoreCustomizer(ResourceLoader resourceLoader) {
        return props -> {
            Object locObj = props.get("ssl.truststore.location");
            if (locObj == null) {
                return;
            }

            String location = locObj.toString();

            // Only handle classpath:
            if (!location.startsWith("classpath:")) {
                return;
            }

            try {
                Resource resource = resourceLoader.getResource(location);

                Path tmp = Files.createTempFile("kafka-truststore-", ".jks");
                try (InputStream in = resource.getInputStream()) {
                    Files.copy(in, tmp, StandardCopyOption.REPLACE_EXISTING);
                }

                tmp.toFile().deleteOnExit();

                props.put("ssl.truststore.location", tmp.toAbsolutePath().toString());

                System.out.println(
                    "Resolved Kafka truststore from " + location +
                    " -> " + tmp.toAbsolutePath()
                );

            } catch (Exception e) {
                throw new IllegalStateException(
                    "Failed to resolve Kafka truststore from " + location, e
                );
            }
        };
    }
}
```

---

## ✅ Why this fits YOUR code perfectly

✔ You **do not** need:

- `Map<String,Object>`
    
- Custom `ConsumerFactory`
    
- Custom `ProducerFactory`
    

✔ You **keep** your existing YAML:

```yaml
ssl.truststore.location: classpath:certs/cacerts-updated
```

✔ Works in:

- IntelliJ
    
- `java -jar`
    
- Docker
    
- WEnix (Linux)
    
- Windows
    

✔ Kafka **never** sees `classpath:`  
It only sees something like:

```
/tmp/kafka-truststore-1837462819.jks
```

---

## 🔍 Why your current loggers show the problem but don’t fix it

### `ActivePropertiesLogger`

Just prints what Spring loaded  
➡️ Still shows `classpath:certs/cacerts-updated`

### `KafkaPropertiesLogger`

Prints values _before Kafka client normalization_  
➡️ Still shows invalid path

🚨 Kafka itself later does:

```java
Paths.get("classpath:certs/cacerts-updated")
```

💥 `InvalidPathException`

---

## 🧠 Mental model (important)

|Layer|Understands `classpath:`?|
|---|---|
|Spring Environment|✅ YES|
|ResourceLoader|✅ YES|
|Kafka Client|❌ NO|
|JVM NIO Paths|❌ NO|

So we **translate** at the boundary.

---

## ⭐ Recommended final YAML (SDK-friendly)

```yaml
spring:
  kafka:
    consumer:
      properties:
        ssl.truststore.location: ${SSL_TRUSTSTORE_LOCATION:classpath:certs/cacerts-updated}
        ssl.truststore.password: ${SSL_TRUSTSTORE_PASSWORD:changeit}
        ssl.truststore.type: JKS
```

Then:

- Local → classpath
    
- Docker → mount `/opt/app/certs/cacerts-updated` and set env var
    

---

## If you want next

I can:

- Refactor this into a **reusable SDK module**
    
- Add **unit test** validating classpath → temp resolution
    
- Add **Confluence-ready explanation** (1 page)
    
- Show how to **disable it for producers only / consumers only**
    

You’re on the right track — this is a _classic_ enterprise Kafka + Docker issue, and your setup is now cleanly fixable.