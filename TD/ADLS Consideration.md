Here is the **cleanest and production-ready** way to generate **multiple fake events using Java Faker** and send them to Kafka using your new:

```java
KafkaTemplate<String, TradeEvent>
```

setup.

---

# ✅ 1. Add Java Faker Dependency

If not already added:

```xml
<dependency>
    <groupId>com.github.javafaker</groupId>
    <artifactId>javafaker</artifactId>
    <version>1.0.2</version>
</dependency>
```

---

# ✅ 2. Create a Faker-based TradeEvent Generator

Create a utility/service that generates a single fake TradeEvent.

### Example TradeEvent model (your fields may vary):

```java
public record TradeEvent(
        String clientId,
        String symbol,
        double quantity,
        double price,
        String direction,
        String ingestTime,
        String validFrom,
        String validTo,
        boolean isCurrent
) {}
```

### Create Faker generator:

```java
import com.github.javafaker.Faker;

@Service
public class FakeTradeEventFactory {

    private final Faker faker = new Faker();

    public TradeEvent generate() {
        String[] symbols = {"TSLA", "NVDA", "AAPL", "AVGO", "MSFT"};
        String[] directions = {"BUY", "SELL"};

        String symbol = symbols[faker.number().numberBetween(0, symbols.length)];
        String direction = directions[faker.number().numberBetween(0, directions.length)];

        return new TradeEvent(
                "C00" + faker.number().numberBetween(1, 9),
                symbol,
                faker.number().randomDouble(0, 50, 500),   // quantity
                faker.number().randomDouble(2, 100, 500),  // price
                direction,
                Instant.now().toString(),
                Instant.now().toString(),
                "9999-12-31T23:59:59.000Z",
                true
        );
    }
}
```

---

# ✅ 3. Modify Producer to Send Multiple Events

You can now inject your faker factory and send as many events as required.

```java
@Service
@Slf4j
public class TradeEventProducerService {

    private static final String TOPIC = "ccd01_sb_its_esp_tap3507_trade";

    private final KafkaTemplate<String, TradeEvent> kafkaTemplate;
    private final FakeTradeEventFactory faker;

    public TradeEventProducerService(KafkaTemplate<String, TradeEvent> kafkaTemplate,
                                     FakeTradeEventFactory faker) {
        this.kafkaTemplate = kafkaTemplate;
        this.faker = faker;
    }

    public void send(TradeEvent event) {
        String key = event.clientId() + "-" + event.symbol();
        kafkaTemplate.send(TOPIC, key, event);
    }

    /** Send N fake events */
    public void sendMultiple(int count) {
        for (int i = 0; i < count; i++) {
            TradeEvent event = faker.generate();
            send(event);
            log.info("Produced fake event: {}", event);
        }
    }
}
```

---

# ✅ 4. Controller Endpoint (Optional)

Expose an endpoint to test:

```java
@RestController
@RequestMapping("/api/producer")
public class ProducerController {

    private final TradeEventProducerService producer;

    public ProducerController(TradeEventProducerService producer) {
        this.producer = producer;
    }

    @PostMapping("/generate/{count}")
    public String generate(@PathVariable int count) {
        producer.sendMultiple(count);
        return "Produced " + count + " events";
    }
}
```

Now call:

```
POST http://localhost:8080/api/producer/generate/10
```

→ Produces **10 fake trade events** to Kafka.

---

# 🎉 Result

- Fake data is generated using Java Faker.
    
- Each event is a proper **TradeEvent POJO**.
    
- KafkaTemplate with **JsonSerializer** serializes it to JSON.
    
- DeepHaven & Angular automatically pick up the new events.
    

---

If you want **burst-mode**, **random delays**, or **infinite continuous streaming**, I can add that too—just tell me!