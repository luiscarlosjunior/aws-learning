# Amazon Athena

## O que é Athena?

Amazon Athena é um serviço de consulta interativa serverless que facilita a análise de dados diretamente no Amazon S3 usando SQL padrão. Não há necessidade de configurar ou gerenciar infraestrutura, e você paga apenas pelas consultas executadas.

## Conceitos Principais

### Serverless Query Engine
- **Sem Infraestrutura**: Não há servidores para gerenciar
- **Escalabilidade Automática**: Escala automaticamente baseado na carga
- **Pay-per-Query**: Paga apenas pelos dados escaneados
- **SQL Padrão**: Usa ANSI SQL para consultas

### Data Sources
- **Amazon S3**: Fonte de dados principal
- **Federated Queries**: Consulta dados de múltiplas fontes
- **AWS Glue Data Catalog**: Metadata central
- **External Connectors**: RDS, DynamoDB, CloudWatch Logs, etc.

### Suporte a Formatos
- **Columnar**: Parquet, ORC (recomendados)
- **Row-based**: CSV, TSV, JSON, Apache Avro
- **Compressed**: GZIP, SNAPPY, ZLIB, LZO
- **Outros**: Apache Hudi, Delta Lake, Iceberg

## Arquitetura

### Como Funciona

```
User/Application
    ↓
Athena Query
    ↓
AWS Glue Data Catalog (schema)
    ↓
S3 Data (raw files)
    ↓
Query Results → S3 bucket
```

### Componentes

1. **Query Engine**: Baseado em Presto
2. **Data Catalog**: AWS Glue Data Catalog
3. **Storage**: Amazon S3
4. **Results**: Armazenados em S3

## Recursos Principais

### 1. Presto-Based Engine
- Motor de query distribuído
- Suporte a ANSI SQL
- Join de múltiplas tabelas
- Window functions
- Array e Map operations

### 2. Integration com Glue
- Schema discovery automático
- Metadata management
- Crawlers para atualização de schema
- Partition detection

### 3. Federated Queries
- Query múltiplas fontes de dados
- Data source connectors
- Join entre diferentes sources
- Lambda-based connectors

### 4. Performance Optimization
- **Partitioning**: Melhora performance e reduz custos
- **Compression**: Reduz dados escaneados
- **Columnar formats**: Parquet/ORC para melhor performance
- **CTAS**: Create Table As Select para otimização

### 5. Security
- **Encryption**: Em trânsito e em repouso
- **IAM**: Controle de acesso granular
- **Lake Formation**: Data governance
- **VPC Endpoints**: Acesso privado

## Casos de Uso

### 1. Ad-Hoc Analysis
Análise exploratória de dados sem necessidade de ETL prévio.

```sql
SELECT 
    date,
    COUNT(*) as total_events,
    AVG(response_time) as avg_response
FROM cloudfront_logs
WHERE date >= '2024-01-01'
GROUP BY date
ORDER BY date;
```

### 2. Log Analysis
Análise de logs de aplicações, CloudFront, VPC Flow Logs, etc.

```sql
-- CloudFront log analysis
SELECT 
    c_ip,
    COUNT(*) as requests,
    SUM(sc_bytes) as total_bytes
FROM cloudfront_logs
WHERE status >= 400
GROUP BY c_ip
ORDER BY requests DESC
LIMIT 10;
```

### 3. Business Intelligence
Dados para dashboards e relatórios em QuickSight.

### 4. Data Lake Queries
Consultas em data lakes hospedados no S3.

### 5. Cost Analysis
Análise de custos AWS usando CUR (Cost and Usage Report).

```sql
SELECT 
    line_item_product_code,
    SUM(line_item_unblended_cost) as cost
FROM cur_table
WHERE year = '2024' AND month = '01'
GROUP BY line_item_product_code
ORDER BY cost DESC;
```

## Melhores Práticas

### Performance

#### 1. Use Columnar Formats
```sql
-- Converter CSV para Parquet usando CTAS
CREATE TABLE logs_parquet
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    external_location = 's3://bucket/logs-parquet/'
)
AS SELECT * FROM logs_csv;
```

#### 2. Partition Data
```sql
-- Criar tabela particionada
CREATE EXTERNAL TABLE logs (
    request_id STRING,
    user_id STRING,
    action STRING
)
PARTITIONED BY (
    year INT,
    month INT,
    day INT
)
STORED AS PARQUET
LOCATION 's3://bucket/logs/';

-- Query com partition pruning
SELECT * FROM logs
WHERE year = 2024 AND month = 1 AND day = 15;
```

#### 3. Compress Data
- Use SNAPPY, GZIP ou ZLIB
- Balance entre compression ratio e query performance
- SNAPPY é recomendado para Parquet/ORC

#### 4. Optimize Data Types
```sql
-- Use tipos apropriados
CREATE TABLE optimized (
    id BIGINT,              -- não use STRING para números
    timestamp TIMESTAMP,    -- não use STRING para datas
    amount DECIMAL(10,2),   -- não use DOUBLE para valores monetários
    is_active BOOLEAN       -- não use STRING para booleans
);
```

### Cost Optimization

#### 1. Limit Data Scanned
```sql
-- Especifique colunas necessárias
SELECT id, name, email  -- ao invés de SELECT *
FROM users;

-- Use WHERE para filtrar
SELECT * FROM logs
WHERE year = 2024 AND month = 1;
```

#### 2. Use CTAS para Pre-aggregate
```sql
-- Criar tabela agregada para queries recorrentes
CREATE TABLE daily_metrics
AS SELECT 
    date,
    COUNT(*) as events,
    SUM(amount) as total
FROM raw_events
GROUP BY date;
```

#### 3. Enable Query Result Reuse
- Cache de resultados por 24 horas
- Queries idênticas retornam resultados cached
- Sem custo adicional

### Security

#### 1. IAM Policies
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "athena:StartQueryExecution",
                "athena:GetQueryExecution",
                "athena:GetQueryResults"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-data-bucket/*",
                "arn:aws:s3:::my-data-bucket"
            ]
        }
    ]
}
```

#### 2. Encryption
```sql
-- Habilitar encryption para query results
-- Configurado no Athena workgroup
-- Suporte a SSE-S3, SSE-KMS, CSE-KMS
```

#### 3. Lake Formation
- Controle de acesso granular
- Row-level e column-level security
- Data filtering

## Workgroups

Workgroups permitem isolar queries e controlar custos.

### Recursos de Workgroups
- **Cost Controls**: Limites de data scanned
- **Query Metrics**: Monitoramento por workgroup
- **Result Location**: Separação de resultados
- **IAM Policies**: Controle de acesso
- **Encryption**: Configuração por workgroup

### Exemplo
```bash
# Criar workgroup via CLI
aws athena create-work-group \
    --name analytics-team \
    --configuration \
        ResultConfigurationUpdates={
            OutputLocation=s3://results-bucket/analytics/
        },\
        EnforceWorkGroupConfiguration=true,\
        BytesScannedCutoffPerQuery=10000000000
```

## Federated Queries

Consulte dados fora do S3 usando Data Source Connectors.

### Connectors Disponíveis
- Amazon RDS (MySQL, PostgreSQL)
- Amazon Redshift
- Amazon DynamoDB
- Amazon DocumentDB
- Amazon CloudWatch
- Amazon CloudWatch Metrics
- Custom connectors via Lambda

### Exemplo
```sql
-- Registrar data source
-- (feito via Console ou CLI)

-- Query federada
SELECT 
    s3_table.user_id,
    s3_table.action,
    rds_table.user_name,
    rds_table.email
FROM 
    "s3_catalog"."default"."user_events" s3_table
JOIN 
    "rds_connector"."mydb"."users" rds_table
ON s3_table.user_id = rds_table.id
WHERE s3_table.date = '2024-01-15';
```

## Exemplos Práticos

### 1. Criar Database
```sql
CREATE DATABASE IF NOT EXISTS analytics;
```

### 2. Criar Tabela Externa
```sql
CREATE EXTERNAL TABLE IF NOT EXISTS analytics.cloudtrail_logs (
    eventVersion STRING,
    userIdentity STRUCT<
        type: STRING,
        principalId: STRING,
        arn: STRING,
        accountId: STRING,
        userName: STRING
    >,
    eventTime STRING,
    eventSource STRING,
    eventName STRING,
    awsRegion STRING,
    sourceIPAddress STRING,
    userAgent STRING,
    requestParameters STRING,
    responseElements STRING
)
PARTITIONED BY (
    year STRING,
    month STRING,
    day STRING
)
ROW FORMAT SERDE 'com.amazon.emr.hive.serde.CloudTrailSerde'
STORED AS INPUTFORMAT 'com.amazon.emr.cloudtrail.CloudTrailInputFormat'
OUTPUTFORMAT 'org.apache.hadoop.hive.ql.io.HiveIgnoreKeyTextOutputFormat'
LOCATION 's3://my-cloudtrail-bucket/AWSLogs/123456789012/CloudTrail/';
```

### 3. Adicionar Partitions
```sql
-- Manual
ALTER TABLE cloudtrail_logs ADD
    PARTITION (year='2024', month='01', day='15')
    LOCATION 's3://my-cloudtrail-bucket/AWSLogs/123456789012/CloudTrail/2024/01/15/';

-- Ou usar MSCK REPAIR para auto-discovery
MSCK REPAIR TABLE cloudtrail_logs;
```

### 4. Query com Window Functions
```sql
SELECT 
    user_id,
    action,
    timestamp,
    LAG(timestamp) OVER (
        PARTITION BY user_id 
        ORDER BY timestamp
    ) as previous_action_time
FROM user_events
WHERE date = '2024-01-15';
```

### 5. Complex Aggregations
```sql
SELECT 
    region,
    product,
    SUM(revenue) as total_revenue,
    AVG(revenue) as avg_revenue,
    PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY revenue) as median_revenue
FROM sales
WHERE year = 2024
GROUP BY region, product
HAVING SUM(revenue) > 10000
ORDER BY total_revenue DESC;
```

## Comandos Úteis (AWS CLI)

```bash
# Executar query
aws athena start-query-execution \
    --query-string "SELECT * FROM my_table LIMIT 10" \
    --query-execution-context Database=my_database \
    --result-configuration OutputLocation=s3://my-results-bucket/

# Verificar status da query
aws athena get-query-execution --query-execution-id <query-id>

# Obter resultados
aws athena get-query-results --query-execution-id <query-id>

# Listar databases
aws athena list-databases --catalog-name AwsDataCatalog

# Listar tabelas
aws athena list-table-metadata \
    --catalog-name AwsDataCatalog \
    --database-name my_database

# Criar workgroup
aws athena create-work-group \
    --name my-workgroup \
    --configuration ResultConfigurationUpdates={OutputLocation=s3://results/}

# Listar workgroups
aws athena list-work-groups
```

## Integration com Outros Serviços

### QuickSight
- Visualização de resultados Athena
- Dashboards interativos
- SPICE para performance

### AWS Glue
- Data Catalog para metadata
- Crawlers para schema discovery
- ETL jobs para data preparation

### Lambda
- Automatizar queries
- Event-driven analytics
- Custom processing

### Step Functions
- Orquestrar workflows de analytics
- Coordenar múltiplas queries
- Error handling

## Pricing

### Modelo de Custo
- **Pay-per-query**: $5.00 por TB de dados escaneados
- **Minimum**: $0.10 por query (10 MB)
- **Workgroup limits**: Opcional
- **DDL queries**: Gratuitas (CREATE, ALTER, DROP)
- **Failed queries**: Sem cobrança

### Exemplo de Cálculo
```
Query escaneia 100 GB = 0.1 TB
Custo = 0.1 TB × $5.00 = $0.50
```

### Reduzir Custos
1. Use Parquet/ORC (reduz até 80% de custos)
2. Particione dados
3. Comprima dados
4. Especifique colunas em SELECT
5. Use CTAS para pre-aggregate
6. Cache de query results

## Troubleshooting

### Query Lenta
1. Verificar se está usando Parquet/ORC
2. Adicionar partitions
3. Reduzir dados escaneados
4. Verificar joins complexos
5. Usar EXPLAIN para query plan

### "Table not found"
1. Verificar database/table name
2. Rodar MSCK REPAIR TABLE
3. Verificar permissões S3
4. Verificar Glue Data Catalog

### "Access Denied"
1. Verificar IAM permissions
2. Verificar S3 bucket policy
3. Verificar Lake Formation permissions
4. Verificar workgroup permissions

### "Exceeded limit"
1. Verificar workgroup limits
2. Otimizar query para escanear menos dados
3. Particionar dados
4. Usar compression

## Comparação com Outras Soluções

| Feature | Athena | Redshift | EMR | RDS |
|---------|--------|----------|-----|-----|
| Tipo | Serverless Query | Data Warehouse | Big Data Platform | Database |
| Setup | Nenhum | Cluster | Cluster | Instance |
| Custo | Pay-per-query | Pay-per-hour | Pay-per-hour | Pay-per-hour |
| Use Case | Ad-hoc queries | Complex analytics | Custom processing | OLTP |
| Latência | Segundos | Sub-segundo | Minutos | Milissegundos |
| Storage | S3 | Managed | S3/HDFS | EBS |

## Recursos de Aprendizado

### Documentação Oficial
- [Athena User Guide](https://docs.aws.amazon.com/athena/)
- [SQL Reference](https://docs.aws.amazon.com/athena/latest/ug/ddl-sql-reference.html)
- [Best Practices](https://docs.aws.amazon.com/athena/latest/ug/performance-tuning.html)

### Workshops e Tutoriais
- [Athena Workshop](https://athena-in-action.workshop.aws/)
- [Query S3 data with Athena](https://aws.amazon.com/getting-started/hands-on/query-data-s3-athena/)
- [Federated Query](https://aws.amazon.com/blogs/big-data/query-any-data-source-with-amazon-athenas-new-federated-query/)

### Ferramentas
- [JDBC Driver](https://docs.aws.amazon.com/athena/latest/ug/connect-with-jdbc.html)
- [ODBC Driver](https://docs.aws.amazon.com/athena/latest/ug/connect-with-odbc.html)
- [Python (boto3)](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/athena.html)

### Blogs e Artigos
- [AWS Big Data Blog - Athena](https://aws.amazon.com/blogs/big-data/tag/amazon-athena/)
- [Performance Tuning](https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/)
- [Cost Optimization](https://aws.amazon.com/blogs/big-data/analyzing-data-in-s3-using-amazon-athena/)

---

**Athena** é ideal para análise de dados ad-hoc no S3 sem necessidade de infraestrutura. Combine com Parquet, partitioning e compression para melhor performance e menor custo! 🚀
