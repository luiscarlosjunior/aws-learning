# Amazon Redshift

## O que é Redshift?

Amazon Redshift é um data warehouse na nuvem rápido, escalável e totalmente gerenciado que facilita a análise de grandes volumes de dados usando SQL e ferramentas de BI existentes. É otimizado para análise de dados (OLAP) com processamento massivamente paralelo (MPP).

## Arquitetura

### Cluster Architecture

```
┌─────────────────────────────────────────┐
│           Leader Node                    │
│  - Query planning                        │
│  - Client connections                    │
│  - Query distribution                    │
└──────────────┬──────────────────────────┘
               │
    ┌──────────┴──────────┐
    │                     │
┌───▼─────┐         ┌────▼────┐
│ Compute │         │ Compute │
│  Node 1 │   ...   │  Node N │
│         │         │         │
│  Slices │         │  Slices │
└─────────┘         └─────────┘
```

**Leader Node:**
- SQL parsing e query planning
- Coordena execução paralela
- Agrega resultados
- Client connection endpoint

**Compute Nodes:**
- Executam queries
- Armazenam dados
- Processamento paralelo
- Cada node tem múltiplos slices (CPUs)

### Node Types

**RA3 (Managed Storage):**
- Storage e compute separados
- Escalável independentemente
- S3-backed storage
- Cache local SSD
- Recomendado para maioria dos casos

**DC2 (Dense Compute):**
- SSD storage local
- High performance
- Fixed storage
- Menor custo por query

**Instance Sizes:**
- **ra3.xlplus**: 4 vCPU, 32 GB RAM, 32 TB managed storage
- **ra3.4xlarge**: 12 vCPU, 96 GB RAM, 128 TB managed storage
- **ra3.16xlarge**: 48 vCPU, 384 GB RAM, 128 TB managed storage
- **dc2.large**: 2 vCPU, 15 GB RAM, 160 GB SSD
- **dc2.8xlarge**: 32 vCPU, 244 GB RAM, 2.56 TB SSD

## Conceitos Principais

### Distribution Styles

Determina como dados são distribuídos entre nodes.

**KEY Distribution:**
```sql
CREATE TABLE orders (
    order_id INT,
    customer_id INT,
    amount DECIMAL
) DISTSTYLE KEY
DISTKEY (customer_id);
```
- Distribui por coluna específica
- Co-locate related data
- Minimize data movement em JOINs

**EVEN Distribution:**
```sql
CREATE TABLE logs (
    log_id BIGINT,
    message VARCHAR(500)
) DISTSTYLE EVEN;
```
- Round-robin distribution
- Use quando sem JOINs claros
- Balanceamento uniforme

**ALL Distribution:**
```sql
CREATE TABLE dim_dates (
    date_id INT,
    date DATE,
    year INT,
    month INT
) DISTSTYLE ALL;
```
- Copia tabela para todos nodes
- Use para tabelas pequenas e frequentes em JOINs
- Diminui data movement

**AUTO Distribution:**
```sql
CREATE TABLE facts (
    id INT,
    value DECIMAL
) DISTSTYLE AUTO;
```
- Redshift decide automaticamente
- Recomendado para novos projetos

### Sort Keys

Ordena dados no disco para melhor performance.

**Compound Sort Key:**
```sql
CREATE TABLE sales (
    date DATE,
    region VARCHAR(50),
    product_id INT,
    amount DECIMAL
) COMPOUND SORTKEY (date, region);
```
- Multi-column ordering
- Melhor para queries com prefix da chave
- Queries: `WHERE date = X` ou `WHERE date = X AND region = Y`

**Interleaved Sort Key:**
```sql
CREATE TABLE events (
    event_date DATE,
    user_id INT,
    event_type VARCHAR(50)
) INTERLEAVED SORTKEY (event_date, user_id, event_type);
```
- Equal weight para todas colunas
- Melhor para queries variadas
- Mais overhead de VACUUM

### Compression (Encoding)

Redshift comprime dados automaticamente.

**Tipos de encoding:**
- **RAW**: Sem compressão
- **AZ64**: Alta compressão para todos tipos
- **LZO**: Compressão geral
- **ZSTD**: Alta compressão
- **DELTA**: Para timestamps, incrementos
- **RUNLENGTH**: Valores repetidos

```sql
-- Ver encodings
SELECT 
    "column", 
    type, 
    encoding 
FROM pg_table_def 
WHERE tablename = 'orders';

-- Analyze compression
ANALYZE COMPRESSION orders;
```

## Redshift Spectrum

Query dados no S3 sem carregar no Redshift.

**Características:**
- Separa storage de compute
- Query petabytes no S3
- Formatos: Parquet, ORC, JSON, CSV, Avro
- Partitioning support
- Predicate pushdown

**Exemplo:**
```sql
-- Criar external schema
CREATE EXTERNAL SCHEMA spectrum_schema
FROM DATA CATALOG
DATABASE 'spectrumdb'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftSpectrumRole'
CREATE EXTERNAL DATABASE IF NOT EXISTS;

-- Criar external table
CREATE EXTERNAL TABLE spectrum_schema.sales_s3 (
    sale_id BIGINT,
    product_id INT,
    customer_id INT,
    amount DECIMAL(10,2),
    sale_date DATE
)
STORED AS PARQUET
LOCATION 's3://my-bucket/sales-data/'
TABLE PROPERTIES ('partition_filtering'='true');

-- Query dados S3 e Redshift juntos
SELECT 
    r.customer_name,
    s.sale_date,
    s.amount
FROM spectrum_schema.sales_s3 s
JOIN redshift_schema.customers r 
    ON s.customer_id = r.customer_id
WHERE s.sale_date >= '2024-01-01';
```

## Redshift Serverless

Executar analytics sem gerenciar clusters.

**Características:**
- Auto-scaling automático
- Pay per use (RPU-hours)
- Zero administration
- Start queries imediatamente
- Share data entre namespaces

**Conceitos:**
- **Namespace**: Coleção de objetos (schemas, tables, users)
- **Workgroup**: Compute resources
- **RPU (Redshift Processing Unit)**: Compute + memory

**Uso via Python:**
```python
import boto3

redshift_data = boto3.client('redshift-data')

# Execute query
response = redshift_data.execute_statement(
    WorkgroupName='my-workgroup',
    Database='dev',
    Sql='SELECT COUNT(*) FROM sales WHERE year = 2024'
)

query_id = response['Id']

# Check status
status = redshift_data.describe_statement(Id=query_id)
print(f"Status: {status['Status']}")

# Get results
if status['Status'] == 'FINISHED':
    result = redshift_data.get_statement_result(Id=query_id)
    for row in result['Records']:
        print(row)
```

## Integração e ETL

### COPY Command

Forma mais eficiente de carregar dados.

```sql
-- COPY de S3
COPY sales
FROM 's3://my-bucket/sales-data/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
FORMAT AS PARQUET;

-- COPY com compression
COPY orders
FROM 's3://my-bucket/orders/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
GZIP
DELIMITER '|'
IGNOREHEADER 1;

-- COPY de DynamoDB
COPY dynamodb_table
FROM 'dynamodb://ProductCatalog'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
READRATIO 50;

-- COPY com manifest
COPY events
FROM 's3://my-bucket/manifest.json'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
MANIFEST;
```

### UNLOAD Command

Exportar dados para S3.

```sql
-- UNLOAD para S3
UNLOAD ('SELECT * FROM sales WHERE year = 2024')
TO 's3://my-bucket/unload/sales-2024-'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftUnloadRole'
PARQUET
PARTITION BY (year);

-- UNLOAD com compressão
UNLOAD ('SELECT * FROM large_table')
TO 's3://my-bucket/export/'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftUnloadRole'
GZIP
PARALLEL ON
MAXFILESIZE 1 GB;
```

### AWS Glue Integration

```python
import boto3

glue = boto3.client('glue')

# Criar Glue job para ETL
response = glue.create_job(
    Name='s3-to-redshift-etl',
    Role='arn:aws:iam::123456789012:role/GlueServiceRole',
    Command={
        'Name': 'glueetl',
        'ScriptLocation': 's3://my-scripts/etl-script.py',
        'PythonVersion': '3'
    },
    DefaultArguments={
        '--TempDir': 's3://my-temp/',
        '--job-bookmark-option': 'job-bookmark-enable',
        '--redshift_connection': 'my-redshift-connection',
        '--target_table': 'public.sales'
    },
    MaxRetries=1,
    Timeout=60,
    GlueVersion='4.0'
)
```

## Performance Optimization

### Workload Management (WLM)

Gerencia recursos de query execution.

**Manual WLM:**
```json
{
  "query_concurrency": 5,
  "query_slots": [
    {
      "memory_percent_to_use": 30,
      "max_execution_time": 3600000,
      "user_group": "etl_group"
    },
    {
      "memory_percent_to_use": 40,
      "max_execution_time": 60000,
      "user_group": "dashboard_group"
    }
  ]
}
```

**Auto WLM (Recomendado):**
- Prioriza queries automaticamente
- Dynamic memory allocation
- Concurrency scaling automático
- Machine learning otimizations

### Concurrency Scaling

Adiciona clusters temporários para picos.

**Características:**
- Auto-scaling em segundos
- Transparente para aplicação
- 1 hora grátis por dia
- Read queries e writes

**Habilitar:**
```sql
-- Modificar WLM queue
ALTER WORKLOAD MANAGEMENT CONFIGURATION
ADD CONCURRENCY SCALING MODE AUTO
TO QUEUE my_queue;
```

### Query Performance

**Analyze Query Execution:**
```sql
-- Ver query execution plan
EXPLAIN
SELECT 
    p.product_name,
    SUM(s.amount) as total_sales
FROM sales s
JOIN products p ON s.product_id = p.product_id
WHERE s.sale_date >= '2024-01-01'
GROUP BY p.product_name;

-- System tables
SELECT 
    query,
    trim(querytxt) as sql,
    starttime,
    endtime,
    datediff(seconds, starttime, endtime) as duration
FROM stl_query
WHERE userid > 1
ORDER BY starttime DESC
LIMIT 10;
```

### Maintenance

**VACUUM:**
```sql
-- Recupera espaço e re-sort
VACUUM FULL table_name;

-- Apenas delete space
VACUUM DELETE ONLY table_name;

-- Apenas sort
VACUUM SORT ONLY table_name;
```

**ANALYZE:**
```sql
-- Atualiza estatísticas
ANALYZE table_name;

-- Analisar todas tabelas
ANALYZE;
```

## Segurança

### Network Isolation

```bash
# VPC-only cluster
aws redshift create-cluster \
    --cluster-identifier my-redshift \
    --node-type ra3.xlplus \
    --master-username admin \
    --master-user-password SecurePass123! \
    --vpc-security-group-ids sg-12345678 \
    --cluster-subnet-group-name my-subnet-group \
    --publicly-accessible false
```

### Encryption

**At Rest:**
```bash
# KMS encryption
aws redshift create-cluster \
    --cluster-identifier my-redshift \
    --encrypted \
    --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678
```

**In Transit:**
- SSL/TLS required
- Enforce SSL via parameter group

### IAM Integration

```sql
-- Criar user com IAM
CREATE USER john WITH PASSWORD DISABLE;

-- Grant role
GRANT ROLE analytics_role TO john;
```

**Python connection com IAM:**
```python
import boto3
import redshift_connector

# Get temporary credentials
client = boto3.client('redshift')

response = client.get_cluster_credentials(
    DbUser='iamuser',
    DbName='dev',
    ClusterIdentifier='my-redshift',
    DurationSeconds=3600
)

# Connect
conn = redshift_connector.connect(
    host='my-redshift.abc123.us-east-1.redshift.amazonaws.com',
    database='dev',
    user=response['DbUser'],
    password=response['DbPassword'],
    port=5439
)
```

## Exemplos Práticos

### 1. Python - Conectar e Query

```python
import redshift_connector

# Conexão
conn = redshift_connector.connect(
    host='my-cluster.abc123.us-east-1.redshift.amazonaws.com',
    database='dev',
    user='admin',
    password='SecurePass123!',
    port=5439
)

# Query
cursor = conn.cursor()
cursor.execute("""
    SELECT 
        date_trunc('month', sale_date) as month,
        SUM(amount) as total_sales,
        COUNT(*) as num_transactions
    FROM sales
    WHERE sale_date >= DATEADD(month, -6, GETDATE())
    GROUP BY 1
    ORDER BY 1
""")

# Fetch results
for row in cursor:
    print(f"Month: {row[0]}, Sales: ${row[1]:,.2f}, Transactions: {row[2]}")

cursor.close()
conn.close()
```

### 2. Pandas Integration

```python
import pandas as pd
import redshift_connector

conn = redshift_connector.connect(
    host='my-cluster.abc123.us-east-1.redshift.amazonaws.com',
    database='dev',
    user='admin',
    password='SecurePass123!',
    port=5439
)

# Query para DataFrame
df = pd.read_sql("""
    SELECT 
        product_category,
        SUM(sales_amount) as total_sales,
        AVG(sales_amount) as avg_sale
    FROM sales_fact sf
    JOIN product_dim pd ON sf.product_key = pd.product_key
    WHERE sale_date >= '2024-01-01'
    GROUP BY product_category
    ORDER BY total_sales DESC
""", conn)

print(df.head())

# DataFrame para Redshift
df_to_load = pd.DataFrame({
    'id': [1, 2, 3],
    'name': ['Alice', 'Bob', 'Charlie'],
    'value': [100, 200, 300]
})

# Via S3 (recomendado para grandes volumes)
import boto3
from io import StringIO

# Upload para S3
csv_buffer = StringIO()
df_to_load.to_csv(csv_buffer, index=False)
s3 = boto3.client('s3')
s3.put_object(
    Bucket='my-bucket',
    Key='temp/data.csv',
    Body=csv_buffer.getvalue()
)

# COPY para Redshift
cursor = conn.cursor()
cursor.execute("""
    COPY my_table
    FROM 's3://my-bucket/temp/data.csv'
    IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
    CSV
    IGNOREHEADER 1
""")
conn.commit()

conn.close()
```

### 3. Data Warehouse Schema (Star Schema)

```sql
-- Dimension Tables
CREATE TABLE dim_customer (
    customer_key INT IDENTITY(1,1) PRIMARY KEY,
    customer_id VARCHAR(50) NOT NULL,
    customer_name VARCHAR(200),
    email VARCHAR(200),
    segment VARCHAR(50),
    region VARCHAR(50),
    created_date DATE,
    updated_date TIMESTAMP
) DISTSTYLE ALL
SORTKEY (customer_key);

CREATE TABLE dim_product (
    product_key INT IDENTITY(1,1) PRIMARY KEY,
    product_id VARCHAR(50) NOT NULL,
    product_name VARCHAR(200),
    category VARCHAR(100),
    subcategory VARCHAR(100),
    brand VARCHAR(100),
    unit_price DECIMAL(10,2)
) DISTSTYLE ALL
SORTKEY (product_key);

CREATE TABLE dim_date (
    date_key INT PRIMARY KEY,
    date DATE NOT NULL,
    year INT,
    quarter INT,
    month INT,
    week INT,
    day_of_week INT,
    day_name VARCHAR(10),
    is_weekend BOOLEAN,
    is_holiday BOOLEAN
) DISTSTYLE ALL
SORTKEY (date_key);

-- Fact Table
CREATE TABLE fact_sales (
    sale_id BIGINT IDENTITY(1,1),
    date_key INT NOT NULL,
    customer_key INT NOT NULL,
    product_key INT NOT NULL,
    quantity INT NOT NULL,
    unit_price DECIMAL(10,2) NOT NULL,
    discount_percent DECIMAL(5,2),
    tax_amount DECIMAL(10,2),
    total_amount DECIMAL(12,2) NOT NULL,
    created_at TIMESTAMP DEFAULT GETDATE()
) DISTSTYLE KEY
DISTKEY (customer_key)
COMPOUND SORTKEY (date_key, customer_key);

-- Analytical Queries
-- Top customers por revenue
SELECT 
    c.customer_name,
    c.segment,
    SUM(f.total_amount) as total_revenue,
    COUNT(DISTINCT f.date_key) as purchase_days,
    AVG(f.total_amount) as avg_order_value
FROM fact_sales f
JOIN dim_customer c ON f.customer_key = c.customer_key
JOIN dim_date d ON f.date_key = d.date_key
WHERE d.year = 2024
GROUP BY c.customer_name, c.segment
HAVING SUM(f.total_amount) > 10000
ORDER BY total_revenue DESC
LIMIT 100;

-- Monthly trends por categoria
SELECT 
    d.year,
    d.month,
    p.category,
    SUM(f.quantity) as units_sold,
    SUM(f.total_amount) as revenue,
    COUNT(DISTINCT f.customer_key) as unique_customers
FROM fact_sales f
JOIN dim_product p ON f.product_key = p.product_key
JOIN dim_date d ON f.date_key = d.date_key
GROUP BY d.year, d.month, p.category
ORDER BY d.year, d.month, p.category;
```

### 4. ETL Pipeline com Airflow

```python
from airflow import DAG
from airflow.providers.amazon.aws.operators.s3 import S3Hook
from airflow.providers.amazon.aws.transfers.redshift_to_s3 import RedshiftToS3Operator
from airflow.providers.amazon.aws.operators.redshift_data import RedshiftDataOperator
from datetime import datetime, timedelta

default_args = {
    'owner': 'analytics',
    'depends_on_past': False,
    'start_date': datetime(2024, 1, 1),
    'email_on_failure': True,
    'email_on_retry': False,
    'retries': 2,
    'retry_delay': timedelta(minutes=5),
}

dag = DAG(
    's3_to_redshift_etl',
    default_args=default_args,
    description='Load data from S3 to Redshift',
    schedule_interval='0 2 * * *',  # Daily at 2 AM
    catchup=False
)

# Criar staging table
create_staging = RedshiftDataOperator(
    task_id='create_staging_table',
    cluster_identifier='my-redshift',
    database='dev',
    db_user='admin',
    sql="""
        DROP TABLE IF EXISTS staging_sales;
        CREATE TABLE staging_sales (
            sale_id VARCHAR(50),
            customer_id VARCHAR(50),
            product_id VARCHAR(50),
            amount DECIMAL(10,2),
            sale_date DATE
        );
    """,
    dag=dag
)

# COPY de S3 para staging
copy_to_staging = RedshiftDataOperator(
    task_id='copy_to_staging',
    cluster_identifier='my-redshift',
    database='dev',
    db_user='admin',
    sql="""
        COPY staging_sales
        FROM 's3://my-bucket/daily-sales/{{ ds }}/'
        IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftCopyRole'
        FORMAT AS PARQUET;
    """,
    dag=dag
)

# Transform e load para produção
transform_load = RedshiftDataOperator(
    task_id='transform_and_load',
    cluster_identifier='my-redshift',
    database='dev',
    db_user='admin',
    sql="""
        BEGIN TRANSACTION;
        
        -- Insert new records
        INSERT INTO fact_sales (date_key, customer_key, product_key, quantity, total_amount)
        SELECT 
            TO_NUMBER(TO_CHAR(s.sale_date, 'YYYYMMDD'), '99999999') as date_key,
            c.customer_key,
            p.product_key,
            1 as quantity,
            s.amount
        FROM staging_sales s
        JOIN dim_customer c ON s.customer_id = c.customer_id
        JOIN dim_product p ON s.product_id = p.product_id
        WHERE NOT EXISTS (
            SELECT 1 FROM fact_sales f 
            WHERE f.sale_id = s.sale_id
        );
        
        -- Cleanup staging
        DROP TABLE staging_sales;
        
        END TRANSACTION;
        
        -- Maintenance
        VACUUM fact_sales;
        ANALYZE fact_sales;
    """,
    dag=dag
)

create_staging >> copy_to_staging >> transform_load
```

### 5. Materialized Views

```sql
-- Criar materialized view
CREATE MATERIALIZED VIEW mv_monthly_sales AS
SELECT 
    DATE_TRUNC('month', sale_date) as month,
    customer_id,
    SUM(amount) as total_sales,
    COUNT(*) as num_transactions,
    AVG(amount) as avg_transaction
FROM sales
GROUP BY 1, 2;

-- Refresh materialized view
REFRESH MATERIALIZED VIEW mv_monthly_sales;

-- Auto-refresh
CREATE MATERIALIZED VIEW mv_daily_summary
AUTO REFRESH YES
AS
SELECT 
    sale_date,
    COUNT(*) as num_sales,
    SUM(amount) as total_amount
FROM sales
GROUP BY sale_date;
```

## Melhores Práticas

### Data Modeling
1. **Use distribution keys** para co-locate related data
2. **Sort keys** para colunas frequentes em WHERE/ORDER BY
3. **Compression** (deixe Redshift decidir com COPY)
4. **Star schema** para data warehouses
5. **Avoid DISTYLE ALL** para tabelas grandes

### Performance
1. **Use COPY** (não INSERT individual)
2. **Batch operations** quando possível
3. **Auto WLM** habilitado
4. **Concurrency Scaling** para workloads variáveis
5. **Redshift Spectrum** para dados raramente acessados
6. **Regular VACUUM e ANALYZE**
7. **Materialized views** para queries frequentes

### Custo
1. **Reserved Instances** (1-3 anos, até 75% desconto)
2. **Pause/resume** para dev/test
3. **RA3 instances** para flexibility
4. **Redshift Spectrum** para archived data
5. **Short query acceleration** (SQA) para dashboards
6. **Delete old data** regularmente

### Segurança
1. **VPC deployment** apenas
2. **Encryption at rest** (KMS) e in transit (SSL)
3. **IAM roles** para COPY/UNLOAD
4. **Row-level security** quando necessário
5. **Audit logging** enabled
6. **Regular snapshots** (automated + manual)

## Limitações

- Max cluster size: 2 PB (RA3)
- Max databases: 60 per cluster
- Max tables: 9,900 per database
- Max columns: 1,600 per table
- Max concurrent connections: 500 (pode aumentar)
- Sort keys: 400 columns max
- Distribution keys: 1 column

## Perguntas e Respostas

### P: Quando usar Redshift vs RDS?
**R:** Use Redshift para analytics/BI (OLAP) com queries complexas em grandes datasets. Use RDS para transactional workloads (OLTP) com muitos writes pequenos e reads rápidos. Redshift otimizado para scans de grandes volumes, RDS para queries de linhas específicas.

### P: O que é Redshift Spectrum?
**R:** Permite query dados no S3 sem carregar no Redshift. Use para dados frios/archived, queries ad-hoc, ou data lake integration. Paga apenas pelo data scanned. Suporta Parquet, ORC, JSON, CSV.

### P: Como escolher distribution key?
**R:** Escolha coluna usada em JOINs frequentes e com alta cardinalidade. Objetivo: co-locate related data no mesmo node para evitar data movement. Para tabelas pequenas (< 1M rows), use DISTSTYLE ALL.

### P: VACUUM é necessário?
**R:** Sim, após DELETEs/UPDATEs pesados para recuperar espaço e re-sort data. Redshift tem auto-vacuum background mas pode precisar manual VACUUM FULL para casos extremos. Monitor via system tables.

### P: Como melhorar query performance?
**R:** 1) Use EXPLAIN para ver plan, 2) Appropriate distribution/sort keys, 3) ANALYZE atualizado, 4) Evite SELECT *, 5) Use WHERE para predicate pushdown, 6) Consider materialized views, 7) Monitor system tables (STL_QUERY, SVL_QUERY_SUMMARY).

### P: Redshift Serverless vs Provisioned?
**R:** Serverless: auto-scaling, pay per use, zero admin, ideal para workloads variáveis ou imprevisíveis. Provisioned: custo previsível, controle sobre resources, better para steady workloads. Serverless pode ser mais caro para heavy constant use.

### P: Como carregar dados efficiently?
**R:** Use COPY command de S3 (paralelizado automaticamente). Comprima arquivos (GZIP). Use columnar formats (Parquet, ORC). Split em múltiplos arquivos. Evite INSERT individual. Para streams, use Kinesis Firehose.

### P: O que é WLM?
**R:** Workload Management: gerencia query queues, memory allocation, concurrency. Auto WLM (recomendado) usa ML para otimizar. Manual WLM para controle granular. Configure priorities para diferentes workloads (ETL, BI, ad-hoc).

### P: Posso pausar Redshift?
**R:** Sim, pause/resume para dev/test clusters economiza compute cost (continua pagando storage). Serverless pausa automaticamente quando idle. Production clusters geralmente sempre ativos.

### P: Como funciona Concurrency Scaling?
**R:** Adiciona clusters temporários automaticamente para absorver picos de read queries e writes. 1 hora grátis por dia. Queries roteadas transparentemente. Habilite por WLM queue.

### P: Redshift suporta transactions?
**R:** Sim, ACID compliant. Use BEGIN/COMMIT/ROLLBACK. Mas otimizado para batch operations, não high-frequency transactions como RDS.

### P: Como fazer disaster recovery?
**R:** Automated snapshots (8 hours retention, configurable até 35 dias). Manual snapshots (retention indefinida). Copy snapshots para outras regiões. Restore cria novo cluster. Test DR procedures regularmente.

### P: Qual melhor formato para S3?
**R:** Parquet ou ORC (columnar) para analytics. 10x melhor compression e performance vs CSV/JSON. Partition por colunas frequentes em WHERE (ex: year, month). Use Glue para conversion.

### P: Como monitorar Redshift?
**R:** CloudWatch metrics (CPUUtilization, DatabaseConnections, PercentageDiskSpaceUsed). System tables (STL_QUERY para queries, SVL_S3QUERY para Spectrum). Query monitoring rules (QMR) para abort long-running queries. Performance Insights.

### P: Redshift vs Athena?
**R:** Redshift: warm data, consistent performance, complex queries, high concurrency, managed cluster. Athena: cold data, serverless, pay per query, ad-hoc analysis, no infrastructure. Combine: use Athena para EDA, load results para Redshift.

### P: Como lidar com slow queries?
**R:** 1) Check EXPLAIN plan, 2) Verify table statistics (ANALYZE), 3) Review distribution/sort keys, 4) Add WHERE clauses, 5) Use columnar formats, 6) Consider materialized views, 7) Enable result caching, 8) Check disk space (VACUUM if needed).

### P: Sort key compound vs interleaved?
**R:** Compound: melhor quando queries usam prefix do sort key (date, então region). Interleaved: igual weight para todas colunas, bom para queries variadas. Compound mais comum e menos overhead de VACUUM.

### P: Como integrar com BI tools?
**R:** JDBC/ODBC drivers para Tableau, Power BI, Looker, Quicksight. Connection pooling recomendado. Use materialized views para dashboards. Consider result caching. Short Query Acceleration para interactive queries.

### P: Redshift Machine Learning?
**R:** CREATE MODEL com SQL para treinar modelos usando SageMaker Autopilot. Inference dentro do Redshift. Use cases: churn prediction, recommendations, forecasting. Simple SQL interface, no ML expertise needed.

### P: Como migrar de on-premise para Redshift?
**R:** 1) Schema Conversion Tool (SCT) para converter DDL, 2) AWS DMS para migração contínua, 3) COPY de S3, 4) Snowball para grandes volumes (>10TB), 5) Test queries e performance, 6) Cutover planejado.

## Recursos de Aprendizado

- [Redshift Documentation](https://docs.aws.amazon.com/redshift/)
- [Redshift Best Practices](https://docs.aws.amazon.com/redshift/latest/dg/best-practices.html)
- [Redshift SQL Reference](https://docs.aws.amazon.com/redshift/latest/dg/c_SQL_reference.html)
- [AWS re:Invent Redshift Sessions](https://www.youtube.com/results?search_query=reinvent+redshift)
- [Redshift Pricing](https://aws.amazon.com/redshift/pricing/)
