- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge

Got it — you **can** keep `ssl.truststore.location: classpath:...` and make it work **without** `KafkaPropertiesCustomizer`.

The trick is: **don’t let Kafka see `classpath:`**. You resolve it yourself **when building the ConsumerFactory/ProducerFactory** (i.e., you take control of Kafka factories).

---

## 1) Keep your `application.yml` (classpath stays)

```yaml
spring:
  kafka:
    consumer:
      properties:
        ssl.truststore.location: classpath:certs/cacerts-updated
        ssl.truststore.password: changeit
        ssl.truststore.type: JKS
```

(If you also produce, same under `spring.kafka.producer.properties`.)

---

## 2) Add a resolver utility (classpath → temp file path)

```java
import org.springframework.core.io.Resource;
import org.springframework.core.io.ResourceLoader;

import java.io.InputStream;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;

public final class ClasspathToTempFile {
    private ClasspathToTempFile() {}

    public static String resolveIfClasspath(String location, ResourceLoader resourceLoader) {
        if (location == null) return null;
        if (!location.startsWith("classpath:")) return location;

        try {
            Resource resource = resourceLoader.getResource(location);

            // If IDE/exploded classes -> can become a real file already
            if (resource.isFile()) {
                return resource.getFile().getAbsolutePath();
            }

            // Fat JAR -> copy out to temp
            Path tmp = Files.createTempFile("kafka-truststore-", ".jks");
            try (InputStream in = resource.getInputStream()) {
                Files.copy(in, tmp, StandardCopyOption.REPLACE_EXISTING);
            }
            tmp.toFile().deleteOnExit();
            return tmp.toAbsolutePath().toString();
        } catch (Exception e) {
            throw new IllegalStateException("Failed to resolve truststore from " + location, e);
        }
    }
}
```

---

## 3) Override Kafka’s ConsumerFactory (and ProducerFactory if needed)

Create a config class:

```java
import org.springframework.boot.autoconfigure.kafka.KafkaProperties;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.core.io.ResourceLoader;
import org.springframework.kafka.core.*;

import java.util.Map;

@Configuration
public class KafkaFactoryConfig {

    @Bean
    public ConsumerFactory<Object, Object> consumerFactory(
            KafkaProperties kafkaProperties,
            ResourceLoader resourceLoader
    ) {
        Map<String, Object> props = kafkaProperties.buildConsumerProperties(null);

        String loc = (String) props.get("ssl.truststore.location");
        String resolved = ClasspathToTempFile.resolveIfClasspath(loc, resourceLoader);
        if (resolved != null) {
            props.put("ssl.truststore.location", resolved);
        }

        return new DefaultKafkaConsumerFactory<>(props);
    }

    // Only if you also produce with SSL:
    @Bean
    public ProducerFactory<Object, Object> producerFactory(
            KafkaProperties kafkaProperties,
            ResourceLoader resourceLoader
    ) {
        Map<String, Object> props = kafkaProperties.buildProducerProperties(null);

        String loc = (String) props.get("ssl.truststore.location");
        String resolved = ClasspathToTempFile.resolveIfClasspath(loc, resourceLoader);
        if (resolved != null) {
            props.put("ssl.truststore.location", resolved);
        }

        return new DefaultKafkaProducerFactory<>(props);
    }

    @Bean
    public KafkaTemplate<Object, Object> kafkaTemplate(ProducerFactory<Object, Object> pf) {
        return new KafkaTemplate<>(pf);
    }
}
```

✅ Now Kafka only ever receives a real path like `C:\Users\...\AppData\Local\Temp\kafka-truststore-xyz.jks` (Windows) or `/tmp/...` (Docker/Linux).  
✅ You keep **classpath** in YAML.  
✅ No `KafkaPropertiesCustomizer`.

---

### Important note

If you already have a custom `ConcurrentKafkaListenerContainerFactory`, make sure it uses this `consumerFactory()` bean (Spring will auto-wire it if bean names/types match).

---

If you show me whether you have any existing Kafka config class (or if you only rely on `@KafkaListener` defaults), I’ll tell you exactly where to paste this so it doesn’t conflict with anything else.