```java
package com.example.kafka_producer_consumer.consumer;  
  
import com.example.enterprise_kafka_starter.annotation.EnterpriseKafkaListener;  
import com.example.enterprise_kafka_starter.jdbc.WarehouseIngestor;  
import org.apache.kafka.clients.consumer.ConsumerRecord;  
import org.slf4j.Logger;  
import org.slf4j.LoggerFactory;  
import org.springframework.beans.factory.annotation.Qualifier;  
import org.springframework.kafka.support.Acknowledgment;  
import org.springframework.stereotype.Component;  
  
@Component  
public class EventsConsumer {  
  
    private static final Logger log = LoggerFactory.getLogger(EventsConsumer.class);  
    private final WarehouseIngestor ingestor;  
  
    public EventsConsumer(@Qualifier("warehouseIngestor") WarehouseIngestor ingestor) {  
        this.ingestor = ingestor;  
    }  
  
    @EnterpriseKafkaListener(  
            topics = "trade-events",  
            groupId = "tcen-consumers",  
            concurrency = "2"  
    )  
    public void onMessage(ConsumerRecord<String, String> record, Acknowledgment ack) {  
        try {  
            log.info("Consume: partition={}, offset={}, key={}, payload={}",  
                    record.partition(), record.offset(), record.key(), record.value());  
            ingestor.enqueueJson(record.value());  
            ack.acknowledge();  
        } catch (Exception e) {  
            log.error("Processing failed, delegating to error handler", e);  
            throw e;  
        }  
    }  
}
```