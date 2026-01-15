Yes — **that’s actually better for interviews** 👍  
Reading a file makes it feel **real**, tests fundamentals, and still stays simple.

Below is a **clean, interview-ready Spark coding exercise** that starts from **reading a file**, then performs transformations.

---

# 🔥 Spark Interview Coding Question (File-based)

## Scenario

You are given a **CSV file** containing transaction data stored in a data lake.

### File: `transactions.csv`

```csv
transaction_id,customer_id,country,amount,event_time
t1,c1,US,120.5,2024-01-01T10:15:00
t2,c2,IN,50.0,2024-01-01T10:20:00
t3,,US,75.0,2024-01-01T11:00:00
t4,c3,US,-10.0,2024-01-01T11:05:00
t5,c4,FR,200.0,2024-01-02T09:00:00
t6,c5,IN,300.0,2024-01-02T09:30:00
```

---

## Task

Write **PySpark code** to:

### 1️⃣ Read the file

- Read the CSV file into a Spark DataFrame
    
- Infer schema
    
- Parse `event_time` as timestamp
    

### 2️⃣ Clean the data

Remove rows where:

- `amount` is null or `<= 0`
    
- `customer_id` is null
    

### 3️⃣ Aggregate

- Compute **total transaction amount per country per day**
    

### 4️⃣ Rank

- For each day, return **top 2 countries** by total amount
    

---

## Expected Output Schema

```text
date           : date
country        : string
total_amount  : double
rank           : int
```

---

## Constraints

- ❌ No Pandas
    
- ❌ No Python UDFs
    
- ✅ Use Spark SQL / DataFrame API
    
- Assume the file is **large**
    

---

# ✅ Model Solution (PySpark)

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F
from pyspark.sql.window import Window

spark = SparkSession.builder.appName("spark-interview").getOrCreate()

# 1️⃣ Read file
df = (
    spark.read
    .option("header", True)
    .option("inferSchema", True)
    .csv("transactions.csv")
)

# 2️⃣ Clean data
clean_df = (
    df
    .withColumn("event_time", F.to_timestamp("event_time"))
    .filter(
        (F.col("amount").isNotNull()) &
        (F.col("amount") > 0) &
        (F.col("customer_id").isNotNull())
    )
    .withColumn("date", F.to_date("event_time"))
)

# 3️⃣ Aggregate
agg_df = (
    clean_df
    .groupBy("date", "country")
    .agg(F.sum("amount").alias("total_amount"))
)

# 4️⃣ Rank
window_spec = Window.partitionBy("date").orderBy(F.col("total_amount").desc())

result = (
    agg_df
    .withColumn("rank", F.dense_rank().over(window_spec))
    .filter(F.col("rank") <= 2)
    .select("date", "country", "total_amount", "rank")
)

result.show()
```

---

# 🎯 What This Tests (Interview Signal)

|Skill|Tested|
|---|---|
|Spark I/O|`spark.read.csv`|
|Schema handling|timestamp parsing|
|Data cleaning|filters|
|Aggregations|`groupBy`, `sum`|
|Window functions|`dense_rank`|
|Performance awareness|shuffle points|

---

# 💬 Follow-up Questions (Very Important)

Ask **any 2**:

1. Where do shuffles happen in this job?
    
2. How would you optimize if `country` is highly skewed?
    
3. How would this change if the file was **Parquet** instead of CSV?
    
4. What changes if this data arrives as a **stream**?
    

---

# 🧠 Senior Signal Answers (What to Listen For)

- “CSV is row-based, Parquet is columnar”
    
- “GroupBy causes shuffle”
    
- “Window also causes shuffle”
    
- “Broadcast or AQE for skew”
    
- “Streaming would need watermark + checkpoint”
    

---

# Variants You Can Swap In

- Read **Parquet** instead of CSV
    
- Use **JSON** with nested fields
    
- Add **deduplication**
    
- Convert to **Delta Lake** write
    

If you want, I can:

- Turn this into a **take-home Spark test**
    
- Add a **grading rubric (1–5)**
    
- Create a **streaming version**
    
- Align it specifically to **Databricks runtime**
    

Just tell me 👍