```java
package com.example.kafka_producer_consumer.producer.model;  
  
import java.time.Instant;  
  
public record TradeEvent(  
        String clientId,  
        String symbol,  
        String direction,  
        Integer quantity,  
        double price,  
        Instant validFrom  
) {  
}
```