```yml
server:  
  port: 8080  
  
spring:  
  
  kafka:  
    bootstrap-servers: localhost:9092  
  
    # Producer configuration  
    producer:  
      key-serializer: org.apache.kafka.common.serialization.StringSerializer  
      value-serializer: org.apache.kafka.common.serialization.StringSerializer  
      acks: all  
      retries: 10  
      properties:  
        enable.idempotence: true  
        linger.ms: 5  
        batch.size: 32768  
  
    # Consumer baseline (deserializers only; group/topic set via annotation)  
    consumer:  
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer  
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer  
      properties:  
        isolation.level: read_committed  
  
# Enterprise starter knobs (from your SDK)  
enterprise:  
  kafka:  
    consumer:  
      enable-auto-commit: false  
      auto-offset-reset: earliest  
      max-poll-records: 500  
      poll-timeout-ms: 1500  
      backoff-delay-ms: 1000  
      backoff-max-retries: 3  
  databricks:  
    host: adb-2900282841250571.11.azuredatabricks.net  
    http-path: /sql/1.0/warehouses/bbb98e34c8ca882c  
    token: dapi60be1890ad3a87888f279f5c4c71ca41  
    catalog: kafka_data  
    schema: bronze  
    table: client_trade_events  
    mode: INSERT #INSERT OR MERGE  
    merge-keys: [ "client_id", "symbol" ]  
    update-allowlist: [ "direction", "quantity", "price", "valid_from" ]  
    batch-max-size: 1000  
    batch-flush-ms: 2000  
  
management:  
  endpoints:  
    web:  
      exposure:  
        include: health,info,beans,env  
logging:  
  level:  
    root: INFO  
    org.springframework.kafka: INFO  
    com.acme: INFO  
    org.springframework.boot.autoconfigure: WARN
```