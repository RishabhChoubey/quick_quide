# Big Data & Streaming Systems Guide

## Table of Contents

1. [Big Data Fundamentals](#big-data-fundamentals)
2. [Hadoop Ecosystem](#hadoop-ecosystem)
3. [Apache Spark](#apache-spark)
4. [Stream Processing](#stream-processing)
5. [Data Lake Architecture](#data-lake-architecture)
6. [Data Warehouse vs Data Lake](#data-warehouse-vs-data-lake)
7. [Lambda & Kappa Architecture](#lambda--kappa-architecture)
8. [Real-time Analytics](#real-time-analytics)
9. [Big Data Interview Questions](#big-data-interview-questions)

---

## Big Data Fundamentals

### The 5 V's of Big Data

1. **Volume**: Terabytes to Petabytes of data
2. **Velocity**: Speed of data generation and processing
3. **Variety**: Structured, semi-structured, unstructured
4. **Veracity**: Data quality and trustworthiness
5. **Value**: Business insights from data

### CAP Theorem for Big Data

```
Consistency vs Availability vs Partition Tolerance
- Choose 2 out of 3
- Most big data systems choose AP (Availability + Partition Tolerance)
```

---

## Hadoop Ecosystem

### HDFS (Hadoop Distributed File System)

**Architecture:**
```
┌──────────────┐
│  NameNode    │  (Master - stores metadata)
│  (Metadata)  │
└──────┬───────┘
       │
   ┌───┴────┬─────────┬─────────┐
   │        │         │         │
┌──▼──┐  ┌──▼──┐  ┌──▼──┐  ┌──▼──┐
│DN 1 │  │DN 2 │  │DN 3 │  │DN 4 │  (DataNodes - store actual data)
└─────┘  └─────┘  └─────┘  └─────┘
```

**Key Features:**
- Block-based storage (default 128MB)
- Replication factor (default 3)
- Rack awareness
- Write-once, read-many

```python
from hdfs import InsecureClient

class HDFSManager:
    def __init__(self, namenode_url='http://namenode:50070'):
        self.client = InsecureClient(namenode_url, user='hadoop')
    
    def upload_file(self, local_path, hdfs_path):
        """Upload file to HDFS"""
        self.client.upload(hdfs_path, local_path)
        print(f"Uploaded {local_path} to {hdfs_path}")
    
    def read_file(self, hdfs_path):
        """Read file from HDFS"""
        with self.client.read(hdfs_path) as reader:
            return reader.read()
    
    def list_directory(self, hdfs_path):
        """List files in HDFS directory"""
        return self.client.list(hdfs_path)
    
    def delete_file(self, hdfs_path):
        """Delete file from HDFS"""
        self.client.delete(hdfs_path)

# Usage
hdfs = HDFSManager()
hdfs.upload_file('/local/data.csv', '/user/hadoop/data.csv')
files = hdfs.list_directory('/user/hadoop/')
```

### MapReduce

**Concept:**
```
Input Data → Map Phase → Shuffle & Sort → Reduce Phase → Output
```

**Word Count Example:**

```python
from mrjob.job import MRJob
import re

class WordCount(MRJob):
    
    def mapper(self, _, line):
        """Map: emit (word, 1) for each word"""
        words = re.findall(r'\w+', line.lower())
        for word in words:
            yield word, 1
    
    def reducer(self, key, values):
        """Reduce: sum counts for each word"""
        yield key, sum(values)

if __name__ == '__main__':
    WordCount.run()

# Run: python wordcount.py input.txt
```

**MapReduce Job Example (Java):**

```java
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

import java.io.IOException;
import java.util.StringTokenizer;

public class WordCount {
    
    // Mapper class
    public static class TokenizerMapper 
        extends Mapper<Object, Text, Text, IntWritable> {
        
        private final static IntWritable one = new IntWritable(1);
        private Text word = new Text();
        
        public void map(Object key, Text value, Context context) 
            throws IOException, InterruptedException {
            
            StringTokenizer itr = new StringTokenizer(value.toString());
            while (itr.hasMoreTokens()) {
                word.set(itr.nextToken());
                context.write(word, one);
            }
        }
    }
    
    // Reducer class
    public static class IntSumReducer 
        extends Reducer<Text, IntWritable, Text, IntWritable> {
        
        private IntWritable result = new IntWritable();
        
        public void reduce(Text key, Iterable<IntWritable> values, Context context) 
            throws IOException, InterruptedException {
            
            int sum = 0;
            for (IntWritable val : values) {
                sum += val.get();
            }
            result.set(sum);
            context.write(key, result);
        }
    }
    
    public static void main(String[] args) throws Exception {
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf, "word count");
        
        job.setJarByClass(WordCount.class);
        job.setMapperClass(TokenizerMapper.class);
        job.setCombinerClass(IntSumReducer.class);
        job.setReducerClass(IntSumReducer.class);
        
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        
        FileInputFormat.addInputPath(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        
        System.exit(job.waitForCompletion(true) ? 0 : 1);
    }
}
```

### YARN (Yet Another Resource Negotiator)

**Resource Management Architecture:**

```
┌────────────────────┐
│  Resource Manager  │  (Master)
└─────────┬──────────┘
          │
    ┌─────┴─────┬─────────┬─────────┐
    │           │         │         │
┌───▼───┐   ┌───▼───┐ ┌───▼───┐ ┌───▼───┐
│  NM1  │   │  NM2  │ │  NM3  │ │  NM4  │  (Node Managers)
└───────┘   └───────┘ └───────┘ └───────┘

Each Node Manager manages containers for applications
```

### Apache Hive

**SQL on Hadoop**

```sql
-- Create external table
CREATE EXTERNAL TABLE user_logs (
    user_id INT,
    action STRING,
    timestamp BIGINT,
    ip_address STRING
)
PARTITIONED BY (date STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/user_logs';

-- Add partition
ALTER TABLE user_logs ADD PARTITION (date='2024-01-15')
LOCATION '/data/logs/2024/01/15';

-- Query with aggregation
SELECT 
    date,
    COUNT(DISTINCT user_id) as unique_users,
    COUNT(*) as total_actions
FROM user_logs
WHERE date BETWEEN '2024-01-01' AND '2024-01-31'
GROUP BY date
ORDER BY date;

-- Join example
SELECT 
    u.user_id,
    u.name,
    COUNT(l.action) as action_count
FROM users u
JOIN user_logs l ON u.user_id = l.user_id
WHERE l.date = '2024-01-15'
GROUP BY u.user_id, u.name
HAVING action_count > 10;
```

**Optimizations:**

```sql
-- Bucketing for faster joins
CREATE TABLE user_logs_bucketed (
    user_id INT,
    action STRING,
    timestamp BIGINT
)
PARTITIONED BY (date STRING)
CLUSTERED BY (user_id) INTO 32 BUCKETS
STORED AS ORC;

-- Enable optimizations
SET hive.exec.dynamic.partition = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
SET hive.optimize.ppd = true;  -- Predicate pushdown
SET hive.vectorized.execution.enabled = true;
```

---

## Apache Spark

### Spark Architecture

```
┌─────────────┐
│   Driver    │  (SparkContext)
└──────┬──────┘
       │
       ├────────────────┬────────────────┐
       │                │                │
┌──────▼──────┐  ┌──────▼──────┐  ┌──────▼──────┐
│  Executor 1 │  │  Executor 2 │  │  Executor 3 │
│  (Tasks)    │  │  (Tasks)    │  │  (Tasks)    │
└─────────────┘  └─────────────┘  └─────────────┘
```

### RDD (Resilient Distributed Dataset)

```python
from pyspark import SparkContext

sc = SparkContext("local", "RDD Example")

# Create RDD
data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
rdd = sc.parallelize(data, numSlices=4)

# Transformations (lazy)
squared = rdd.map(lambda x: x * x)
filtered = squared.filter(lambda x: x > 20)
reduced = filtered.reduce(lambda a, b: a + b)

# Actions (trigger execution)
result = filtered.collect()
print(f"Filtered results: {result}")
print(f"Sum: {reduced}")

# Word count with RDD
text_rdd = sc.textFile("hdfs://namenode:9000/data/text.txt")
counts = text_rdd \
    .flatMap(lambda line: line.split(" ")) \
    .map(lambda word: (word, 1)) \
    .reduceByKey(lambda a, b: a + b)

counts.saveAsTextFile("hdfs://namenode:9000/output/wordcount")
```

### Spark DataFrames & SQL

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, count, avg, sum, max, min, window
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType

# Create Spark session
spark = SparkSession.builder \
    .appName("Spark DataFrame Example") \
    .config("spark.sql.warehouse.dir", "/user/hive/warehouse") \
    .enableHiveSupport() \
    .getOrCreate()

# Read data
df = spark.read \
    .format("parquet") \
    .load("/data/user_events.parquet")

# DataFrame operations
result = df \
    .filter(col("event_type") == "purchase") \
    .groupBy("user_id", "product_category") \
    .agg(
        count("*").alias("purchase_count"),
        sum("amount").alias("total_spent"),
        avg("amount").alias("avg_amount")
    ) \
    .orderBy(col("total_spent").desc()) \
    .limit(100)

result.show()

# SQL queries
df.createOrReplaceTempView("user_events")

sql_result = spark.sql("""
    SELECT 
        DATE(timestamp) as date,
        event_type,
        COUNT(*) as event_count,
        COUNT(DISTINCT user_id) as unique_users
    FROM user_events
    WHERE timestamp >= '2024-01-01'
    GROUP BY DATE(timestamp), event_type
    ORDER BY date, event_count DESC
""")

sql_result.show()

# Write results
result.write \
    .mode("overwrite") \
    .partitionBy("date") \
    .parquet("/output/user_purchases")
```

### Spark Streaming (Structured Streaming)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, window, from_json
from pyspark.sql.types import StructType, StructField, StringType, IntegerType, TimestampType

spark = SparkSession.builder \
    .appName("Structured Streaming") \
    .getOrCreate()

# Define schema
schema = StructType([
    StructField("user_id", IntegerType()),
    StructField("event_type", StringType()),
    StructField("timestamp", TimestampType()),
    StructField("amount", IntegerType())
])

# Read from Kafka
df = spark \
    .readStream \
    .format("kafka") \
    .option("kafka.bootstrap.servers", "kafka:9092") \
    .option("subscribe", "user-events") \
    .option("startingOffsets", "latest") \
    .load()

# Parse JSON
events = df.select(
    from_json(col("value").cast("string"), schema).alias("data")
).select("data.*")

# Windowed aggregation (5-minute windows)
windowed_counts = events \
    .groupBy(
        window(col("timestamp"), "5 minutes", "1 minute"),
        col("event_type")
    ) \
    .agg(
        count("*").alias("event_count"),
        sum("amount").alias("total_amount")
    )

# Write to console
query = windowed_counts \
    .writeStream \
    .outputMode("update") \
    .format("console") \
    .option("truncate", False) \
    .start()

# Write to HDFS
hdfs_query = windowed_counts \
    .writeStream \
    .outputMode("append") \
    .format("parquet") \
    .option("path", "/streaming/output") \
    .option("checkpointLocation", "/streaming/checkpoint") \
    .start()

query.awaitTermination()
```

### Spark Performance Optimization

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import broadcast

spark = SparkSession.builder \
    .appName("Optimized Spark Job") \
    .config("spark.sql.adaptive.enabled", "true") \
    .config("spark.sql.adaptive.coalescePartitions.enabled", "true") \
    .config("spark.sql.autoBroadcastJoinThreshold", "10485760") \
    .config("spark.executor.memory", "4g") \
    .config("spark.executor.cores", "4") \
    .config("spark.sql.shuffle.partitions", "200") \
    .getOrCreate()

# Optimization 1: Broadcast join for small tables
small_table = spark.read.parquet("/data/small_lookup.parquet")
large_table = spark.read.parquet("/data/large_events.parquet")

result = large_table.join(
    broadcast(small_table),
    "key"
)

# Optimization 2: Partitioning
df.repartition(200, "user_id") \
    .write \
    .partitionBy("date", "country") \
    .mode("overwrite") \
    .parquet("/output/partitioned_data")

# Optimization 3: Caching
cached_df = df.filter(col("status") == "active").cache()
cached_df.count()  # Materialize cache

# Optimization 4: Coalesce for reducing partitions
df.coalesce(10).write.parquet("/output/coalesced")
```

---

## Stream Processing

### Apache Kafka Streams

```java
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.StreamsConfig;
import org.apache.kafka.streams.kstream.*;

import java.time.Duration;
import java.util.Properties;

public class KafkaStreamsExample {
    
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "stream-processor");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "kafka:9092");
        
        StreamsBuilder builder = new StreamsBuilder();
        
        // Read from topic
        KStream<String, String> events = builder.stream("user-events");
        
        // Transformation: Filter and map
        KStream<String, Long> clickEvents = events
            .filter((key, value) -> value.contains("click"))
            .mapValues(value -> parseEventTimestamp(value));
        
        // Aggregation: Count clicks per user in 5-minute windows
        KTable<Windowed<String>, Long> clickCounts = clickEvents
            .groupByKey()
            .windowedBy(TimeWindows.of(Duration.ofMinutes(5)))
            .count();
        
        // Write to output topic
        clickCounts.toStream()
            .map((windowedKey, count) -> 
                new KeyValue<>(windowedKey.key(), count))
            .to("click-counts");
        
        // Start stream processing
        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
        
        // Shutdown hook
        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
    
    private static Long parseEventTimestamp(String event) {
        // Parse timestamp from event
        return System.currentTimeMillis();
    }
}
```

### Apache Flink

**Real-time Processing with Event Time**

```java
import org.apache.flink.streaming.api.datastream.DataStream;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.windowing.time.Time;
import org.apache.flink.streaming.api.windowing.assigners.TumblingEventTimeWindows;

public class FlinkStreamingExample {
    
    public static void main(String[] args) throws Exception {
        // Set up execution environment
        StreamExecutionEnvironment env = 
            StreamExecutionEnvironment.getExecutionEnvironment();
        
        // Enable checkpointing
        env.enableCheckpointing(60000);
        
        // Read from Kafka
        DataStream<Event> events = env
            .addSource(new FlinkKafkaConsumer<>(
                "user-events",
                new EventDeserializationSchema(),
                properties
            ))
            .assignTimestampsAndWatermarks(
                WatermarkStrategy
                    .<Event>forBoundedOutOfOrderness(Duration.ofSeconds(10))
                    .withTimestampAssigner((event, timestamp) -> event.getTimestamp())
            );
        
        // Windowed aggregation
        DataStream<AggregatedResult> results = events
            .keyBy(Event::getUserId)
            .window(TumblingEventTimeWindows.of(Time.minutes(5)))
            .aggregate(new EventAggregator());
        
        // Write to sink
        results.addSink(new FlinkKafkaProducer<>(
            "aggregated-results",
            new ResultSerializationSchema(),
            properties
        ));
        
        // Execute
        env.execute("Flink Streaming Job");
    }
}
```

---

## Data Lake Architecture

### Modern Data Lake Design

```
┌─────────────────────────────────────────────────────┐
│                    Data Sources                      │
│  (Web Logs, IoT, Databases, APIs, Files)           │
└───────────────────┬─────────────────────────────────┘
                    │
          ┌─────────┴─────────┐
          │                   │
┌─────────▼─────────┐  ┌──────▼───────┐
│  Ingestion Layer  │  │   Change     │
│  (Kafka, Flume)   │  │   Data       │
│                   │  │   Capture    │
└─────────┬─────────┘  └──────┬───────┘
          │                   │
          └─────────┬─────────┘
                    │
          ┌─────────▼─────────┐
          │   Raw Data Zone   │
          │   (Bronze Layer)  │
          │   Landing Area    │
          └─────────┬─────────┘
                    │
          ┌─────────▼─────────┐
          │  Cleansed Data    │
          │  (Silver Layer)   │
          │  Validated Data   │
          └─────────┬─────────┘
                    │
          ┌─────────▼─────────┐
          │  Curated Data     │
          │  (Gold Layer)     │
          │  Business Ready   │
          └─────────┬─────────┘
                    │
          ┌─────────┴─────────┬─────────────┐
          │                   │             │
    ┌─────▼──────┐    ┌──────▼──────┐  ┌──▼────┐
    │  Analytics │    │  ML Models  │  │  BI   │
    │  Workbench │    │  Training   │  │ Tools │
    └────────────┘    └─────────────┘  └───────┘
```

### Delta Lake Implementation

```python
from delta import *
from pyspark.sql import SparkSession
from pyspark.sql.functions import col, current_timestamp

# Create Spark session with Delta
spark = SparkSession.builder \
    .appName("Delta Lake Example") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# Write to Delta table
df = spark.read.json("/raw/user_events.json")

df.write \
    .format("delta") \
    .mode("overwrite") \
    .partitionBy("date") \
    .save("/delta/user_events")

# ACID transactions - Upsert (merge)
from delta.tables import DeltaTable

deltaTable = DeltaTable.forPath(spark, "/delta/user_events")

updates = spark.read.json("/updates/user_events.json")

deltaTable.alias("target") \
    .merge(
        updates.alias("source"),
        "target.event_id = source.event_id"
    ) \
    .whenMatchedUpdate(set={
        "status": "source.status",
        "updated_at": current_timestamp()
    }) \
    .whenNotMatchedInsert(values={
        "event_id": "source.event_id",
        "user_id": "source.user_id",
        "event_type": "source.event_type",
        "timestamp": "source.timestamp"
    }) \
    .execute()

# Time travel
# Read version from 1 hour ago
df_historical = spark.read \
    .format("delta") \
    .option("timestampAsOf", "2024-01-15 10:00:00") \
    .load("/delta/user_events")

# Read specific version
df_version = spark.read \
    .format("delta") \
    .option("versionAsOf", 5) \
    .load("/delta/user_events")

# Optimize with Z-ordering
deltaTable.optimize().executeZOrderBy("user_id", "timestamp")

# Vacuum old files (older than 7 days)
deltaTable.vacuum(168)  # hours
```

---

## Data Warehouse vs Data Lake

### Comparison Table

| **Aspect** | **Data Warehouse** | **Data Lake** |
|------------|-------------------|---------------|
| **Data Type** | Structured | All types (structured, semi-structured, unstructured) |
| **Schema** | Schema-on-write | Schema-on-read |
| **Processing** | ETL (Extract, Transform, Load) | ELT (Extract, Load, Transform) |
| **Cost** | Expensive (optimized storage) | Cheaper (commodity hardware) |
| **Users** | Business analysts | Data scientists, analysts, engineers |
| **Agility** | Less agile (schema changes) | More agile (store now, define later) |
| **Performance** | Optimized queries | Flexible but slower queries |
| **Examples** | Snowflake, Redshift, BigQuery | S3 + Spark, Azure Data Lake |

---

## Lambda & Kappa Architecture

### Lambda Architecture

**Combines batch and stream processing**

```
         ┌────────────────┐
         │  Data Sources  │
         └────────┬───────┘
                  │
         ┌────────┴───────┐
         │                │
    ┌────▼─────┐    ┌────▼──────┐
    │  Batch   │    │  Stream   │
    │  Layer   │    │  Layer    │
    │ (Hadoop) │    │  (Kafka)  │
    └────┬─────┘    └────┬──────┘
         │               │
    ┌────▼─────┐    ┌────▼──────┐
    │  Batch   │    │  Real-time│
    │  Views   │    │  Views    │
    └────┬─────┘    └────┬──────┘
         │               │
         └────────┬──────┘
                  │
         ┌────────▼───────┐
         │  Serving Layer │
         │   (Query API)  │
         └────────────────┘
```

**Implementation:**

```python
class LambdaArchitecture:
    def __init__(self):
        self.batch_processor = BatchProcessor()
        self.stream_processor = StreamProcessor()
        self.serving_layer = ServingLayer()
    
    def process_batch(self, start_date, end_date):
        """Batch layer - comprehensive, accurate"""
        data = self.batch_processor.read_from_hdfs(start_date, end_date)
        
        # Complex aggregations
        results = data.groupBy("user_id", "date") \
            .agg(
                sum("revenue").alias("total_revenue"),
                count("transactions").alias("transaction_count"),
                avg("order_value").alias("avg_order_value")
            )
        
        # Write batch views
        results.write \
            .mode("overwrite") \
            .parquet("/batch_views/user_metrics")
    
    def process_stream(self):
        """Speed layer - low latency, approximate"""
        stream = self.stream_processor.read_from_kafka("transactions")
        
        # Real-time aggregations
        realtime_metrics = stream \
            .groupBy(window("timestamp", "1 minute"), "user_id") \
            .agg(sum("amount").alias("realtime_revenue"))
        
        # Write to fast storage
        realtime_metrics.writeStream \
            .format("memory") \
            .queryName("realtime_views") \
            .start()
    
    def query(self, user_id, date):
        """Serving layer - merge batch + realtime"""
        # Get batch view (accurate historical data)
        batch_data = self.serving_layer.query_batch_view(user_id, date)
        
        # Get realtime view (recent data)
        realtime_data = self.serving_layer.query_realtime_view(user_id)
        
        # Merge results
        return self.merge_views(batch_data, realtime_data)
```

### Kappa Architecture

**Stream-only processing (simpler than Lambda)**

```
         ┌────────────────┐
         │  Data Sources  │
         └────────┬───────┘
                  │
         ┌────────▼───────┐
         │  Kafka / Event │
         │     Stream     │
         └────────┬───────┘
                  │
         ┌────────▼───────┐
         │  Stream        │
         │  Processing    │
         │  (Flink/Spark) │
         └────────┬───────┘
                  │
         ┌────────▼───────┐
         │  Serving Layer │
         │  (Fast Store)  │
         └────────────────┘
```

**Implementation:**

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import window, sum, count

class KappaArchitecture:
    def __init__(self):
        self.spark = SparkSession.builder \
            .appName("Kappa Architecture") \
            .getOrCreate()
    
    def process_stream(self):
        """Single stream processing path"""
        # Read from Kafka
        stream = self.spark \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "kafka:9092") \
            .option("subscribe", "all-events") \
            .load()
        
        # Parse and process
        events = stream.selectExpr("CAST(value AS STRING)") \
            .select(from_json(col("value"), schema).alias("data")) \
            .select("data.*")
        
        # Compute metrics (works for both real-time and batch)
        metrics = events \
            .withWatermark("timestamp", "10 minutes") \
            .groupBy(
                window("timestamp", "1 hour", "5 minutes"),
                "user_id"
            ) \
            .agg(
                sum("amount").alias("total_spent"),
                count("*").alias("event_count")
            )
        
        # Write to serving layer
        query = metrics \
            .writeStream \
            .format("parquet") \
            .option("path", "/serving/user_metrics") \
            .option("checkpointLocation", "/checkpoints/metrics") \
            .outputMode("append") \
            .start()
        
        # For reprocessing historical data:
        # Just replay Kafka from beginning
        return query
    
    def reprocess_historical(self, start_date):
        """Reprocess by replaying stream from specific point"""
        # Same processing logic, different starting offset
        stream = self.spark \
            .readStream \
            .format("kafka") \
            .option("kafka.bootstrap.servers", "kafka:9092") \
            .option("subscribe", "all-events") \
            .option("startingOffsets", f"{start_date}") \
            .load()
        
        # Same processing as real-time
        # This is the key advantage of Kappa: one codebase
```

---

## Real-time Analytics

### ClickHouse for Real-time Analytics

```sql
-- Create MergeTree table (ClickHouse engine)
CREATE TABLE events (
    event_id UUID,
    user_id UInt64,
    event_type String,
    timestamp DateTime,
    properties String,
    date Date MATERIALIZED toDate(timestamp)
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (user_id, timestamp)
SETTINGS index_granularity = 8192;

-- Create materialized view for real-time aggregation
CREATE MATERIALIZED VIEW events_hourly
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(hour)
ORDER BY (hour, event_type)
AS SELECT
    toStartOfHour(timestamp) AS hour,
    event_type,
    count() AS event_count,
    uniq(user_id) AS unique_users
FROM events
GROUP BY hour, event_type;

-- Real-time query (sub-second response)
SELECT
    toStartOfHour(timestamp) AS hour,
    event_type,
    count() AS events,
    uniq(user_id) AS users,
    quantile(0.95)(response_time) AS p95_response
FROM events
WHERE timestamp >= now() - INTERVAL 24 HOUR
GROUP BY hour, event_type
ORDER BY hour DESC, events DESC;
```

---

## Big Data Interview Questions

### Common Interview Questions

#### 1. **How do you handle skewed data in Spark?**

```python
# Problem: Some keys have way more data than others
# Solution 1: Salting
from pyspark.sql.functions import concat, lit, rand

# Add salt to skewed keys
df_salted = df.withColumn("salted_key", 
    concat(col("key"), lit("_"), (rand() * 10).cast("int")))

# Perform operation on salted keys
result = df_salted.groupBy("salted_key").agg(sum("value"))

# Solution 2: Adaptive Query Execution
spark.conf.set("spark.sql.adaptive.enabled", "true")
spark.conf.set("spark.sql.adaptive.skewJoin.enabled", "true")
```

#### 2. **Explain HDFS rack awareness**

```
Rack 1                  Rack 2
├── DataNode 1          ├── DataNode 4
├── DataNode 2          ├── DataNode 5
└── DataNode 3          └── DataNode 6

Block replication strategy:
- 1st replica: Same node as writer
- 2nd replica: Different rack
- 3rd replica: Same rack as 2nd, different node

Why? Balance between reliability and network bandwidth
```

#### 3. **How do you optimize Spark jobs?**

```python
# 1. Use appropriate partitioning
df.repartition(200, "key_column")

# 2. Broadcast small tables
from pyspark.sql.functions import broadcast
large_df.join(broadcast(small_df), "key")

# 3. Use columnar formats (Parquet, ORC)
df.write.parquet("path")

# 4. Cache frequently used DataFrames
df.cache()

# 5. Avoid shuffles when possible
# Bad: df.groupBy("col").count()
# Better: Use DataFrames instead of RDDs

# 6. Use appropriate compression
spark.conf.set("spark.sql.parquet.compression.codec", "snappy")
```

#### 4. **Design a real-time recommendation system**

```
Architecture:
┌──────────────┐
│  User Actions│ → Kafka → Spark Streaming
└──────────────┘              ↓
                    Feature Engineering
                              ↓
                    Update ML Model Online
                              ↓
                    Store in Redis (fast lookup)
                              ↓
                    API serves recommendations
```

This comprehensive Big Data guide covers everything needed for big data engineering interviews, from fundamentals to advanced architectures!
