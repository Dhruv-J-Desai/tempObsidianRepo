- SpEL
- One Script to run all the programs
- Move the upsert logic from Angular to DeepHaven
- Look at Metrics View
- Try to tune the performance of Streaming Ingestion SDK while doing SCD2-merge
Yes — you can (and should) **read Parquet in a streaming / chunked way** so you don’t load the whole file into memory. The exact “best” way depends on what you mean by “streaming” (Kafka streaming? Spark structured streaming? just “process big files safely”?), but in a plain Java app you have good options.

## What _not_ to do

- Don’t read Parquet into a full in-memory list / dataframe before processing.
    
- Don’t convert the whole file to JSON strings at once.
    

## Best options in a Java app (low memory)

### Option A: Use Parquet “record reader” (row-by-row / batch-by-batch)

Parquet is columnar, but you can still iterate records without materializing everything.

Typical approach:

- Open Parquet file with a reader
    
- Read records in a loop
    
- Process each record (or a small batch) and immediately write downstream (Kafka/DB/etc.)
    

In the Hadoop/Parquet stack, you’ll see patterns like:

- `ParquetReader<GenericRecord>` (Avro)
    
- or `ParquetReader<Group>` (parquet-example)
    

This keeps memory bounded (only a small buffer).

### Option B: Use Spark (best if you already have it)

If “streaming” is part of Databricks/Spark:

- Use Spark to read Parquet with partition pruning + predicate pushdown
    
- Write out in micro-batches
    
- Spark manages memory and spilling
    

But if you’re _not_ running Spark, Option A is simpler.

### Option C: If the file is on S3/ADLS — still stream-ish

You typically can’t “HTTP stream” Parquet row-by-row easily because Parquet needs footer metadata and does random access.  
However you can still process in a bounded-memory way by:

- Downloading to local temp (or using a filesystem client that supports seeking)
    
- Using ParquetReader in batches
    

## Practical tactics to avoid OOM

- Process **one record at a time** or **small batches** (e.g., 1k–10k rows), flush downstream, clear references.
    
- If you convert to JSON, do it per record/batch and stream output (don’t build a giant string).
    
- Avoid collecting large lists; write continuously.
    
- Use filters early (only columns you need).
    

## One key gotcha: Parquet needs “seekable” input

Parquet reading usually requires a **seekable input stream** (random access) because it reads the footer and row groups.  
So “true streaming over a non-seekable InputStream” is limited.  
But “streaming processing” in the sense of **iterating row groups** with bounded memory is absolutely doable.

---

If you tell me:

1. where the Parquet file lives (local disk? ADLS? S3?),
    
2. what you want to do with each row (send to Kafka? write to DB? transform to JSON?),  
    I’ll give you the best concrete Java approach + a skeleton loop that processes row-by-row without OOM.