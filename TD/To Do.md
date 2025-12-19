




---------------

Yep — you can totally do it **without the Hadoop Azure connector** by doing a 2-step flow:

1. **Download Parquet from ADLS to a local temp file (streaming download, no OOM)**
    
2. **Stream-read the local Parquet file row-by-row with `AvroParquetReader`**, convert each row to JSON, and **write to DB in micro-batches**.
    

That’s a clean Spring Boot approach.

Below is a complete, practical skeleton you can drop in.

---

## 1) Dependencies (Maven)

You need:

- Azure Storage Blob SDK (to download)
    
- Parquet Avro reader
    
- Avro
    
- Jackson
    
- DB driver (example shown with Spring JDBC)
    

```xml
<dependencies>
  <!-- Azure Blob SDK (works for ADLS Gen2 when using the Blob endpoint) -->
  <dependency>
    <groupId>com.azure</groupId>
    <artifactId>azure-storage-blob</artifactId>
    <version>12.28.1</version>
  </dependency>

  <!-- Parquet + Avro -->
  <dependency>
    <groupId>org.apache.parquet</groupId>
    <artifactId>parquet-avro</artifactId>
    <version>1.14.3</version>
  </dependency>
  <dependency>
    <groupId>org.apache.avro</groupId>
    <artifactId>avro</artifactId>
    <version>1.11.3</version>
  </dependency>

  <!-- Jackson -->
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>

  <!-- Spring JDBC -->
  <dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jdbc</artifactId>
  </dependency>

  <!-- Your DB driver example: Postgres -->
  <dependency>
    <groupId>org.postgresql</groupId>
    <artifactId>postgresql</artifactId>
    <scope>runtime</scope>
  </dependency>
</dependencies>
```

> Note: `parquet-avro` pulls Hadoop-ish transitive dependencies, but you are **not configuring** Hadoop Azure FS. You’re reading from a **local file**, so that part stays unused.

---

## 2) Download Parquet from ADLS to a temp file (streaming)

This uses Azure Blob SDK. For ADLS Gen2, you typically access via the **Blob endpoint**:  
`https://<account>.blob.core.windows.net/<container>/<path>`

You can auth via:

- **connection string**
    
- **SAS URL**
    
- **service principal** (requires `azure-identity`, if you want that later)
    

### Download code

```java
import com.azure.storage.blob.BlobClient;
import com.azure.storage.blob.BlobClientBuilder;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;

public class AdlsDownloader {

  public static Path downloadToTempFile(String blobUrl, String sasToken) throws IOException {
    // Example blobUrl: https://acct.blob.core.windows.net/container/folder/file.parquet
    BlobClient client = new BlobClientBuilder()
        .endpoint(blobUrl)
        .sasToken(sasToken) // or omit if blobUrl already includes SAS
        .buildClient();

    Path tmp = Files.createTempFile("adls-", ".parquet");
    // Streams to disk (doesn't load whole file into memory)
    client.downloadToFile(tmp.toString(), true);
    return tmp;
  }
}
```

**If you already have a SAS URL** (blobUrl includes `?sv=...`), you can just pass it directly with `.endpoint(sasUrl)` and skip `.sasToken(...)`.

---

## 3) Stream-read Parquet locally using `AvroParquetReader` + micro-batch DB writes

### Key design points

- Read records one by one (`reader.read()` loop)
    
- Convert to JSON per row
    
- Accumulate JSON strings into a `List<String>` until `batchSize`
    
- Use `JdbcTemplate.batchUpdate()` to insert in one go
    
- Clear and continue
    

### Implementation

```java
import com.fasterxml.jackson.databind.ObjectMapper;
import org.apache.avro.Schema;
import org.apache.avro.generic.GenericRecord;
import org.apache.parquet.avro.AvroParquetReader;
import org.apache.parquet.hadoop.ParquetReader;
import org.apache.hadoop.fs.Path;
import org.springframework.jdbc.core.JdbcTemplate;

import java.io.Closeable;
import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.attribute.FileTime;
import java.util.*;

public class ParquetToDbIngestor {

  private final JdbcTemplate jdbcTemplate;
  private final ObjectMapper objectMapper;

  public ParquetToDbIngestor(JdbcTemplate jdbcTemplate, ObjectMapper objectMapper) {
    this.jdbcTemplate = jdbcTemplate;
    this.objectMapper = objectMapper;
  }

  public void ingestLocalParquetToDb(java.nio.file.Path parquetFile,
                                     int batchSize) throws IOException {

    List<String> batch = new ArrayList<>(batchSize);

    // ParquetReader expects Hadoop Path; local file is fine: "file:/..."
    Path hadoopPath = new Path(parquetFile.toUri());

    try (ParquetReader<GenericRecord> reader = AvroParquetReader.<GenericRecord>builder(hadoopPath).build()) {

      GenericRecord record;
      long total = 0;

      while ((record = reader.read()) != null) {
        String json = genericRecordToJson(record);
        batch.add(json);
        total++;

        if (batch.size() >= batchSize) {
          flushBatch(batch);
        }
      }

      // flush remaining
      if (!batch.isEmpty()) flushBatch(batch);

      System.out.println("Done. Total rows processed: " + total);
    }
  }

  private String genericRecordToJson(GenericRecord record) throws IOException {
    // Convert GenericRecord -> Map so JSON is clean and doesn't include Avro internals
    Map<String, Object> map = new LinkedHashMap<>();
    Schema schema = record.getSchema();

    for (Schema.Field f : schema.getFields()) {
      Object val = record.get(f.name());
      map.put(f.name(), normalizeAvroValue(val));
    }

    return objectMapper.writeValueAsString(map);
  }

  private Object normalizeAvroValue(Object val) {
    if (val == null) return null;

    // Avro Utf8 -> String
    if (val instanceof org.apache.avro.util.Utf8) return val.toString();

    // Arrays / lists
    if (val instanceof Collection<?> c) {
      List<Object> out = new ArrayList<>(c.size());
      for (Object o : c) out.add(normalizeAvroValue(o));
      return out;
    }

    // Maps
    if (val instanceof Map<?, ?> m) {
      Map<String, Object> out = new LinkedHashMap<>();
      for (Map.Entry<?, ?> e : m.entrySet()) {
        out.put(String.valueOf(e.getKey()), normalizeAvroValue(e.getValue()));
      }
      return out;
    }

    // Nested records
    if (val instanceof GenericRecord gr) {
      Map<String, Object> out = new LinkedHashMap<>();
      for (Schema.Field f : gr.getSchema().getFields()) {
        out.put(f.name(), normalizeAvroValue(gr.get(f.name())));
      }
      return out;
    }

    // ByteBuffer (often used for bytes/logical types)
    if (val instanceof java.nio.ByteBuffer bb) {
      byte[] bytes = new byte[bb.remaining()];
      bb.slice().get(bytes);
      // store as base64 to be JSON-safe
      return Base64.getEncoder().encodeToString(bytes);
    }

    // Everything else: primitives, numbers, booleans, etc.
    return val;
  }

  private void flushBatch(List<String> batch) {
    // Example: store JSON in a JSON column or TEXT column
    // Adjust SQL for your DB (Postgres JSONB: cast ?::jsonb)
    String sql = "INSERT INTO parquet_events(payload_json) VALUES (?)";

    jdbcTemplate.batchUpdate(
        sql,
        batch,
        batch.size(),
        (ps, json) -> ps.setString(1, json)
    );

    batch.clear();
  }
}
```

---

## 4) Wiring it in Spring Boot (simple service method)

Example “orchestration” method:

```java
import org.springframework.stereotype.Service;

import java.nio.file.Files;
import java.nio.file.Path;

@Service
public class ParquetIngestionService {

  private final ParquetToDbIngestor ingestor;

  public ParquetIngestionService(ParquetToDbIngestor ingestor) {
    this.ingestor = ingestor;
  }

  public void ingestFromAdls(String blobUrl, String sasToken) throws Exception {
    Path tmp = null;
    try {
      tmp = AdlsDownloader.downloadToTempFile(blobUrl, sasToken);
      ingestor.ingestLocalParquetToDb(tmp, 1000); // micro-batch size
    } finally {
      if (tmp != null) Files.deleteIfExists(tmp);
    }
  }
}
```

---

## Notes / gotchas you should know

### 1) This is still “streaming enough” for OOM avoidance

- Download step streams to disk.
    
- Parquet reader streams rows; you only hold `batchSize` rows in memory.
    

### 2) Temp disk space

You _will_ need enough local disk to hold the Parquet file. If files are huge, consider:

- placing temp file on a mounted volume with space
    
- or streaming from ADLS FS directly later (that would need connector)
    

### 3) Performance tips

- Increase DB batch size (1000–5000)
    
- Use DB-side optimizations (indexes, constraints) carefully during load
    
- For Postgres JSONB inserts, you may want: `VALUES (?::jsonb)` depending on your column type
    

### 4) Complex Parquet schemas

Nested structs/arrays/maps work with the recursive normalization above.  
Logical types (timestamps/decimals) might come through in formats you want to standardize. If you tell me your schema/logical types, I can add clean conversions.

---

If you tell me your DB type (Postgres / SQL Server / MySQL) and your ADLS auth method (SAS vs service principal), I’ll tailor:

- the exact Azure client creation
    
- the best batch insert SQL (including JSON/JSONB casting)