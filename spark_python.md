# Apache Spark (PySpark) Tutorial: Beginner to Advanced

## Table of Contents

1. Introduction to Apache Spark
2. Beginner Level
   - Setting up PySpark
   - First PySpark Application (local)
   - RDDs and DataFrames
   - Basic transformations and actions
3. Intermediate Level
   - Spark SQL
   - DataFrame operations and optimization
   - UDFs and UDAFs
   - Reading/Writing data (Parquet, CSV, JSON)
   - Spark Streaming (Structured Streaming)
4. Advanced Level
   - Performance tuning and partitioning
   - Join strategies and shuffle optimization
   - MLlib (Machine Learning)
   - GraphFrames (graph processing)
   - Production patterns and deployment
5. Complete Examples
   - ETL pipeline example (CSV -> Parquet -> Aggregations)
   - Streaming example (Kafka -> Spark -> Sink)
6. Best Practices

---

## 1. Introduction to Apache Spark

Apache Spark is a fast, general-purpose cluster-computing system for big data processing. PySpark is the Python API for Spark.

Key features:
- In-memory processing
- Lazy evaluation
- Rich APIs: RDDs, DataFrames, Datasets (Scala/Java)
- Libraries: Spark SQL, MLlib, GraphX/GraphFrames, Streaming

---

## 2. Beginner Level

### Setting up PySpark

Install PySpark with pip for local development:

```powershell
pip install pyspark
```

For a specific Spark version, use:

```powershell
pip install pyspark==3.4.1
```

### First PySpark Application (local)

`example_local.py` - word count example

```python
from pyspark.sql import SparkSession

def main():
    spark = SparkSession.builder 
        .appName("WordCountLocal") 
        .master("local[*]") 
        .getOrCreate()

    sc = spark.sparkContext

    data = sc.parallelize([
        "hello world",
        "apache spark",
        "hello spark",
        "hello world"
    ])

    counts = (data.flatMap(lambda line: line.split())
                 .map(lambda w: (w, 1))
                 .reduceByKey(lambda a, b: a + b))

    for word, count in counts.collect():
        print(f"{word}: {count}")

    spark.stop()

if __name__ == '__main__':
    main()
```

Run locally:

```powershell
python example_local.py
```

### RDDs and DataFrames

- RDD: low-level resilient distributed dataset
- DataFrame: higher-level, optimized, with schema

Creating DataFrame:

```python
from pyspark.sql import SparkSession
from pyspark.sql import Row

spark = SparkSession.builder.master("local[*]").appName("DFExample").getOrCreate()

rows = [Row(id=1, name='Alice', age=30), Row(id=2, name='Bob', age=25)]
df = spark.createDataFrame(rows)
df.printSchema()
df.show()

spark.stop()
```

### Basic transformations and actions

- Transformations: map, filter, flatMap, join (lazy)
- Actions: collect, count, take, save

Example: filter and aggregate

```python
filtered = df.filter(df.age > 26)
print(filtered.collect())
```

---

## 3. Intermediate Level

### Spark SQL

Register DataFrame as temp view and run SQL:

```python
spark.sql("""
SELECT name, age FROM people WHERE age > 25
""")
```

### DataFrame operations and optimization

- Use select, withColumn, groupBy, agg
- Avoid UDFs when possible; use built-in functions
- Use explain() to inspect physical plan

Example: aggregation and window functions

```python
from pyspark.sql import functions as F
from pyspark.sql.window import Window

sales = spark.createDataFrame([
    ("2025-01-01", "A", 100),
    ("2025-01-02", "A", 200),
    ("2025-01-01", "B", 300),
], ["date", "product", "revenue"])

window = Window.partitionBy("product").orderBy("date").rowsBetween(-1, 0)

sales.withColumn("rolling_sum", F.sum("revenue").over(window)).show()
```

### UDFs and UDAFs

Define UDF:

```python
from pyspark.sql.functions import udf
from pyspark.sql.types import StringType

@udf(returnType=StringType())
def normalize(s):
    return s.strip().lower() if s else None

df.withColumn('name_normalized', normalize(df.name)).show()
```

For heavy use, prefer pandas UDFs (vectorized) for performance.

### Reading/Writing data

```python
# Parquet
df.write.mode('overwrite').parquet('/tmp/output/parquet')
# CSV
df.write.option('header', True).csv('/tmp/output/csv')
# Read
parquet_df = spark.read.parquet('/tmp/output/parquet')
```

### Structured Streaming (basic)

```python
from pyspark.sql.types import StructType, StructField, StringType

schema = StructType([StructField('value', StringType())])

stream = (spark.readStream
          .format('socket')
          .option('host', 'localhost')
          .option('port', 9999)
          .load())

query = (stream.writeStream
         .format('console')
         .start())

query.awaitTermination()
```

---

## 4. Advanced Level

### Performance tuning and partitioning

- control partition numbers with repartition/coalesce
- use map-side aggregation with reduceByKey
- persist/cache hot DataFrames

Example: repartition

```python
df = df.repartition(200, 'date')
```

### Join strategies and shuffle optimization

- Broadcast small tables with broadcast() to avoid shuffle

```python
from pyspark.sql.functions import broadcast

joined = large_df.join(broadcast(small_df), 'key')
```

### MLlib (Machine Learning)

Simple pipeline: feature assembler + LogisticRegression

```python
from pyspark.ml.feature import VectorAssembler
from pyspark.ml.classification import LogisticRegression

assembler = VectorAssembler(inputCols=['f1', 'f2'], outputCol='features')
train = assembler.transform(train_df)

lr = LogisticRegression(featuresCol='features', labelCol='label')
model = lr.fit(train)

pred = model.transform(test_df)
```

### GraphFrames

Install graphframes package and use for graph algorithms (PageRank, connected components)

---

## 5. Complete Examples

### ETL Pipeline (CSV -> Parquet -> Aggregations)

`etl_pipeline.py`

```python
from pyspark.sql import SparkSession
from pyspark.sql import functions as F

spark = SparkSession.builder.appName('ETL').getOrCreate()

# Read CSV
raw = spark.read.option('header', True).csv('data/sales.csv')

# Cast and clean
clean = (raw
         .withColumn('revenue', F.col('revenue').cast('double'))
         .withColumn('date', F.to_date('date', 'yyyy-MM-dd')))

# Write Parquet
clean.write.mode('overwrite').partitionBy('date').parquet('output/sales_parquet')

# Read back and aggregate
sales = spark.read.parquet('output/sales_parquet')
agg = sales.groupBy('date').agg(F.sum('revenue').alias('daily_revenue'))
agg.orderBy('date').show()

spark.stop()
```

### Streaming Example (Kafka -> Spark -> Console)

```python
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col
from pyspark.sql.types import StructType, StringType, DoubleType

spark = SparkSession.builder.appName('KafkaStreaming').getOrCreate()

schema = StructType().add('id', StringType()).add('value', DoubleType())

df = (spark.readStream
      .format('kafka')
      .option('kafka.bootstrap.servers', 'localhost:9092')
      .option('subscribe', 'input-topic')
      .load())

json_df = df.selectExpr('CAST(value AS STRING) as json')
parsed = json_df.select(from_json(col('json'), schema).alias('data')).select('data.*')

query = (parsed.writeStream
         .outputMode('append')
         .format('console')
         .start())

query.awaitTermination()
```

---

## 6. Best Practices

- Prefer DataFrames and dataset APIs for performance
- Push computations to Spark SQL when possible
- Tune parallelism and partitions based on data size and cluster
- Avoid small files (use coalesce before writing many small partitions)
- Use checkpointing for long streaming jobs
- Monitor via Spark UI and logs

---

## Try it

Run local examples with an installed PySpark. For streaming examples, provide external services like Kafka.

```powershell
python example_local.py
python etl_pipeline.py
```

---

## Conclusion

This PySpark guide gives a full path from local development to production-ready patterns. Use it as a living document and adapt cluster settings and versions to your environment.
