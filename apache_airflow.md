# Apache Airflow Tutorial: Beginner to Advanced

## Table of Contents

1. Introduction to Airflow
2. Beginner Level
   - Installing Airflow
   - First DAG and concepts (Operators, Tasks, DAGs)
   - Scheduling and dependencies
3. Intermediate Level
   - XComs and Variables
   - Sensors and Hooks
   - Task retries, SLA, and alerts
   - Modular DAGs and best structuring
4. Advanced Level
   - Dynamic DAGs and Task Mapping
   - Scaling Airflow (Celery/Kubernetes Executors)
   - Observability and monitoring
   - Production deployment patterns
5. Complete Examples
   - ETL DAG (CSV -> Parquet -> Load)
   - Streaming orchestrator DAG (triggering jobs)
6. Best Practices

---

## 1. Introduction to Airflow

Apache Airflow is a platform to programmatically author, schedule, and monitor workflows as DAGs (Directed Acyclic Graphs). Use Airflow to orchestrate ETL jobs, ML training, and other data workflows.

Key concepts:
- DAG: Directed Acyclic Graph representing the workflow
- Task: Unit of work, implemented by Operators
- Operator: Template for work (BashOperator, PythonOperator, SparkSubmitOperator)
- Sensor: Waits for external condition
- Hook: Integration for external systems (S3, Postgres, Kafka)

---

## 2. Beginner Level

### Installing Airflow (local)

Use constraints for stable versions. Example using pip (Linux/Mac/Windows WSL):

```powershell
pip install "apache-airflow==2.7.1" --constraint "https://raw.githubusercontent.com/apache/airflow/constraints-2.7.1/constraints-3.10.txt"
```

Initialize the database and start the webserver and scheduler:

```powershell
airflow db init
airflow users create --username admin --firstname Admin --lastname User --role Admin --email admin@example.com
airflow webserver --port 8080
airflow scheduler
```

### First DAG (Hello World)

`dags/hello_world.py`

```python
from datetime import datetime
from airflow import DAG
from airflow.operators.bash import BashOperator

with DAG('hello_world', start_date=datetime(2025,1,1), schedule_interval='@daily', catchup=False) as dag:
	t1 = BashOperator(task_id='print_date', bash_command='date')
	t2 = BashOperator(task_id='sleep', bash_command='sleep 5')
	t1 >> t2
```

Place the file in your `dags/` folder and watch the DAG appear in the Airflow UI.

### Operators and Sensors

- PythonOperator: run Python callable
- BashOperator: execute shell command
- SparkSubmitOperator: submit Spark jobs
- S3KeySensor: wait for a file in S3

Example PythonOperator:

```python
from airflow.operators.python import PythonOperator

def greet(name):
	print(f"Hello {name}")

greet_task = PythonOperator(task_id='greet', python_callable=greet, op_args=['Airflow'])
```

---

## 3. Intermediate Level

### XComs and Variables

XComs allow tasks to pass small amounts of data. Variables store config values.

```python
from airflow.models import Variable

my_var = Variable.get("my_config", default_var="default")

def push_xcom(**kwargs):
	kwargs['ti'].xcom_push(key='value', value=123)

def pull_xcom(**kwargs):
	v = kwargs['ti'].xcom_pull(key='value', task_ids='push_task')
	print(v)
```

### Hooks and Connections

Configure Connections in the Airflow UI for services (AWS, Postgres, Kafka). Use Hooks to interact programmatically.

Example PostgresHook usage:

```python
from airflow.providers.postgres.hooks.postgres import PostgresHook

pg = PostgresHook(postgres_conn_id='my_postgres')
conn = pg.get_conn()
cursor = conn.cursor()
cursor.execute('SELECT 1')
```

### Task retries, SLA, and alerts

Set retries and email alerts on failures:

```python
from airflow.utils.email import send_email

def on_failure(context):
	send_email(to=['dev@example.com'], subject='Task failed', html_content='See logs')

task = PythonOperator(task_id='task', python_callable=do_work, retries=3, on_failure_callback=on_failure)
```

### Modular DAGs and subDAGs (Task Groups)

Use TaskGroup to group related tasks and keep DAGs readable.

```python
from airflow.utils.task_group import TaskGroup

with TaskGroup('extract') as extract_group:
	extract_a = BashOperator(task_id='ex_a', bash_command='echo a')
	extract_b = BashOperator(task_id='ex_b', bash_command='echo b')
```

---

## 4. Advanced Level

### Dynamic DAGs and Task Mapping

Use `expand`/`map` APIs for dynamic, parallel task creation.

```python
from airflow.decorators import task

@task
def process(x):
	return x * 2

inputs = [1,2,3]
processed = process.expand(x=inputs)
```

### Scaling Airflow (Executors)

- LocalExecutor: single machine (development)
- CeleryExecutor: scale with workers (production)
- KubernetesExecutor: scale with k8s pods (cloud-native)

Example: switching executor requires configuring `airflow.cfg` and setting up a message broker like RabbitMQ/Redis for Celery.

### Observability and monitoring

- Use Prometheus and Grafana for metrics (Airflow exporter)
- Use ELK/Cloud logging for logs
- Set up SLAs and email alerts

### Production deployment patterns

- Use CI/CD to deploy DAGs and Docker images
- Lock dependencies with constraints files
- Secure Airflow UI with RBAC and proper authentication

---

## 5. Complete Examples

### ETL DAG (CSV -> Parquet -> Load)

`dags/etl_pipeline.py`

```python
from datetime import datetime, timedelta
from airflow import DAG
from airflow.operators.python import PythonOperator
from airflow.providers.amazon.aws.hooks.s3 import S3Hook
from pyspark.sql import SparkSession

default_args = {
	'owner': 'airflow',
	'depends_on_past': False,
	'retries': 1,
	'retry_delay': timedelta(minutes=5),
}

def etl_task(**kwargs):
	s3 = S3Hook(aws_conn_id='aws_default')
	# download from S3, process with Spark locally or submit to cluster
	spark = SparkSession.builder.appName('AirflowETL').getOrCreate()
	df = spark.read.option('header', True).csv('/path/to/input.csv')
	df = df.withColumn('revenue', df['revenue'].cast('double'))
	df.write.mode('overwrite').parquet('/path/to/output')
	spark.stop()

with DAG('etl_pipeline', default_args=default_args, start_date=datetime(2025,1,1), schedule_interval='@daily', catchup=False) as dag:
	etl = PythonOperator(task_id='run_etl', python_callable=etl_task)

	etl
```

### Triggering External Jobs (Spark submit)

Use `SparkSubmitOperator` to submit Spark jobs from Airflow:

```python
from airflow.providers.apache.spark.operators.spark_submit import SparkSubmitOperator

spark_task = SparkSubmitOperator(
	task_id='spark_job',
	application='/opt/airflow/dags/jobs/my_spark_job.jar',
	conn_id='spark_default'
)
```

---

## 6. Best Practices

- Keep DAGs small and modular
- Use connections and variables for secrets (do NOT hardcode credentials)
- Write idempotent tasks (safe retries)
- Use task mapping and dynamic workflows for scalability
- Version control DAGs and deploy via CI/CD
- Monitor DAG run times and set SLAs
- Use environment-specific configurations for dev/prod

---

## Try it

Start the Airflow components locally and place the sample DAGs in the `dags/` folder. Use the Airflow UI at `http://localhost:8080` to trigger and monitor runs.

```powershell
airflow db init
airflow webserver --port 8080
airflow scheduler
```

---

## Conclusion

This Airflow guide provides a path from initial DAGs to production-grade orchestration. Combine Airflow with Spark, Kubernetes, and cloud services to build robust data platforms.
