```java
package com.example.kafka_producer_consumer.producer.controller;  
  
import com.example.kafka_producer_consumer.producer.model.TradeEvent;  
import com.example.kafka_producer_consumer.producer.service.TradeEventProducer;  
import org.springframework.http.ResponseEntity;  
import org.springframework.web.bind.annotation.PostMapping;  
import org.springframework.web.bind.annotation.RequestBody;  
import org.springframework.web.bind.annotation.RequestMapping;  
import org.springframework.web.bind.annotation.RestController;  
  
import java.time.Instant;  
import java.util.Map;  
import java.util.UUID;  
  
@RestController  
@RequestMapping("/produce")  
public class ProduceController {  
    private final TradeEventProducer producer;  
  
    public ProduceController(TradeEventProducer producer) {  
        this.producer = producer;  
    }  
  
    @PostMapping  
    public ResponseEntity<Map<String, Object>> produce(@RequestBody TradeEvent input) {  
        TradeEvent event = (input.validFrom() != null)  
                ? input  
                : new TradeEvent(  
                input.clientId(),  
                input.symbol(),  
                input.direction(),  
                input.quantity(),  
                input.price(),  
                Instant.now()  
        );  
  
        producer.send(event);  
  
        return ResponseEntity.ok(Map.of(  
                "status", "SENT",  
                "clientId", event.clientId(),  
                "symbol", event.symbol(),  
                "direction", event.direction(),  
                "quantity", event.quantity(),  
                "price", event.price(),  
                "validFrom", event.validFrom().toString()  
        ));  
    }  
}
```