# Analytics

## Visão Geral
Os serviços de analytics da AWS permitem processar, analisar e visualizar grandes volumes de dados para obter insights de negócio.

## Serviços

### Athena
Serviço de query interativo para analisar dados no S3 usando SQL.

**Recursos principais:**
- Serverless (sem infraestrutura)
- Suporte a SQL padrão
- Query dados em S3 diretamente
- Integration com QuickSight
- Pay per query (dados scanned)
- Federated queries

**Formatos Suportados:**
- CSV, JSON, Apache Parquet, Apache ORC
- Avro, Apache Hudi, Delta Lake

**Casos de Uso:**
- Ad-hoc querying
- Log analysis
- Data exploration
- Business intelligence

### EMR (Elastic MapReduce)
Plataforma de big data gerenciada.

**Frameworks Suportados:**
- Apache Hadoop
- Apache Spark
- Apache HBase
- Apache Hive
- Apache Flink
- Presto

**Recursos principais:**
- Auto scaling
- Spot instances support
- EMR Notebooks
- EMR Studio
- Multiple storage (S3, HDFS, EBS)

**Casos de Uso:**
- Big data processing
- Machine learning
- ETL workflows
- Data transformation
- Real-time streaming

### Kinesis
Processamento de streaming data em tempo real.

**Componentes:**
- **Kinesis Data Streams**: Coleta e processa streams
- **Kinesis Data Firehose**: Carrega streams para destinos
- **Kinesis Data Analytics**: Analisa streams com SQL
- **Kinesis Video Streams**: Processa video streams

**Recursos principais:**
- Real-time processing
- Automatic scaling
- Data retention (24h a 365 dias)
- Multiple consumers
- Integration com Lambda, EMR, S3

**Casos de Uso:**
- Real-time analytics
- Log aggregation
- IoT telemetry
- Clickstream analysis

### Glue
Serviço ETL serverless e data catalog.

**Componentes:**
- **Glue Data Catalog**: Metadata repository
- **Glue Crawlers**: Schema discovery
- **Glue ETL Jobs**: Data transformation (Spark/Python)
- **Glue DataBrew**: Visual data preparation
- **Glue Schema Registry**: Schema versioning

**Recursos principais:**
- Serverless
- Auto-scaling
- Job scheduling
- Development endpoints
- Integration com Athena, EMR, Redshift

**Casos de Uso:**
- ETL pipelines
- Data catalog
- Schema evolution
- Data quality

### QuickSight
Business Intelligence (BI) serverless.

**Recursos principais:**
- Interactive dashboards
- ML-powered insights
- Embedded analytics
- SPICE in-memory engine
- Mobile support
- Pay per session

**Data Sources:**
- AWS services (S3, Athena, RDS, Redshift, etc.)
- Third-party databases
- SaaS applications
- Files (Excel, CSV)

**Casos de Uso:**
- Business dashboards
- Ad-hoc analysis
- Embedded analytics
- Mobile BI

### Data Pipeline
Orquestração de data workflows (legacy).

**Nota**: Considere usar AWS Glue, Step Functions ou MWAA para novos projetos.

**Recursos principais:**
- Data movement
- ETL workflows
- Scheduled pipelines
- On-premises integration

### Managed Streaming for Apache Kafka (MSK)
Apache Kafka totalmente gerenciado.

**Recursos principais:**
- Kafka compatível 100%
- Multi-AZ deployment
- Automatic patching
- Monitoring com CloudWatch
- MSK Connect (Kafka Connect)
- MSK Serverless

**Casos de Uso:**
- Event streaming
- Log aggregation
- Real-time analytics
- Microservices messaging

## Pipeline de Analytics Típico

```
Data Sources
    ↓
Ingestion (Kinesis/MSK)
    ↓
Storage (S3 Data Lake)
    ↓
Catalog (Glue Data Catalog)
    ↓
Processing (EMR/Glue/Athena)
    ↓
Visualization (QuickSight)
```

## Data Lake Architecture

```
Raw Zone (S3)
├── Ingestion via Kinesis Firehose
└── Data Catalog via Glue Crawlers

Processed Zone (S3)
├── Transformation via Glue ETL
└── Optimized formats (Parquet/ORC)

Curated Zone (S3)
├── Business-ready data
└── Query via Athena
    ↓
Visualization via QuickSight
```

## Comparação de Serviços

| Serviço | Tipo | Uso | Latência | Custo |
|---------|------|-----|----------|-------|
| Athena | Interactive Query | Ad-hoc SQL | Segundos | Pay per query |
| EMR | Big Data Platform | Batch processing | Minutos | Pay per cluster |
| Kinesis | Streaming | Real-time | Milissegundos | Pay per hour |
| Glue | ETL | Data transformation | Minutos | Pay per DPU-hour |
| QuickSight | BI | Visualization | Segundos | Pay per session |
| MSK | Message Broker | Event streaming | Milissegundos | Pay per hour |

## Escolhendo o Serviço Certo

### Use Athena quando:
- Ad-hoc queries em S3
- Análise exploratória
- Não precisa de infraestrutura
- Queries ocasionais

### Use EMR quando:
- Processamento complexo de big data
- Frameworks específicos (Spark, Hadoop)
- Machine learning em larga escala
- ETL de alto volume

### Use Kinesis quando:
- Streaming data real-time
- IoT telemetry
- Log aggregation
- Clickstream analysis

### Use Glue quando:
- ETL serverless
- Data catalog
- Schema discovery
- Simple transformations

### Use MSK quando:
- Kafka workloads
- Event streaming
- Migração de Kafka on-premises

## Melhores Práticas

### Athena
- Use columnar formats (Parquet/ORC)
- Partition data
- Compress data
- Use appropriate data types
- Limit scanned data
- Use CTAS para optimization

### EMR
- Use Spot Instances para task nodes
- Enable S3 as storage
- Configure auto-scaling
- Use EMR Studio para development
- Implement cost allocation tags
- Right-size clusters

### Kinesis
- Choose appropriate shard count
- Monitor iterator age
- Implement error handling
- Use enhanced fan-out quando necessário
- Batch records para efficiency
- Implement checkpointing

### Glue
- Use job bookmarks
- Partition output data
- Use development endpoints para testing
- Monitor job metrics
- Implement data quality checks
- Use Glue DataBrew para visual prep

### QuickSight
- Use SPICE quando possível
- Optimize data sources
- Implement row-level security
- Schedule refresh appropriadamente
- Share dashboards eficientemente

### Geral
- Design for cost optimization
- Implement data lifecycle policies
- Use appropriate data formats
- Monitor costs e usage
- Implement data governance
- Test performance regularly

## Exemplo: Athena Query

```sql
-- Create external table
CREATE EXTERNAL TABLE IF NOT EXISTS logs (
  timestamp STRING,
  user_id STRING,
  action STRING,
  response_time INT
)
PARTITIONED BY (year INT, month INT, day INT)
STORED AS PARQUET
LOCATION 's3://my-bucket/logs/';

-- Query with partition
SELECT 
  action,
  AVG(response_time) as avg_response
FROM logs
WHERE year = 2024 AND month = 1
GROUP BY action
ORDER BY avg_response DESC;
```

## Exemplo: Glue ETL Job (PySpark)

```python
import sys
from awsglue.transforms import *
from awsglue.utils import getResolvedOptions
from pyspark.context import SparkContext
from awsglue.context import GlueContext
from awsglue.job import Job

args = getResolvedOptions(sys.argv, ['JOB_NAME'])
sc = SparkContext()
glueContext = GlueContext(sc)
spark = glueContext.spark_session
job = Job(glueContext)
job.init(args['JOB_NAME'], args)

# Read from Glue Data Catalog
datasource = glueContext.create_dynamic_frame.from_catalog(
    database = "my_database",
    table_name = "my_table"
)

# Transform
transformed = ApplyMapping.apply(
    frame = datasource,
    mappings = [
        ("id", "string", "user_id", "string"),
        ("timestamp", "string", "event_time", "timestamp")
    ]
)

# Write to S3 in Parquet
glueContext.write_dynamic_frame.from_options(
    frame = transformed,
    connection_type = "s3",
    connection_options = {"path": "s3://output-bucket/data"},
    format = "parquet"
)

job.commit()
```

## Recursos de Aprendizado

- [AWS Analytics Services](https://aws.amazon.com/big-data/datalakes-and-analytics/)
- [Athena Best Practices](https://docs.aws.amazon.com/athena/latest/ug/best-practices.html)
- [Big Data on AWS](https://aws.amazon.com/big-data/)
- [Data Lakes on AWS](https://aws.amazon.com/solutions/implementations/data-lake-solution/)
- [AWS Analytics Workshops](https://workshops.aws/categories/Analytics)
