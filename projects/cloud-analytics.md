# Multi-Source Cloud Analytics Platform

**Company:** Pepper Banking (Bank Leumi)  
**Role:** Sr. Data Analyst & BI Developer  
**Duration:** Nov 2021 – Nov 2024  
**Tech Stack:** AWS (S3, Glue, Athena), BigQuery, PostgreSQL, Python

---

## 🎯 Business Challenge

The organization needed to combine diverse data sources for comprehensive analytics:

- **Siloed Data:** On-premises user behavior data + external cloud datasets isolated
- **Manual Integration:** No infrastructure to efficiently merge cross-platform data
- **Scalability Issues:** Traditional ETL couldn't handle growing data volumes
- **Cost Concerns:** Infrastructure expenses climbing with data growth
- **Time to Insight:** Hours-long processes to generate cross-source reports

---

## 🏗️ Solution Architecture

### System Design

```
┌──────────────────────────────────────────────────────────────┐
│                     Data Sources                              │
├──────────────────────────────────────────────────────────────┤
│  PostgreSQL (On-Prem)    │    AWS S3 (External Data)         │
│  • User behavior         │    • Market data                  │
│  • Session tracking      │    • External analytics           │
│  • Transaction logs      │    • Third-party feeds            │
└───────────┬──────────────┴────────────┬──────────────────────┘
            │                            │
            ▼                            ▼
┌───────────────────────┐    ┌─────────────────────────────────┐
│  Extract & Transform  │    │    AWS Glue Data Catalog        │
│  • Python scripts     │    │    • Schema definitions         │
│  • Event processing   │    │    • Metadata management        │
│  • Data validation    │    │    • Crawler automation         │
└───────────┬───────────┘    └──────────┬──────────────────────┘
            │                            │
            └────────────┬───────────────┘
                         ▼
            ┌────────────────────────────┐
            │   AWS Glue ETL Jobs        │
            │   • Data transformation     │
            │   • Quality checks          │
            │   • Partitioning            │
            └────────────┬───────────────┘
                         │
         ┌───────────────┼───────────────┐
         ▼               ▼               ▼
┌─────────────┐  ┌──────────────┐  ┌──────────────┐
│  BigQuery   │  │  AWS Athena  │  │  Combined    │
│  Warehouse  │  │  Serverless  │  │  Reporting   │
└─────────────┘  └──────────────┘  └──────────────┘
```

---

## ⚙️ Technical Implementation

### 1. Data Extraction & Integration

**PostgreSQL Analytics Extraction:**

```python
# extract_postgres_data.py
import psycopg2
import pandas as pd
from datetime import datetime, timedelta

class PostgreSQLExtractor:
    """
    Extract user behavior and analytics data from on-prem PostgreSQL
    """
    
    def __init__(self, connection_config):
        self.conn = psycopg2.connect(**connection_config)
    
    def extract_user_events(self, start_date, end_date):
        """
        Extract user behavior metrics, session data, and interactions
        """
        query = """
        SELECT 
            event_date,
            user_id,
            event_type,
            event_properties,
            session_id,
            device_type,
            country_code,
            revenue_amount
        FROM analytics.user_events
        WHERE event_date BETWEEN %s AND %s
        AND event_type NOT IN ('page_ping', 'heartbeat')
        ORDER BY event_date, event_timestamp;
        """
        
        df = pd.read_sql(query, self.conn, params=(start_date, end_date))
        
        # Data validation
        assert df['user_id'].notnull().all(), "Null user_ids found"
        assert df['event_date'].between(start_date, end_date).all()
        
        return df
    
    def upload_to_s3(self, df, bucket, key):
        """
        Upload extracted data to S3 in Parquet format for efficiency
        """
        df.to_parquet(
            f's3://{bucket}/{key}',
            engine='pyarrow',
            compression='snappy',
            partition_cols=['event_date']
        )
```

**AWS Glue Data Catalog Setup:**

```python
# setup_glue_catalog.py
import boto3

class GlueCatalogManager:
    """
    Manage AWS Glue Data Catalog for schema definitions
    """
    
    def __init__(self):
        self.glue_client = boto3.client('glue')
    
    def create_external_tables(self, database_name):
        """
        Define external tables pointing to S3 data
        """
        tables = [
            {
                'name': 'user_events',
                'location': 's3://analytics-bucket/user-events/',
                'columns': [
                    {'name': 'event_date', 'type': 'date'},
                    {'name': 'user_id', 'type': 'string'},
                    {'name': 'event_type', 'type': 'string'},
                    {'name': 'revenue_amount', 'type': 'decimal(10,2)'},
                ]
            },
            {
                'name': 'external_market_data',
                'location': 's3://external-data-bucket/market/',
                'columns': [
                    {'name': 'date', 'type': 'date'},
                    {'name': 'symbol', 'type': 'string'},
                    {'name': 'value', 'type': 'decimal(15,4)'},
                ]
            }
        ]
        
        for table_def in tables:
            self.glue_client.create_table(
                DatabaseName=database_name,
                TableInput=self._build_table_input(table_def)
            )
```

### 2. AWS Glue ETL Pipeline

**Transformation Jobs:**

```python
# glue_etl_job.py
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

# Initialize Glue context
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)

def transform_user_data():
    """
    Transform and enrich user behavior data
    """
    # Read from Glue catalog
    user_events = glueContext.create_dynamic_frame.from_catalog(
        database="analytics_db",
        table_name="user_events"
    )
    
    # Transform to DataFrame for complex operations
    df = user_events.toDF()
    
    # Data enrichment and transformation
    from pyspark.sql import functions as F
    
    enriched_df = df \
        .withColumn('event_hour', F.hour('event_timestamp')) \
        .withColumn('is_revenue_event', F.col('revenue_amount') > 0) \
        .withColumn('user_cohort', F.date_trunc('month', F.col('first_seen_date'))) \
        .filter(F.col('event_date') >= F.current_date() - 90)  # Last 90 days
    
    # Data quality checks
    assert enriched_df.filter(F.col('user_id').isNull()).count() == 0
    
    # Write to S3 partitioned by date
    enriched_df.write \
        .mode('overwrite') \
        .partitionBy('event_date') \
        .parquet('s3://processed-data-bucket/enriched-events/')

def merge_sources():
    """
    Combine on-prem PostgreSQL data with external S3 datasets
    """
    # User data from PostgreSQL (via S3 staging)
    user_data = spark.read.parquet('s3://analytics-bucket/user-events/')
    
    # External market data from S3
    market_data = spark.read.parquet('s3://external-data-bucket/market/')
    
    # Join on date for combined analysis
    combined = user_data.alias('u') \
        .join(
            market_data.alias('m'),
            F.col('u.event_date') == F.col('m.date'),
            'left'
        ) \
        .select(
            'u.event_date',
            'u.user_id',
            'u.revenue_amount',
            'm.symbol',
            'm.value'
        )
    
    # Write combined dataset
    combined.write \
        .mode('overwrite') \
        .partitionBy('event_date') \
        .parquet('s3://combined-analytics/merged-data/')
```

### 3. Athena Query Optimization

**Serverless SQL Queries:**

```sql
-- Optimized query with partitioning for cost efficiency
CREATE EXTERNAL TABLE IF NOT EXISTS combined_analytics (
    user_id STRING,
    revenue_amount DECIMAL(10,2),
    market_value DECIMAL(15,4),
    event_hour INT
)
PARTITIONED BY (event_date DATE)
STORED AS PARQUET
LOCATION 's3://combined-analytics/merged-data/'
TBLPROPERTIES ('parquet.compression'='SNAPPY');

-- Query with partition pruning (reduces scanned data = lower cost)
SELECT 
    event_date,
    COUNT(DISTINCT user_id) as active_users,
    SUM(revenue_amount) as total_revenue,
    AVG(market_value) as avg_market_value
FROM combined_analytics
WHERE event_date BETWEEN DATE '2024-01-01' AND DATE '2024-12-31'  -- Partition filter
GROUP BY event_date
ORDER BY event_date;
```

**Cost Optimization Strategies:**
- ✅ Parquet format (columnar storage, 60% smaller than CSV)
- ✅ Snappy compression (additional 30% reduction)
- ✅ Partition pruning (scan only relevant data)
- ✅ Column selection (avoid `SELECT *`)

### 4. BigQuery Integration

**Cross-Cloud Analytics:**

```python
# bigquery_integration.py
from google.cloud import bigquery
import pandas as pd

class BigQueryIntegration:
    """
    Load data from AWS S3 to BigQuery for unified analytics
    """
    
    def __init__(self, project_id):
        self.client = bigquery.Client(project=project_id)
    
    def load_from_s3(self, s3_uri, dataset_id, table_id):
        """
        Transfer data from S3 to BigQuery
        """
        job_config = bigquery.LoadJobConfig(
            source_format=bigquery.SourceFormat.PARQUET,
            write_disposition=bigquery.WriteDisposition.WRITE_TRUNCATE,
        )
        
        load_job = self.client.load_table_from_uri(
            source_uris=[s3_uri],
            destination=f'{dataset_id}.{table_id}',
            job_config=job_config
        )
        
        load_job.result()  # Wait for completion
        
        table = self.client.get_table(f'{dataset_id}.{table_id}')
        print(f'Loaded {table.num_rows} rows into {table_id}')
    
    def run_cross_source_query(self):
        """
        Query combining BigQuery and Athena data
        """
        query = """
        SELECT 
            bq.user_id,
            bq.total_revenue,
            athena.conversion_rate
        FROM `project.dataset.bigquery_table` bq
        LEFT JOIN `project.dataset.athena_federated` athena
            ON bq.user_id = athena.user_id
        WHERE bq.event_date >= CURRENT_DATE() - 30
        """
        
        results = self.client.query(query).to_dataframe()
        return results
```

---

## 📈 Quantifiable Results

### Performance & Cost

| Metric | Before | After | Improvement |
|--------|--------|-------|-------------|
| **Infrastructure Cost** | $12K/month | $7.2K/month | **40% reduction** |
| **Query Response Time** | 2-5 minutes | 5-15 seconds | **92% faster** |
| **Data Processing** | Batch (daily) | Near real-time | **Same-day insights** |
| **Scalability** | Limited | Serverless | **Unlimited growth** |

### Business Impact

- ✅ **5+ data sources** integrated into single platform
- ✅ **Unified reporting** eliminating manual data gathering
- ✅ **Serverless architecture** scaling automatically with demand
- ✅ **Cross-team adoption** (product, finance, marketing all using platform)

### Technical Achievements

- ✅ **Zero downtime** migration from legacy system
- ✅ **Automated pipelines** reducing manual intervention 90%
- ✅ **Data freshness** improved from daily to hourly
- ✅ **Query flexibility** - SQL access for non-engineers

---

## 🔍 Data Quality Framework

**Validation Pipeline:**

```python
class DataQualityChecker:
    """
    Comprehensive validation for cloud data pipelines
    """
    
    def validate_pipeline_output(self, table_name, date):
        checks = {
            'row_count': self.check_expected_volume(table_name, date),
            'freshness': self.check_data_freshness(table_name),
            'schema': self.validate_schema_compliance(table_name),
            'duplicates': self.check_for_duplicates(table_name, date),
            'nulls': self.check_critical_nulls(table_name),
            'reconciliation': self.reconcile_with_source(table_name, date)
        }
        
        failed = [k for k, v in checks.items() if not v['passed']]
        
        if failed:
            self.send_slack_alert(table_name, failed)
            self.log_to_cloudwatch(checks)
        
        return len(failed) == 0
```

---

## 🎯 Key Learnings

### Technical Insights

1. **Serverless is cost-effective at scale**  
   Pay-per-query model saved 40% vs always-on infrastructure

2. **Parquet + partitioning = performance**  
   Columnar storage with smart partitioning reduced costs and improved speed

3. **Data quality must be automated**  
   Manual validation doesn't scale - build checks into pipeline

4. **Cross-cloud is achievable**  
   AWS + GCP integration works well with proper architecture

### Business Insights

1. **Unified data unlocks insights**  
   Combining sources revealed patterns invisible in silos

2. **Self-service requires infrastructure**  
   SQL access to clean data empowered non-technical teams

3. **Cloud flexibility beats on-prem rigidity**  
   Serverless handled unexpected load without manual intervention

---

## 🔄 Future Enhancements

Next steps if continuing this project:

- **Real-time streaming:** Kafka + Kinesis for event processing
- **Data mesh:** Federated ownership with centralized governance
- **ML integration:** Feature store for predictive models
- **Advanced monitoring:** Datadog for observability
- **Cost optimization:** Spot instances and reserved capacity

---

## 📚 Technologies Used

**Cloud Infrastructure:**
- AWS S3 (data lake storage)
- AWS Glue (ETL, data catalog)
- AWS Athena (serverless SQL)
- GCP BigQuery (data warehouse)

**Data Processing:**
- Python (orchestration, validation)
- PySpark (Glue transformations)
- SQL (analytics queries)

**Monitoring & Governance:**
- CloudWatch (logging, alerting)
- Glue Data Catalog (metadata)
- Custom validation framework

---

## 💡 Why This Project Matters

This platform transformed how the organization accessed and used data:

- **From siloed to unified:** Breaking down data barriers
- **From expensive to efficient:** 40% cost reduction with better performance
- **From slow to fast:** Minutes to seconds for insights
- **From rigid to flexible:** Serverless scaling with demand

**This is infrastructure that enables data-driven culture.**

---

[← Back to Portfolio](../README.md)
