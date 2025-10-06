# Apache Spark (Scala) Tutorial: Beginner to Advanced

## Table of Contents

1. Introduction to Apache Spark
2. Beginner Level
	 - Setting up Spark with Scala
	 - First Spark Application (local)
	 - RDDs vs DataFrames vs Datasets
	 - Basic transformations and actions
3. Intermediate Level
	 - Spark SQL and Datasets
	 - Writing optimized DataFrame code
	 - UDFs and Typed UDFs
	 - Reading/Writing data (Parquet, CSV, JSON)
	 - Structured Streaming
4. Advanced Level
	 - Performance tuning and partitioning
	 - Join strategies and shuffle optimization
	 - Spark MLlib
	 - Graph processing with GraphFrames
	 - Productionizing Spark jobs
5. Complete Examples
	 - Batch ETL example (CSV -> Parquet -> Aggregation)
	 - Streaming example (Kafka -> Spark -> Sink)
6. Best Practices

---

## 1. Introduction to Apache Spark

Apache Spark is a distributed data processing engine for big data workloads. Scala is Spark's native language, and many Spark APIs are first-class in Scala.

Key features:
- In-memory processing
- Lazy evaluation
- Rich type-safe Dataset API
- Libraries: Spark SQL, MLlib, GraphX/GraphFrames, Structured Streaming

---

## 2. Beginner Level

### Setting up Spark with Scala

Use sbt or Maven. Example `build.sbt` for sbt projects:

```scala
name := "spark-scala-tutorial"

version := "0.1.0"

scalaVersion := "2.13.12"

libraryDependencies ++= Seq(
	"org.apache.spark" %% "spark-core" % "3.4.1" % "provided",
	"org.apache.spark" %% "spark-sql" % "3.4.1" % "provided",
	"org.apache.spark" %% "spark-mllib" % "3.4.1" % "provided",
	"org.graphframes" %% "graphframes" % "0.8.2-spark3.4-s_2.13" // optional
)
```

For running locally during development, remove `% "provided"` or use the Spark libraries packaged into an assembly jar.

### First Spark Application (local)

`src/main/scala/com/example/WordCount.scala`

```scala
package com.example

import org.apache.spark.sql.SparkSession

object WordCount {
	def main(args: Array[String]): Unit = {
		val spark = SparkSession.builder()
			.appName("WordCountLocal")
			.master("local[*]")
			.getOrCreate()

		val sc = spark.sparkContext

		val data = sc.parallelize(Seq(
			"hello world",
			"apache spark",
			"hello spark",
			"hello world"
		))

		val counts = data.flatMap(_.split("\\s+"))
			.map((_, 1))
			.reduceByKey(_ + _)

		counts.collect().foreach { case (w, c) => println(s"$w: $c") }

		spark.stop()
	}
}
```

Build and run with sbt (or use spark-submit on a cluster):

```powershell
sbt run
```

### RDDs vs DataFrames vs Datasets

- RDD: low-level, untyped, flexible
- DataFrame: untyped row-based API with Catalyst optimizations
- Dataset[T]: typed API (Scala/Java) combining type-safety and optimization

Example creating a Dataset:

```scala
case class Person(id: Long, name: String, age: Int)

val spark = SparkSession.builder.master("local[*]").appName("DSExample").getOrCreate()
import spark.implicits._

val ds = Seq(Person(1, "Alice", 30), Person(2, "Bob", 25)).toDS()
ds.printSchema()
ds.show()

spark.stop()
```

### Basic transformations and actions

- Transformations (map, filter, flatMap, groupByKey) are lazy
- Actions (collect, count, take) trigger execution

Example: filter and collect

```scala
val adults = ds.filter(_.age >= 18).collect()
```

---

## 3. Intermediate Level

### Spark SQL and Datasets

Register a Dataset/DataFrame as a view and run SQL queries:

```scala
ds.createOrReplaceTempView("people")
val adults = spark.sql("SELECT name, age FROM people WHERE age >= 18")
adults.show()
```

### Writing optimized DataFrame code

- Use built-in functions (org.apache.spark.sql.functions) instead of UDFs
- Chain select and withColumn to avoid unnecessary shuffles
- Use broadcast joins when joining small tables

Example: window functions and aggregations

```scala
import org.apache.spark.sql.expressions.Window
import org.apache.spark.sql.functions._

val sales = Seq(("2025-01-01", "A", 100), ("2025-01-02", "A", 200), ("2025-01-01", "B", 300)).toDF("date", "product", "revenue")
val w = Window.partitionBy("product").orderBy("date").rowsBetween(-1, 0)
val withRolling = sales.withColumn("rolling_sum", sum("revenue").over(w))
withRolling.show()
```

### UDFs and Typed UDFs

Use UDFs sparingly. Example of registering a simple UDF:

```scala
import org.apache.spark.sql.functions.udf

val normalize = udf((s: String) => if (s == null) null else s.trim.toLowerCase)
df.withColumn("name_normalized", normalize(col("name"))).show()
```

Typed transformations (mapPartitions, mapGroups) on Datasets can be more efficient.

### Reading/Writing data

```scala
df.write.mode("overwrite").parquet("/tmp/output/parquet")
val parquet = spark.read.parquet("/tmp/output/parquet")
```

### Structured Streaming

Basic streaming from socket:

```scala
import org.apache.spark.sql.functions._

val stream = spark.readStream.format("socket").option("host", "localhost").option("port", 9999).load()
val words = stream.as[String].flatMap(_.split("\\s+"))
val counts = words.groupBy("value").count()

val query = counts.writeStream.format("console").outputMode("complete").start()
query.awaitTermination()
```

---

## 4. Advanced Level

### Performance tuning and partitioning

- Control parallelism with `repartition` and `coalesce`
- Cache hot Datasets: `df.persist(StorageLevel.MEMORY_ONLY)`
- Avoid wide transformations when possible

Example: repartition by a key

```scala
val byDate = df.repartition(200, col("date"))
```

### Join strategies and shuffle optimization

- Broadcast small tables using `broadcast` from functions

```scala
import org.apache.spark.sql.functions.broadcast
val joined = large.join(broadcast(small), Seq("key"))
```

### MLlib (Machine Learning)

Simple pipeline example (VectorAssembler + LogisticRegression):

```scala
import org.apache.spark.ml.feature.VectorAssembler
import org.apache.spark.ml.classification.LogisticRegression

val assembler = new VectorAssembler().setInputCols(Array("f1", "f2")).setOutputCol("features")
val train = assembler.transform(trainDF)
val lr = new LogisticRegression().setLabelCol("label").setFeaturesCol("features")
val model = lr.fit(train)
val predictions = model.transform(testDF)
```

### GraphFrames

Graph processing using GraphFrames (requires package) for algorithms like PageRank.

### Productionizing Spark jobs

- Build fat/assembly JAR and submit with `spark-submit`
- Use cluster schedulers (YARN, Kubernetes)
- Use monitoring (Spark UI, Ganglia, Prometheus)

---

## 5. Complete Examples

### Batch ETL (CSV -> Parquet -> Aggregation)

`src/main/scala/com/example/etl/ETLApp.scala`

```scala
package com.example.etl

import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._

object ETLApp {
	def main(args: Array[String]): Unit = {
		val spark = SparkSession.builder().appName("ETLApp").master("local[*]").getOrCreate()
		import spark.implicits._

		val raw = spark.read.option("header", true).csv("data/sales.csv")

		val clean = raw.withColumn("revenue", col("revenue").cast("double"))
			.withColumn("date", to_date(col("date"), "yyyy-MM-dd"))

		clean.write.mode("overwrite").partitionBy("date").parquet("output/sales_parquet")

		val sales = spark.read.parquet("output/sales_parquet")
		val agg = sales.groupBy("date").agg(sum("revenue").alias("daily_revenue"))
		agg.orderBy("date").show()

		spark.stop()
	}
}
```

### Streaming (Kafka -> Spark -> Console)

```scala
import org.apache.spark.sql.SparkSession
import org.apache.spark.sql.functions._
import org.apache.spark.sql.types._

val spark = SparkSession.builder.appName("KafkaStream").master("local[*]").getOrCreate()

val schema = new StructType().add("id", StringType).add("value", DoubleType)

val df = spark.readStream.format("kafka").option("kafka.bootstrap.servers", "localhost:9092").option("subscribe", "input-topic").load()
val json = df.selectExpr("CAST(value AS STRING) as json")
val parsed = json.select(from_json(col("json"), schema).alias("data")).select("data.*")

val query = parsed.writeStream.format("console").start()
query.awaitTermination()
```

---

## 6. Best Practices

- Prefer Datasets for type safety in Scala
- Push computation into Spark SQL when possible
- Tune partitioning based on cluster size and data volume
- Avoid small files; coalesce before writing many small partitions
- Use checkpointing for streaming jobs
- Monitor via Spark UI and logs

---

## Try it

Build and run locally with sbt. For cluster runs, package an assembly jar and run `spark-submit`.

```powershell
sbt assembly
spark-submit --class com.example.etl.ETLApp --master local[*] target/scala-2.13/spark-scala-tutorial-assembly-0.1.0.jar
```

---

## Conclusion

This Scala Spark guide maps the learning path from local development to production patterns. Tailor partitioning, shuffle tuning, and resource configurations to your cluster and data sizes.
