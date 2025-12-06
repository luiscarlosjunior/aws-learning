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

## Curso Completo de SQL para Athena

### Nível 1: Fundamentos SQL

#### 1.1. SELECT Básico
```sql
-- Selecionar todas as colunas
SELECT * FROM users LIMIT 10;

-- Selecionar colunas específicas
SELECT user_id, name, email FROM users;

-- Usar aliases
SELECT 
    user_id AS id,
    name AS full_name,
    email AS contact_email
FROM users;

-- Concatenação de strings
SELECT 
    user_id,
    name || ' <' || email || '>' AS contact_info
FROM users;
```

#### 1.2. WHERE - Filtragem
```sql
-- Comparações simples
SELECT * FROM orders WHERE total_amount > 100;

-- Múltiplas condições
SELECT * FROM orders 
WHERE total_amount > 100 
  AND status = 'completed'
  AND order_date >= DATE '2024-01-01';

-- IN operator
SELECT * FROM users 
WHERE country IN ('Brazil', 'Argentina', 'Chile');

-- LIKE pattern matching
SELECT * FROM users 
WHERE email LIKE '%@gmail.com';

-- NOT operator
SELECT * FROM orders 
WHERE status NOT IN ('cancelled', 'refunded');

-- BETWEEN
SELECT * FROM orders 
WHERE order_date BETWEEN DATE '2024-01-01' AND DATE '2024-12-31';

-- IS NULL / IS NOT NULL
SELECT * FROM users WHERE phone_number IS NULL;
SELECT * FROM users WHERE phone_number IS NOT NULL;
```

#### 1.3. ORDER BY - Ordenação
```sql
-- Ordem crescente
SELECT * FROM products 
ORDER BY price ASC;

-- Ordem decrescente
SELECT * FROM products 
ORDER BY price DESC;

-- Múltiplas colunas
SELECT * FROM orders 
ORDER BY order_date DESC, total_amount DESC;

-- NULLS FIRST / NULLS LAST
SELECT * FROM products 
ORDER BY discount NULLS LAST;
```

#### 1.4. LIMIT - Limitação de Resultados
```sql
-- Top 10
SELECT * FROM orders 
ORDER BY total_amount DESC 
LIMIT 10;

-- Paginação (com OFFSET no Presto)
SELECT * FROM users 
ORDER BY created_at DESC 
OFFSET 100 LIMIT 50;
```

### Nível 2: Agregações e Agrupamentos

#### 2.1. Funções de Agregação
```sql
-- COUNT
SELECT COUNT(*) AS total_users FROM users;
SELECT COUNT(DISTINCT country) AS unique_countries FROM users;

-- SUM
SELECT SUM(total_amount) AS total_revenue FROM orders;

-- AVG
SELECT AVG(total_amount) AS average_order_value FROM orders;

-- MIN / MAX
SELECT 
    MIN(total_amount) AS min_order,
    MAX(total_amount) AS max_order
FROM orders;

-- Multiple aggregations
SELECT 
    COUNT(*) AS total_orders,
    SUM(total_amount) AS total_revenue,
    AVG(total_amount) AS avg_order,
    MIN(total_amount) AS min_order,
    MAX(total_amount) AS max_order
FROM orders
WHERE order_date >= DATE '2024-01-01';
```

#### 2.2. GROUP BY
```sql
-- Agrupar por uma coluna
SELECT 
    country,
    COUNT(*) AS user_count
FROM users
GROUP BY country
ORDER BY user_count DESC;

-- Agrupar por múltiplas colunas
SELECT 
    country,
    city,
    COUNT(*) AS user_count
FROM users
GROUP BY country, city
ORDER BY country, user_count DESC;

-- GROUP BY com agregações múltiplas
SELECT 
    product_category,
    COUNT(*) AS total_orders,
    SUM(quantity) AS total_quantity,
    SUM(total_amount) AS revenue,
    AVG(total_amount) AS avg_order
FROM orders
GROUP BY product_category
ORDER BY revenue DESC;
```

#### 2.3. HAVING - Filtrar Grupos
```sql
-- HAVING com agregação
SELECT 
    country,
    COUNT(*) AS user_count
FROM users
GROUP BY country
HAVING COUNT(*) > 100
ORDER BY user_count DESC;

-- WHERE vs HAVING
SELECT 
    country,
    COUNT(*) AS active_users
FROM users
WHERE status = 'active'  -- Filtro ANTES do agrupamento
GROUP BY country
HAVING COUNT(*) > 50     -- Filtro DEPOIS do agrupamento
ORDER BY active_users DESC;

-- Múltiplas condições HAVING
SELECT 
    product_category,
    SUM(total_amount) AS revenue,
    AVG(total_amount) AS avg_order
FROM orders
GROUP BY product_category
HAVING SUM(total_amount) > 10000 
   AND AVG(total_amount) > 50
ORDER BY revenue DESC;
```

### Nível 3: JOINs e Relacionamentos

#### 3.1. INNER JOIN
```sql
-- JOIN básico
SELECT 
    o.order_id,
    o.order_date,
    u.name AS customer_name,
    u.email
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id;

-- JOIN com múltiplas tabelas
SELECT 
    o.order_id,
    u.name AS customer,
    p.product_name,
    oi.quantity,
    oi.price
FROM orders o
INNER JOIN users u ON o.user_id = u.user_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id
WHERE o.order_date >= DATE '2024-01-01';

-- JOIN com agregação
SELECT 
    u.name,
    COUNT(o.order_id) AS total_orders,
    SUM(o.total_amount) AS lifetime_value
FROM users u
INNER JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.name
HAVING COUNT(o.order_id) > 5
ORDER BY lifetime_value DESC;
```

#### 3.2. LEFT JOIN
```sql
-- LEFT JOIN para incluir usuários sem pedidos
SELECT 
    u.user_id,
    u.name,
    COUNT(o.order_id) AS order_count,
    COALESCE(SUM(o.total_amount), 0) AS total_spent
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id, u.name
ORDER BY total_spent DESC;

-- Encontrar registros sem correspondência
SELECT 
    u.user_id,
    u.name,
    u.email
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
WHERE o.order_id IS NULL;  -- Usuários que nunca fizeram pedidos
```

#### 3.3. CROSS JOIN
```sql
-- Combinação cartesiana
SELECT 
    d.date,
    c.category,
    COALESCE(SUM(s.amount), 0) AS revenue
FROM date_dimension d
CROSS JOIN category_dimension c
LEFT JOIN sales s 
    ON d.date = s.sale_date 
    AND c.category = s.category
GROUP BY d.date, c.category
ORDER BY d.date, c.category;
```

### Nível 4: Subconsultas e CTEs

#### 4.1. Subconsultas no WHERE
```sql
-- Subconsulta simples
SELECT * FROM orders
WHERE user_id IN (
    SELECT user_id FROM users WHERE country = 'Brazil'
);

-- NOT IN
SELECT * FROM products
WHERE product_id NOT IN (
    SELECT DISTINCT product_id FROM orders
    WHERE order_date >= DATE '2024-01-01'
);

-- Subconsulta com agregação
SELECT * FROM orders
WHERE total_amount > (
    SELECT AVG(total_amount) FROM orders
);
```

#### 4.2. Subconsultas no FROM
```sql
-- Subconsulta como tabela derivada
SELECT 
    country,
    avg_order_value,
    total_orders
FROM (
    SELECT 
        u.country,
        AVG(o.total_amount) AS avg_order_value,
        COUNT(o.order_id) AS total_orders
    FROM users u
    INNER JOIN orders o ON u.user_id = o.user_id
    GROUP BY u.country
) AS country_stats
WHERE total_orders > 100
ORDER BY avg_order_value DESC;
```

#### 4.3. CTEs (Common Table Expressions)
```sql
-- CTE simples
WITH active_users AS (
    SELECT user_id, name, email
    FROM users
    WHERE status = 'active'
)
SELECT 
    au.name,
    COUNT(o.order_id) AS order_count
FROM active_users au
LEFT JOIN orders o ON au.user_id = o.user_id
GROUP BY au.user_id, au.name;

-- Múltiplos CTEs
WITH 
monthly_revenue AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        SUM(total_amount) AS revenue
    FROM orders
    GROUP BY DATE_TRUNC('month', order_date)
),
previous_month AS (
    SELECT 
        month,
        revenue,
        LAG(revenue) OVER (ORDER BY month) AS prev_revenue
    FROM monthly_revenue
)
SELECT 
    month,
    revenue,
    prev_revenue,
    ((revenue - prev_revenue) / prev_revenue * 100) AS growth_pct
FROM previous_month
WHERE prev_revenue IS NOT NULL
ORDER BY month;
```

### Nível 5: Window Functions

#### 5.1. ROW_NUMBER, RANK, DENSE_RANK
```sql
-- ROW_NUMBER: numeração única
SELECT 
    user_id,
    name,
    total_spent,
    ROW_NUMBER() OVER (ORDER BY total_spent DESC) AS rank
FROM (
    SELECT 
        u.user_id,
        u.name,
        SUM(o.total_amount) AS total_spent
    FROM users u
    INNER JOIN orders o ON u.user_id = o.user_id
    GROUP BY u.user_id, u.name
) AS user_spending;

-- RANK: permite empates (pula posições)
SELECT 
    product_name,
    sales,
    RANK() OVER (ORDER BY sales DESC) AS rank
FROM product_sales;

-- DENSE_RANK: permite empates (não pula posições)
SELECT 
    product_name,
    sales,
    DENSE_RANK() OVER (ORDER BY sales DESC) AS dense_rank
FROM product_sales;

-- PARTITION BY: ranking dentro de grupos
SELECT 
    category,
    product_name,
    sales,
    ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rank_in_category
FROM product_sales;

-- Top N por grupo
WITH ranked_products AS (
    SELECT 
        category,
        product_name,
        sales,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY sales DESC) AS rn
    FROM product_sales
)
SELECT category, product_name, sales
FROM ranked_products
WHERE rn <= 3;  -- Top 3 de cada categoria
```

#### 5.2. LAG e LEAD
```sql
-- LAG: valor anterior
SELECT 
    order_date,
    revenue,
    LAG(revenue) OVER (ORDER BY order_date) AS previous_day_revenue,
    revenue - LAG(revenue) OVER (ORDER BY order_date) AS day_over_day_change
FROM daily_revenue;

-- LEAD: próximo valor
SELECT 
    order_date,
    revenue,
    LEAD(revenue) OVER (ORDER BY order_date) AS next_day_revenue
FROM daily_revenue;

-- LAG com PARTITION BY
SELECT 
    user_id,
    order_date,
    total_amount,
    LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS previous_order_date,
    order_date - LAG(order_date) OVER (PARTITION BY user_id ORDER BY order_date) AS days_since_last_order
FROM orders;
```

#### 5.3. Funções de Agregação como Window Functions
```sql
-- SUM acumulado (running total)
SELECT 
    order_date,
    daily_revenue,
    SUM(daily_revenue) OVER (ORDER BY order_date) AS cumulative_revenue
FROM daily_revenue;

-- Média móvel (7 dias)
SELECT 
    order_date,
    revenue,
    AVG(revenue) OVER (
        ORDER BY order_date 
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) AS moving_avg_7day
FROM daily_revenue;

-- Percentual do total
SELECT 
    product_category,
    revenue,
    revenue / SUM(revenue) OVER () * 100 AS pct_of_total
FROM category_revenue
ORDER BY revenue DESC;

-- Percentual dentro do grupo
SELECT 
    country,
    city,
    revenue,
    revenue / SUM(revenue) OVER (PARTITION BY country) * 100 AS pct_of_country
FROM city_revenue
ORDER BY country, revenue DESC;
```

### Nível 6: Funções Avançadas

#### 6.1. Funções de Data
```sql
-- Date manipulation
SELECT 
    order_date,
    DATE_FORMAT(order_date, '%Y-%m') AS year_month,
    DATE_TRUNC('month', order_date) AS month_start,
    DATE_TRUNC('week', order_date) AS week_start,
    YEAR(order_date) AS year,
    MONTH(order_date) AS month,
    DAY(order_date) AS day,
    DAY_OF_WEEK(order_date) AS day_of_week,
    DAY_OF_YEAR(order_date) AS day_of_year,
    QUARTER(order_date) AS quarter
FROM orders;

-- Date arithmetic
SELECT 
    order_date,
    order_date + INTERVAL '7' DAY AS one_week_later,
    order_date - INTERVAL '1' MONTH AS one_month_ago,
    DATE_DIFF('day', order_date, CURRENT_DATE) AS days_ago
FROM orders;

-- Current date/time
SELECT 
    CURRENT_DATE AS today,
    CURRENT_TIMESTAMP AS now,
    DATE_ADD('day', -30, CURRENT_DATE) AS thirty_days_ago;
```

#### 6.2. Funções de String
```sql
-- String manipulation
SELECT 
    name,
    UPPER(name) AS uppercase,
    LOWER(name) AS lowercase,
    CONCAT(first_name, ' ', last_name) AS full_name,
    SUBSTR(email, 1, POSITION('@' IN email) - 1) AS username,
    SPLIT_PART(email, '@', 2) AS domain,
    LENGTH(name) AS name_length,
    TRIM(name) AS trimmed_name,
    REPLACE(phone, '-', '') AS phone_no_dash,
    REVERSE(name) AS reversed
FROM users;

-- Pattern matching
SELECT 
    email,
    REGEXP_EXTRACT(email, '([^@]+)@(.+)', 1) AS username,
    REGEXP_EXTRACT(email, '([^@]+)@(.+)', 2) AS domain,
    REGEXP_LIKE(phone, '^\d{3}-\d{3}-\d{4}$') AS valid_phone_format
FROM users;
```

#### 6.3. Funções de Conversão
```sql
-- Type casting
SELECT 
    CAST(price AS DECIMAL(10,2)) AS price_decimal,
    CAST(order_date AS VARCHAR) AS date_string,
    TRY_CAST(user_input AS INTEGER) AS parsed_int,  -- Retorna NULL se falhar
    COALESCE(discount, 0) AS discount_or_zero,
    NULLIF(quantity, 0) AS quantity_null_if_zero
FROM orders;

-- CASE expressions
SELECT 
    order_id,
    total_amount,
    CASE 
        WHEN total_amount < 50 THEN 'Small'
        WHEN total_amount < 200 THEN 'Medium'
        WHEN total_amount < 1000 THEN 'Large'
        ELSE 'Very Large'
    END AS order_size,
    CASE
        WHEN status = 'completed' THEN 1
        ELSE 0
    END AS is_completed
FROM orders;
```

#### 6.4. Funções de Array e Map
```sql
-- Arrays
SELECT 
    ARRAY[1, 2, 3, 4, 5] AS numbers,
    ARRAY['a', 'b', 'c'] AS letters,
    CARDINALITY(ARRAY[1, 2, 3]) AS array_length,
    CONTAINS(ARRAY[1, 2, 3], 2) AS contains_two,
    ARRAY_JOIN(ARRAY['a', 'b', 'c'], ',') AS joined;

-- Array operations
WITH data AS (
    SELECT 
        user_id,
        ARRAY['product1', 'product2', 'product3'] AS purchased_products
    FROM orders
)
SELECT 
    user_id,
    product,
    purchased_products
FROM data
CROSS JOIN UNNEST(purchased_products) AS t(product);

-- Maps
SELECT 
    MAP(ARRAY['key1', 'key2'], ARRAY['value1', 'value2']) AS my_map,
    ELEMENT_AT(MAP(ARRAY['a', 'b'], ARRAY[1, 2]), 'a') AS value_for_a;
```

### Nível 7: Otimização e Performance

#### 7.1. Particionamento
```sql
-- Criar tabela particionada por data
CREATE EXTERNAL TABLE sales_partitioned (
    order_id STRING,
    user_id STRING,
    product_id STRING,
    amount DECIMAL(10,2)
)
PARTITIONED BY (
    year INT,
    month INT,
    day INT
)
STORED AS PARQUET
LOCATION 's3://my-bucket/sales/';

-- Query com partition pruning
SELECT 
    order_id,
    amount
FROM sales_partitioned
WHERE year = 2024 AND month = 1 AND day = 15;  -- Escaneia apenas essa partição

-- Adicionar partitions
MSCK REPAIR TABLE sales_partitioned;
```

#### 7.2. Bucketing e Sorting
```sql
-- Criar tabela com bucketing
CREATE TABLE user_events_bucketed
WITH (
    format = 'PARQUET',
    bucketed_by = ARRAY['user_id'],
    bucket_count = 50,
    sorted_by = ARRAY['event_timestamp']
)
AS SELECT * FROM user_events;
```

#### 7.3. CTAS para Otimização
```sql
-- Converter CSV para Parquet
CREATE TABLE logs_parquet
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    external_location = 's3://bucket/logs-parquet/'
)
AS SELECT * FROM logs_csv;

-- Pré-agregar dados
CREATE TABLE daily_metrics
WITH (
    format = 'PARQUET',
    partitioned_by = ARRAY['year', 'month']
)
AS
SELECT 
    DATE_TRUNC('day', event_timestamp) AS event_date,
    YEAR(event_timestamp) AS year,
    MONTH(event_timestamp) AS month,
    COUNT(*) AS event_count,
    COUNT(DISTINCT user_id) AS unique_users,
    SUM(revenue) AS total_revenue
FROM events
GROUP BY DATE_TRUNC('day', event_timestamp), YEAR(event_timestamp), MONTH(event_timestamp);
```

## Cenários Comuns do Dia a Dia

### Cenário 1: Análise de Vendas por Período

**Problema**: Preciso de um relatório mensal de vendas com comparação ao mês anterior.

```sql
WITH monthly_sales AS (
    SELECT 
        DATE_TRUNC('month', order_date) AS month,
        COUNT(DISTINCT order_id) AS total_orders,
        COUNT(DISTINCT user_id) AS unique_customers,
        SUM(total_amount) AS revenue,
        AVG(total_amount) AS avg_order_value
    FROM orders
    WHERE order_date >= DATE '2024-01-01'
    GROUP BY DATE_TRUNC('month', order_date)
),
sales_with_comparison AS (
    SELECT 
        month,
        total_orders,
        unique_customers,
        revenue,
        avg_order_value,
        LAG(revenue) OVER (ORDER BY month) AS prev_month_revenue,
        LAG(total_orders) OVER (ORDER BY month) AS prev_month_orders
    FROM monthly_sales
)
SELECT 
    DATE_FORMAT(month, '%Y-%m') AS month,
    total_orders,
    unique_customers,
    ROUND(revenue, 2) AS revenue,
    ROUND(avg_order_value, 2) AS avg_order_value,
    ROUND(prev_month_revenue, 2) AS prev_month_revenue,
    ROUND(((revenue - prev_month_revenue) / prev_month_revenue * 100), 2) AS revenue_growth_pct,
    ROUND(((total_orders - prev_month_orders) / CAST(prev_month_orders AS DOUBLE) * 100), 2) AS orders_growth_pct
FROM sales_with_comparison
ORDER BY month DESC;
```

### Cenário 2: Identificar Clientes Inativos

**Problema**: Encontrar clientes que não compram há mais de 90 dias para campanha de reativação.

```sql
WITH last_purchase AS (
    SELECT 
        user_id,
        MAX(order_date) AS last_order_date,
        COUNT(order_id) AS total_orders,
        SUM(total_amount) AS lifetime_value
    FROM orders
    GROUP BY user_id
)
SELECT 
    u.user_id,
    u.name,
    u.email,
    lp.last_order_date,
    DATE_DIFF('day', lp.last_order_date, CURRENT_DATE) AS days_since_last_purchase,
    lp.total_orders,
    ROUND(lp.lifetime_value, 2) AS lifetime_value,
    ROUND(lp.lifetime_value / lp.total_orders, 2) AS avg_order_value
FROM users u
INNER JOIN last_purchase lp ON u.user_id = lp.user_id
WHERE DATE_DIFF('day', lp.last_order_date, CURRENT_DATE) > 90
    AND u.status = 'active'
ORDER BY lifetime_value DESC, days_since_last_purchase DESC;
```

### Cenário 3: Análise de Cohort

**Problema**: Análise de retenção de clientes por mês de primeira compra.

```sql
WITH first_purchase AS (
    SELECT 
        user_id,
        DATE_TRUNC('month', MIN(order_date)) AS cohort_month
    FROM orders
    GROUP BY user_id
),
user_activities AS (
    SELECT 
        fp.user_id,
        fp.cohort_month,
        DATE_TRUNC('month', o.order_date) AS activity_month,
        DATE_DIFF('month', fp.cohort_month, DATE_TRUNC('month', o.order_date)) AS months_since_first
    FROM first_purchase fp
    INNER JOIN orders o ON fp.user_id = o.user_id
)
SELECT 
    DATE_FORMAT(cohort_month, '%Y-%m') AS cohort,
    months_since_first,
    COUNT(DISTINCT user_id) AS active_users,
    COUNT(DISTINCT user_id) * 100.0 / 
        FIRST_VALUE(COUNT(DISTINCT user_id)) OVER (
            PARTITION BY cohort_month 
            ORDER BY months_since_first
        ) AS retention_pct
FROM user_activities
WHERE cohort_month >= DATE '2024-01-01'
GROUP BY cohort_month, months_since_first
ORDER BY cohort_month, months_since_first;
```

### Cenário 4: Detecção de Anomalias em Logs

**Problema**: Identificar picos anormais de erros nos logs da aplicação.

```sql
WITH hourly_errors AS (
    SELECT 
        DATE_TRUNC('hour', timestamp) AS hour,
        COUNT(*) AS error_count
    FROM application_logs
    WHERE log_level = 'ERROR'
        AND timestamp >= CURRENT_TIMESTAMP - INTERVAL '7' DAY
    GROUP BY DATE_TRUNC('hour', timestamp)
),
error_stats AS (
    SELECT 
        hour,
        error_count,
        AVG(error_count) OVER (
            ORDER BY hour 
            ROWS BETWEEN 23 PRECEDING AND CURRENT ROW
        ) AS moving_avg_24h,
        STDDEV(error_count) OVER (
            ORDER BY hour 
            ROWS BETWEEN 23 PRECEDING AND CURRENT ROW
        ) AS moving_stddev_24h
    FROM hourly_errors
)
SELECT 
    hour,
    error_count,
    ROUND(moving_avg_24h, 2) AS avg_errors,
    ROUND(moving_stddev_24h, 2) AS stddev_errors,
    CASE 
        WHEN error_count > moving_avg_24h + (2 * moving_stddev_24h) THEN 'ANOMALY'
        ELSE 'NORMAL'
    END AS status
FROM error_stats
WHERE moving_stddev_24h IS NOT NULL
ORDER BY hour DESC;
```

### Cenário 5: Análise de Funil de Conversão

**Problema**: Calcular taxas de conversão em cada etapa do funil.

```sql
WITH funnel_events AS (
    SELECT 
        user_id,
        MAX(CASE WHEN event_type = 'page_view' THEN 1 ELSE 0 END) AS viewed,
        MAX(CASE WHEN event_type = 'add_to_cart' THEN 1 ELSE 0 END) AS added_to_cart,
        MAX(CASE WHEN event_type = 'checkout_started' THEN 1 ELSE 0 END) AS started_checkout,
        MAX(CASE WHEN event_type = 'purchase' THEN 1 ELSE 0 END) AS purchased
    FROM user_events
    WHERE event_date >= DATE '2024-01-01'
    GROUP BY user_id
)
SELECT 
    SUM(viewed) AS step1_views,
    SUM(added_to_cart) AS step2_add_to_cart,
    SUM(started_checkout) AS step3_checkout,
    SUM(purchased) AS step4_purchase,
    ROUND(SUM(added_to_cart) * 100.0 / NULLIF(SUM(viewed), 0), 2) AS view_to_cart_pct,
    ROUND(SUM(started_checkout) * 100.0 / NULLIF(SUM(added_to_cart), 0), 2) AS cart_to_checkout_pct,
    ROUND(SUM(purchased) * 100.0 / NULLIF(SUM(started_checkout), 0), 2) AS checkout_to_purchase_pct,
    ROUND(SUM(purchased) * 100.0 / NULLIF(SUM(viewed), 0), 2) AS overall_conversion_pct
FROM funnel_events;
```

### Cenário 6: Análise RFM (Recency, Frequency, Monetary)

**Problema**: Segmentar clientes para marketing direcionado.

```sql
WITH customer_rfm AS (
    SELECT 
        user_id,
        DATE_DIFF('day', MAX(order_date), CURRENT_DATE) AS recency,
        COUNT(order_id) AS frequency,
        SUM(total_amount) AS monetary
    FROM orders
    WHERE order_date >= CURRENT_DATE - INTERVAL '365' DAY
    GROUP BY user_id
),
rfm_scores AS (
    SELECT 
        user_id,
        recency,
        frequency,
        monetary,
        NTILE(5) OVER (ORDER BY recency DESC) AS r_score,
        NTILE(5) OVER (ORDER BY frequency ASC) AS f_score,
        NTILE(5) OVER (ORDER BY monetary ASC) AS m_score
    FROM customer_rfm
)
SELECT 
    user_id,
    recency,
    frequency,
    ROUND(monetary, 2) AS monetary,
    r_score,
    f_score,
    m_score,
    CONCAT(CAST(r_score AS VARCHAR), CAST(f_score AS VARCHAR), CAST(m_score AS VARCHAR)) AS rfm_segment,
    CASE 
        WHEN r_score >= 4 AND f_score >= 4 AND m_score >= 4 THEN 'Champions'
        WHEN r_score >= 3 AND f_score >= 3 AND m_score >= 3 THEN 'Loyal Customers'
        WHEN r_score >= 4 AND f_score <= 2 THEN 'New Customers'
        WHEN r_score <= 2 AND f_score >= 3 THEN 'At Risk'
        WHEN r_score <= 2 AND f_score <= 2 THEN 'Lost Customers'
        ELSE 'Potential'
    END AS customer_segment
FROM rfm_scores
ORDER BY monetary DESC;
```

### Cenário 7: Performance de Produtos

**Problema**: Identificar produtos mais vendidos e menos vendidos por categoria.

```sql
WITH product_performance AS (
    SELECT 
        p.category,
        p.product_id,
        p.product_name,
        COUNT(DISTINCT o.order_id) AS order_count,
        SUM(oi.quantity) AS units_sold,
        SUM(oi.quantity * oi.price) AS revenue,
        AVG(oi.price) AS avg_price
    FROM products p
    LEFT JOIN order_items oi ON p.product_id = oi.product_id
    LEFT JOIN orders o ON oi.order_id = o.order_id 
        AND o.order_date >= CURRENT_DATE - INTERVAL '90' DAY
    GROUP BY p.category, p.product_id, p.product_name
),
ranked_products AS (
    SELECT 
        category,
        product_id,
        product_name,
        order_count,
        units_sold,
        ROUND(revenue, 2) AS revenue,
        ROUND(avg_price, 2) AS avg_price,
        ROW_NUMBER() OVER (PARTITION BY category ORDER BY revenue DESC) AS rank_in_category,
        ROUND(revenue * 100.0 / SUM(revenue) OVER (PARTITION BY category), 2) AS pct_of_category_revenue
    FROM product_performance
    WHERE revenue IS NOT NULL
)
SELECT *
FROM ranked_products
WHERE rank_in_category <= 5  -- Top 5 por categoria
ORDER BY category, rank_in_category;
```

## Problemas Reais do StackOverflow e Soluções

### Problema 1: "Query muito lenta, escaneando muitos dados"

**Contexto**: Query simples está custando muito e demorando.

```sql
-- ❌ EVITE: Sem filtros de partição
SELECT COUNT(*) 
FROM logs 
WHERE status_code = 500;

-- ✅ SOLUÇÃO: Use partition pruning
SELECT COUNT(*) 
FROM logs 
WHERE year = 2024 
  AND month = 1 
  AND day = 15 
  AND status_code = 500;

-- ✅ MELHOR: Converta para Parquet e particione
CREATE TABLE logs_optimized
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY',
    partitioned_by = ARRAY['year', 'month', 'day']
)
AS
SELECT 
    *,
    YEAR(timestamp) AS year,
    MONTH(timestamp) AS month,
    DAY(timestamp) AS day
FROM logs_csv
WHERE timestamp >= DATE '2024-01-01';
```

**Resultado**: Redução de 90-95% no custo e tempo de execução.

### Problema 2: "HIVE_CURSOR_ERROR: Row is not a valid JSON Object"

**Contexto**: Erro ao processar JSON malformado em logs.

```sql
-- ❌ Problema: JSON inválido causa falha
SELECT 
    json_extract(log_data, '$.user_id') AS user_id
FROM logs;

-- ✅ SOLUÇÃO: Use TRY para tratar erros
SELECT 
    TRY(json_extract(log_data, '$.user_id')) AS user_id,
    CASE 
        WHEN json_extract(log_data, '$.user_id') IS NULL THEN 'invalid_json'
        ELSE 'valid'
    END AS json_status
FROM logs;

-- ✅ MELHOR: Filtrar JSONs inválidos primeiro
WITH valid_logs AS (
    SELECT *
    FROM logs
    WHERE TRY(json_parse(log_data)) IS NOT NULL
)
SELECT 
    json_extract_scalar(log_data, '$.user_id') AS user_id,
    json_extract_scalar(log_data, '$.action') AS action,
    json_extract_scalar(log_data, '$.timestamp') AS timestamp
FROM valid_logs;
```

### Problema 3: "Como fazer PIVOT/UNPIVOT no Athena?"

**Contexto**: Transformar linhas em colunas e vice-versa.

```sql
-- PIVOT: Linhas para colunas
SELECT 
    user_id,
    SUM(CASE WHEN product_category = 'Electronics' THEN total_amount ELSE 0 END) AS electronics,
    SUM(CASE WHEN product_category = 'Clothing' THEN total_amount ELSE 0 END) AS clothing,
    SUM(CASE WHEN product_category = 'Books' THEN total_amount ELSE 0 END) AS books,
    SUM(CASE WHEN product_category = 'Home' THEN total_amount ELSE 0 END) AS home
FROM orders
GROUP BY user_id;

-- UNPIVOT: Colunas para linhas
WITH sales_data AS (
    SELECT 
        'Q1' AS quarter,
        100 AS electronics,
        200 AS clothing,
        150 AS books
)
SELECT quarter, 'electronics' AS category, electronics AS amount FROM sales_data
UNION ALL
SELECT quarter, 'clothing' AS category, clothing AS amount FROM sales_data
UNION ALL
SELECT quarter, 'books' AS category, books AS amount FROM sales_data;

-- UNPIVOT com UNNEST (mais elegante)
WITH sales_data AS (
    SELECT 
        'Q1' AS quarter,
        MAP(ARRAY['electronics', 'clothing', 'books'], 
            ARRAY[100, 200, 150]) AS sales
)
SELECT 
    quarter,
    category,
    amount
FROM sales_data
CROSS JOIN UNNEST(sales) AS t(category, amount);
```

### Problema 4: "Como lidar com dados nested/JSON complexos?"

**Contexto**: Logs com estrutura JSON aninhada.

```sql
-- Estrutura JSON exemplo:
-- {"user": {"id": 123, "name": "John"}, "events": [{"type": "click", "ts": "2024-01-01"}]}

-- Extrair campos simples
SELECT 
    json_extract_scalar(data, '$.user.id') AS user_id,
    json_extract_scalar(data, '$.user.name') AS user_name
FROM logs;

-- Extrair arrays
SELECT 
    json_extract_scalar(data, '$.user.id') AS user_id,
    json_extract(data, '$.events') AS events_array
FROM logs;

-- Flatten arrays (UNNEST)
WITH parsed_logs AS (
    SELECT 
        json_extract_scalar(data, '$.user.id') AS user_id,
        CAST(json_extract(data, '$.events') AS ARRAY(JSON)) AS events
    FROM logs
)
SELECT 
    user_id,
    json_extract_scalar(event, '$.type') AS event_type,
    json_extract_scalar(event, '$.ts') AS event_timestamp
FROM parsed_logs
CROSS JOIN UNNEST(events) AS t(event);
```

### Problema 5: "EXCEEDED_LOCAL_MEMORY_LIMIT"

**Contexto**: Query muito grande falha por falta de memória.

```sql
-- ❌ Problema: JOIN sem filtros adequados
SELECT *
FROM large_table_1 a
JOIN large_table_2 b ON a.id = b.id;

-- ✅ SOLUÇÃO 1: Filtrar antes do JOIN
SELECT *
FROM (
    SELECT * FROM large_table_1 
    WHERE date >= DATE '2024-01-01'
) a
JOIN (
    SELECT * FROM large_table_2 
    WHERE date >= DATE '2024-01-01'
) b ON a.id = b.id;

-- ✅ SOLUÇÃO 2: Usar partições
SELECT *
FROM large_table_1 a
JOIN large_table_2 b 
    ON a.id = b.id 
    AND a.year = b.year 
    AND a.month = b.month
WHERE a.year = 2024 AND a.month = 1;

-- ✅ SOLUÇÃO 3: Quebrar em múltiplas queries menores
-- Criar tabela intermediária
CREATE TABLE temp_filtered AS
SELECT id, date, amount
FROM large_table_1
WHERE date >= DATE '2024-01-01';

-- Depois fazer JOIN
SELECT *
FROM temp_filtered a
JOIN large_table_2 b ON a.id = b.id;
```

### Problema 6: "Como fazer um DELETE/UPDATE no Athena?"

**Contexto**: Athena não suporta DELETE/UPDATE diretamente em tabelas no S3.

```sql
-- ❌ NÃO FUNCIONA
DELETE FROM users WHERE status = 'inactive';
UPDATE users SET status = 'active' WHERE user_id = 123;

-- ✅ SOLUÇÃO: Recriar tabela sem os registros
CREATE TABLE users_new
WITH (
    format = 'PARQUET',
    external_location = 's3://bucket/users-new/'
)
AS
SELECT *
FROM users
WHERE status != 'inactive';  -- Excluir registros

-- Depois renomear/substituir
DROP TABLE users;
ALTER TABLE users_new RENAME TO users;

-- ✅ PARA UPDATE: Recriar com dados modificados
CREATE TABLE users_updated
WITH (
    format = 'PARQUET',
    external_location = 's3://bucket/users-updated/'
)
AS
SELECT 
    user_id,
    name,
    email,
    CASE 
        WHEN user_id = 123 THEN 'active'
        ELSE status
    END AS status
FROM users;

-- ✅ ALTERNATIVA: Use Apache Iceberg/Hudi para suporte a ACID
-- Permite DELETE/UPDATE nativos
```

### Problema 7: "Query retorna resultados duplicados"

**Contexto**: JOINs causam duplicação inesperada de dados.

```sql
-- ❌ Problema: JOIN gera duplicatas
SELECT 
    u.user_id,
    u.name,
    o.order_id,
    COUNT(*) AS item_count  -- Errado: conta items repetidos
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_items oi ON o.order_id = oi.order_id
GROUP BY u.user_id, u.name, o.order_id;

-- ✅ SOLUÇÃO 1: Usar DISTINCT
SELECT DISTINCT
    u.user_id,
    u.name,
    o.order_id
FROM users u
JOIN orders o ON u.user_id = o.user_id;

-- ✅ SOLUÇÃO 2: Agregar em subquery
WITH order_summary AS (
    SELECT 
        order_id,
        COUNT(*) AS item_count
    FROM order_items
    GROUP BY order_id
)
SELECT 
    u.user_id,
    u.name,
    o.order_id,
    os.item_count
FROM users u
JOIN orders o ON u.user_id = o.user_id
JOIN order_summary os ON o.order_id = os.order_id;

-- ✅ SOLUÇÃO 3: Window function para deduplicação
WITH numbered_rows AS (
    SELECT 
        *,
        ROW_NUMBER() OVER (PARTITION BY user_id, order_date ORDER BY order_id) AS rn
    FROM orders
)
SELECT *
FROM numbered_rows
WHERE rn = 1;
```

### Problema 8: "Como fazer paginação eficiente?"

**Contexto**: Implementar paginação para grandes resultados.

```sql
-- ❌ Problema: OFFSET é lento para páginas grandes
SELECT * FROM users
ORDER BY user_id
LIMIT 100 OFFSET 10000;  -- Lento: escaneia 10,100 linhas

-- ✅ SOLUÇÃO: Use keyset pagination (cursor-based)
-- Primeira página
SELECT * FROM users
WHERE user_id > 0
ORDER BY user_id
LIMIT 100;

-- Página seguinte (último user_id da página anterior = 100)
SELECT * FROM users
WHERE user_id > 100
ORDER BY user_id
LIMIT 100;

-- ✅ Para ordenação complexa
-- Primeira página
SELECT * FROM orders
WHERE (order_date, order_id) > (TIMESTAMP '1970-01-01', 0)
ORDER BY order_date DESC, order_id DESC
LIMIT 100;

-- Próxima página (última linha anterior: 2024-01-15, 12345)
SELECT * FROM orders
WHERE (order_date, order_id) < (TIMESTAMP '2024-01-15', 12345)
ORDER BY order_date DESC, order_id DESC
LIMIT 100;
```

### Problema 9: "Como calcular percentuais acumulados?"

**Contexto**: Mostrar distribuição acumulada de valores.

```sql
-- Distribuição de vendas por produto (Pareto)
WITH product_sales AS (
    SELECT 
        product_id,
        product_name,
        SUM(quantity * price) AS revenue
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    GROUP BY product_id, product_name
),
ranked_products AS (
    SELECT 
        product_id,
        product_name,
        revenue,
        SUM(revenue) OVER () AS total_revenue,
        SUM(revenue) OVER (ORDER BY revenue DESC) AS cumulative_revenue
    FROM product_sales
)
SELECT 
    product_id,
    product_name,
    ROUND(revenue, 2) AS revenue,
    ROUND(revenue * 100.0 / total_revenue, 2) AS pct_of_total,
    ROUND(cumulative_revenue * 100.0 / total_revenue, 2) AS cumulative_pct,
    CASE 
        WHEN cumulative_revenue * 100.0 / total_revenue <= 80 THEN 'Top 80%'
        ELSE 'Bottom 20%'
    END AS pareto_category
FROM ranked_products
ORDER BY revenue DESC;
```

### Problema 10: "Como encontrar gaps em séries temporais?"

**Contexto**: Identificar períodos sem dados.

```sql
-- ⚠️ NOTA: Athena/Presto NÃO suporta RECURSIVE CTEs
-- Use SEQUENCE para gerar série de datas

-- Gerar série de datas completa e identificar gaps
WITH date_array AS (
    SELECT SEQUENCE(DATE '2024-01-01', DATE '2024-12-31', INTERVAL '1' DAY) AS dates
),
date_series AS (
    SELECT date_value AS date
    FROM date_array
    CROSS JOIN UNNEST(dates) AS t(date_value)
),
actual_data AS (
    SELECT 
        DATE(order_date) AS date,
        COUNT(*) AS order_count
    FROM orders
    WHERE order_date >= DATE '2024-01-01'
    GROUP BY DATE(order_date)
)
SELECT 
    ds.date,
    COALESCE(ad.order_count, 0) AS order_count,
    CASE 
        WHEN ad.order_count IS NULL THEN 'MISSING DATA'
        ELSE 'OK'
    END AS status
FROM date_series ds
LEFT JOIN actual_data ad ON ds.date = ad.date
WHERE ad.order_count IS NULL  -- Apenas dias sem dados
ORDER BY ds.date;
```

## Técnicas Avançadas de Otimização

### 1. Estratégia de Bucketing para JOINs Frequentes

```sql
-- Criar tabelas bucketed pelo campo de JOIN
CREATE TABLE orders_bucketed
WITH (
    format = 'PARQUET',
    bucketed_by = ARRAY['user_id'],
    bucket_count = 50
)
AS SELECT * FROM orders;

CREATE TABLE users_bucketed
WITH (
    format = 'PARQUET',
    bucketed_by = ARRAY['user_id'],
    bucket_count = 50
)
AS SELECT * FROM users;

-- JOINs serão mais eficientes
SELECT *
FROM orders_bucketed o
JOIN users_bucketed u ON o.user_id = u.user_id;
```

### 2. Columnar Storage para Queries Analíticas

```sql
-- Comparação de formatos
-- CSV: 10 GB, $0.50 por query
-- Parquet: 2 GB (comprimido), $0.10 por query

CREATE TABLE events_parquet
WITH (
    format = 'PARQUET',
    parquet_compression = 'SNAPPY'
)
AS
SELECT 
    -- Otimizar tipos de dados
    CAST(user_id AS BIGINT) AS user_id,
    CAST(timestamp AS TIMESTAMP) AS timestamp,
    event_type,
    CAST(revenue AS DECIMAL(10,2)) AS revenue,
    -- Comprimir strings longas em categóricos quando possível
    CASE 
        WHEN LENGTH(url) > 100 THEN SUBSTR(url, 1, 100)
        ELSE url
    END AS url
FROM events_csv;
```

### 3. Incremental Processing

```sql
-- ⚠️ NOTA: Athena não suporta INSERT INTO para tabelas externas
-- Use CTAS (Create Table As Select) para processar dados incrementais

-- Primeira execução: processar tudo
CREATE TABLE processed_events
WITH (
    format = 'PARQUET',
    external_location = 's3://bucket/processed-events/'
)
AS
SELECT 
    event_id,
    user_id,
    event_type,
    timestamp,
    CURRENT_TIMESTAMP AS processed_at
FROM raw_events;

-- Execuções subsequentes: recrie com dados novos + antigos
CREATE TABLE processed_events_new
WITH (
    format = 'PARQUET',
    external_location = 's3://bucket/processed-events-new/'
)
AS
SELECT * FROM processed_events
UNION ALL
SELECT 
    re.event_id,
    re.user_id,
    re.event_type,
    re.timestamp,
    CURRENT_TIMESTAMP AS processed_at
FROM raw_events re
LEFT JOIN processed_events pe ON re.event_id = pe.event_id
WHERE pe.event_id IS NULL;

-- Substituir tabela antiga
DROP TABLE processed_events;
ALTER TABLE processed_events_new RENAME TO processed_events;
```

### 4. Query Result Reuse

```sql
-- Athena cacheia resultados por 24h
-- Para aproveitar:

-- 1. Use queries idênticas (mesma string SQL)
SELECT COUNT(*) FROM orders WHERE status = 'completed';

-- 2. Configure workgroup para enable query result reuse
-- aws athena update-work-group \
--   --work-group primary \
--   --configuration-updates \
--     ResultConfigurationUpdates={OutputLocation=s3://bucket/},\
--     EnforceWorkGroupConfiguration=true,\
--     QueryResultsReuseConfiguration={Enable=true}

-- 3. Verifique se query usou cache
-- aws athena get-query-execution --query-execution-id <id>
-- "ResultReuseInformation": {"ReusedPreviousResult": true}
```

## Recursos Adicionais

### Blogs e Artigos Recomendados
- [Top 10 Performance Tuning Tips for Amazon Athena](https://aws.amazon.com/blogs/big-data/top-10-performance-tuning-tips-for-amazon-athena/)
- [Athena Query Optimization Techniques](https://aws.amazon.com/blogs/big-data/analyzing-data-in-s3-using-amazon-athena/)
- [Cost Optimization Best Practices](https://aws.amazon.com/blogs/big-data/save-up-to-90-on-amazon-athena-query-costs-by-using-columnar-formats/)

### Ferramentas da Comunidade
- [dbt-athena](https://github.com/dbt-athena/dbt-athena): dbt adapter para Athena
- [PyAthena](https://github.com/laughingman7743/PyAthena): Python DB API para Athena
- [Athena JDBC Driver](https://docs.aws.amazon.com/athena/latest/ug/connect-with-jdbc.html): Conectar via JDBC

### Prática e Exercícios
1. **Dataset Público**: Use [AWS Public Datasets](https://registry.opendata.aws/) para praticar
2. **Sample Queries**: [Athena Query Examples](https://github.com/aws-samples/aws-athena-query-federation)
3. **Workshops**: [Athena Workshop](https://athena-in-action.workshop.aws/)

---

**Athena** é ideal para análise de dados ad-hoc no S3 sem necessidade de infraestrutura. Combine com Parquet, partitioning e compression para melhor performance e menor custo! 🚀
