# Amazon Timestream

## O que é Timestream?

Amazon Timestream é um banco de dados de séries temporais serverless, rápido e escalável, otimizado para armazenar e analisar trilhões de eventos por dia. Ideal para IoT, aplicações operacionais e analytics de séries temporais.

## Características

- **Serverless**: Sem provisionamento de capacidade
- **Auto-scaling**: Escalabilidade automática
- **Fast queries**: 1000x mais rápido que bancos relacionais
- **Cost-effective**: 1/10 do custo de bancos relacionais
- **Time-series optimized**: Built-in time-series analytics
- **Data tiering**: Automático (memory → SSD → S3)

## Conceitos Principais

### Database e Tables

```
Database
  ├─ Table 1 (IoT sensors)
  │   ├─ Memory store (recent data, horas/dias)
  │   └─ Magnetic store (historical data, meses/anos)
  │
  └─ Table 2 (Application metrics)
      ├─ Memory store
      └─ Magnetic store
```

### Dados de Série Temporal

```python
{
    'Dimensions': [
        {'Name': 'device_id', 'Value': 'sensor_001'},
        {'Name': 'location', 'Value': 'warehouse_1'},
        {'Name': 'type', 'Value': 'temperature'}
    ],
    'MeasureName': 'temperature',
    'MeasureValue': '25.5',
    'MeasureValueType': 'DOUBLE',
    'Time': '1672531200000',  # Unix timestamp ms
    'TimeUnit': 'MILLISECONDS'
}
```

### Data Tiering

**Memory Store:**
- High-speed queries
- Recent data (configurável: 1 hora - 1 ano)
- Mais caro

**Magnetic Store:**
- Long-term storage
- Historical data (até 200 anos)
- Mais barato
- Queries um pouco mais lentas

## Exemplos Práticos

### 1. IoT Sensor Data

```python
import boto3
from datetime import datetime, timedelta
import time

timestream_write = boto3.client('timestream-write')
timestream_query = boto3.client('timestream-query')

DATABASE_NAME = 'IoTDatabase'
TABLE_NAME = 'SensorData'

def create_database_and_table():
    """Criar database e table"""
    try:
        timestream_write.create_database(DatabaseName=DATABASE_NAME)
        print(f"Database {DATABASE_NAME} created")
    except timestream_write.exceptions.ConflictException:
        print(f"Database {DATABASE_NAME} already exists")
    
    try:
        timestream_write.create_table(
            DatabaseName=DATABASE_NAME,
            TableName=TABLE_NAME,
            RetentionProperties={
                'MemoryStoreRetentionPeriodInHours': 24,  # 1 dia em memory
                'MagneticStoreRetentionPeriodInDays': 365  # 1 ano em magnetic
            }
        )
        print(f"Table {TABLE_NAME} created")
    except timestream_write.exceptions.ConflictException:
        print(f"Table {TABLE_NAME} already exists")

def write_sensor_data(device_id, location, temperature, humidity):
    """Escrever dados do sensor"""
    current_time = str(int(time.time() * 1000))
    
    records = [
        {
            'Dimensions': [
                {'Name': 'device_id', 'Value': device_id},
                {'Name': 'location', 'Value': location}
            ],
            'MeasureName': 'metrics',
            'MeasureValueType': 'MULTI',
            'MeasureValues': [
                {'Name': 'temperature', 'Value': str(temperature), 'Type': 'DOUBLE'},
                {'Name': 'humidity', 'Value': str(humidity), 'Type': 'DOUBLE'}
            ],
            'Time': current_time,
            'TimeUnit': 'MILLISECONDS'
        }
    ]
    
    try:
        result = timestream_write.write_records(
            DatabaseName=DATABASE_NAME,
            TableName=TABLE_NAME,
            Records=records
        )
        print(f"Written records: {result['RecordsIngested']}")
    except Exception as e:
        print(f"Error writing records: {e}")

def query_sensor_data(device_id, hours=24):
    """Query dados do sensor"""
    query = f"""
        SELECT 
            device_id,
            location,
            time,
            measure_value::double as temperature,
            measure_value::double as humidity
        FROM "{DATABASE_NAME}"."{TABLE_NAME}"
        WHERE device_id = '{device_id}'
            AND time > ago({hours}h)
            AND measure_name IN ('temperature', 'humidity')
        ORDER BY time DESC
    """
    
    try:
        response = timestream_query.query(QueryString=query)
        
        rows = response['Rows']
        for row in rows:
            values = [col['ScalarValue'] for col in row['Data']]
            print(f"Device: {values[0]}, Location: {values[1]}, Time: {values[2]}")
            print(f"Temperature: {values[3]}°C, Humidity: {values[4]}%")
    except Exception as e:
        print(f"Error querying: {e}")

# Uso
create_database_and_table()

# Simular dados de múltiplos sensores
for i in range(10):
    write_sensor_data(
        device_id=f'sensor_{i:03d}',
        location='warehouse_1',
        temperature=20 + i * 0.5,
        humidity=60 + i * 2
    )
    time.sleep(1)

# Query dados
query_sensor_data('sensor_001', hours=1)
```

### 2. Application Metrics Monitoring

```python
import boto3
import time
import random

timestream_write = boto3.client('timestream-write')

DATABASE_NAME = 'ApplicationMetrics'
TABLE_NAME = 'APIMetrics'

def write_api_metrics(service_name, endpoint, status_code, latency_ms):
    """Escrever métricas de API"""
    current_time = str(int(time.time() * 1000))
    
    records = [{
        'Dimensions': [
            {'Name': 'service', 'Value': service_name},
            {'Name': 'endpoint', 'Value': endpoint},
            {'Name': 'status_code', 'Value': str(status_code)}
        ],
        'MeasureName': 'latency_ms',
        'MeasureValue': str(latency_ms),
        'MeasureValueType': 'DOUBLE',
        'Time': current_time,
        'TimeUnit': 'MILLISECONDS'
    }]
    
    timestream_write.write_records(
        DatabaseName=DATABASE_NAME,
        TableName=TABLE_NAME,
        Records=records
    )

def query_p99_latency(service_name, minutes=60):
    """Query P99 latency"""
    query = f"""
        SELECT 
            service,
            endpoint,
            approx_percentile(latency_ms, 0.99) as p99_latency,
            approx_percentile(latency_ms, 0.95) as p95_latency,
            avg(latency_ms) as avg_latency,
            count(*) as request_count
        FROM "{DATABASE_NAME}"."{TABLE_NAME}"
        WHERE service = '{service_name}'
            AND time > ago({minutes}m)
        GROUP BY service, endpoint
        ORDER BY p99_latency DESC
    """
    
    timestream_query = boto3.client('timestream-query')
    response = timestream_query.query(QueryString=query)
    
    return response['Rows']

# Simular métricas de API
endpoints = ['/api/users', '/api/orders', '/api/products']
for _ in range(100):
    write_api_metrics(
        service_name='web-api',
        endpoint=random.choice(endpoints),
        status_code=random.choice([200, 200, 200, 201, 400, 500]),
        latency_ms=random.uniform(10, 1000)
    )
    time.sleep(0.1)

# Analisar performance
results = query_p99_latency('web-api', minutes=10)
for row in results:
    print(row)
```

### 3. Financial Market Data

```python
import boto3
from datetime import datetime, timedelta

timestream_write = boto3.client('timestream-write')
timestream_query = boto3.client('timestream-query')

DATABASE_NAME = 'FinancialData'
TABLE_NAME = 'StockPrices'

def write_stock_price(symbol, price, volume):
    """Escrever preço de ação"""
    current_time = str(int(time.time() * 1000))
    
    records = [{
        'Dimensions': [
            {'Name': 'symbol', 'Value': symbol}
        ],
        'MeasureName': 'metrics',
        'MeasureValueType': 'MULTI',
        'MeasureValues': [
            {'Name': 'price', 'Value': str(price), 'Type': 'DOUBLE'},
            {'Name': 'volume', 'Value': str(volume), 'Type': 'BIGINT'}
        ],
        'Time': current_time,
        'TimeUnit': 'MILLISECONDS'
    }]
    
    timestream_write.write_records(
        DatabaseName=DATABASE_NAME,
        TableName=TABLE_NAME,
        Records=records
    )

def query_moving_average(symbol, window_minutes=15):
    """Query moving average"""
    query = f"""
        WITH binned_data AS (
            SELECT 
                symbol,
                bin(time, {window_minutes}m) as time_bucket,
                avg(measure_value::double) as avg_price
            FROM "{DATABASE_NAME}"."{TABLE_NAME}"
            WHERE symbol = '{symbol}'
                AND measure_name = 'price'
                AND time > ago(24h)
            GROUP BY symbol, bin(time, {window_minutes}m)
        )
        SELECT 
            symbol,
            time_bucket,
            avg_price,
            LAG(avg_price, 1) OVER (ORDER BY time_bucket) as prev_price,
            avg_price - LAG(avg_price, 1) OVER (ORDER BY time_bucket) as price_change
        FROM binned_data
        ORDER BY time_bucket DESC
    """
    
    response = timestream_query.query(QueryString=query)
    return response['Rows']

# Uso
symbols = ['AAPL', 'GOOGL', 'MSFT']
for symbol in symbols:
    write_stock_price(symbol, 150.0 + hash(symbol) % 100, 1000000)

# Analisar tendências
results = query_moving_average('AAPL', window_minutes=15)
for row in results:
    print(row)
```

### 4. DevOps Monitoring

```python
import boto3
import psutil
import time

timestream_write = boto3.client('timestream-write')

DATABASE_NAME = 'DevOpsMetrics'
TABLE_NAME = 'SystemMetrics'

def collect_system_metrics(hostname):
    """Coletar métricas do sistema"""
    current_time = str(int(time.time() * 1000))
    
    # Coletar métricas
    cpu_percent = psutil.cpu_percent(interval=1)
    memory = psutil.virtual_memory()
    disk = psutil.disk_usage('/')
    
    records = [{
        'Dimensions': [
            {'Name': 'hostname', 'Value': hostname},
            {'Name': 'region', 'Value': 'us-east-1'}
        ],
        'MeasureName': 'system_metrics',
        'MeasureValueType': 'MULTI',
        'MeasureValues': [
            {'Name': 'cpu_percent', 'Value': str(cpu_percent), 'Type': 'DOUBLE'},
            {'Name': 'memory_percent', 'Value': str(memory.percent), 'Type': 'DOUBLE'},
            {'Name': 'disk_percent', 'Value': str(disk.percent), 'Type': 'DOUBLE'}
        ],
        'Time': current_time,
        'TimeUnit': 'MILLISECONDS'
    }]
    
    timestream_write.write_records(
        DatabaseName=DATABASE_NAME,
        TableName=TABLE_NAME,
        Records=records
    )
    print(f"Metrics collected: CPU={cpu_percent}%, Memory={memory.percent}%")

def query_anomalies(threshold=90):
    """Detectar anomalias (uso > threshold)"""
    query = f"""
        SELECT 
            hostname,
            time,
            measure_value::double as cpu_percent
        FROM "{DATABASE_NAME}"."{TABLE_NAME}"
        WHERE measure_name = 'cpu_percent'
            AND measure_value::double > {threshold}
            AND time > ago(1h)
        ORDER BY time DESC
    """
    
    timestream_query = boto3.client('timestream-query')
    response = timestream_query.query(QueryString=query)
    
    if response['Rows']:
        print(f"⚠️  CPU anomalies detected (>{threshold}%):")
        for row in response['Rows']:
            print(row)
    else:
        print("✓ No anomalies detected")

# Loop de monitoramento
hostname = 'web-server-01'
for i in range(10):
    collect_system_metrics(hostname)
    time.sleep(5)

# Checar anomalias
query_anomalies(threshold=80)
```

### 5. Time-Series Analytics

```python
import boto3

timestream_query = boto3.client('timestream-query')

def time_series_analytics(database, table):
    """Analytics avançados de séries temporais"""
    
    # 1. Interpolação
    interpolation_query = f"""
        SELECT 
            interpolate_linear(
                create_time_series(time, measure_value::double),
                SEQUENCE(ago(1h), now(), 1m)
            ) as interpolated_values
        FROM "{database}"."{table}"
        WHERE measure_name = 'temperature'
    """
    
    # 2. Detecção de anomalias
    anomaly_query = f"""
        WITH stats AS (
            SELECT 
                avg(measure_value::double) as mean,
                stddev(measure_value::double) as stddev
            FROM "{database}"."{table}"
            WHERE time > ago(24h)
                AND measure_name = 'temperature'
        )
        SELECT 
            time,
            measure_value::double as value,
            stats.mean,
            stats.stddev
        FROM "{database}"."{table}", stats
        WHERE measure_name = 'temperature'
            AND time > ago(1h)
            AND ABS(measure_value::double - stats.mean) > 3 * stats.stddev
    """
    
    # 3. Agregação por janela de tempo
    windowed_query = f"""
        SELECT 
            bin(time, 5m) as time_window,
            avg(measure_value::double) as avg_value,
            min(measure_value::double) as min_value,
            max(measure_value::double) as max_value,
            count(*) as data_points
        FROM "{database}"."{table}"
        WHERE measure_name = 'temperature'
            AND time > ago(1h)
        GROUP BY bin(time, 5m)
        ORDER BY time_window DESC
    """
    
    # 4. Comparação período anterior
    comparison_query = f"""
        WITH current_period AS (
            SELECT avg(measure_value::double) as current_avg
            FROM "{database}"."{table}"
            WHERE time > ago(1h)
        ),
        previous_period AS (
            SELECT avg(measure_value::double) as previous_avg
            FROM "{database}"."{table}"
            WHERE time BETWEEN ago(2h) AND ago(1h)
        )
        SELECT 
            current_avg,
            previous_avg,
            (current_avg - previous_avg) / previous_avg * 100 as percent_change
        FROM current_period, previous_period
    """
    
    # Executar queries
    results = {}
    for name, query in [
        ('interpolation', interpolation_query),
        ('anomalies', anomaly_query),
        ('windowed', windowed_query),
        ('comparison', comparison_query)
    ]:
        try:
            response = timestream_query.query(QueryString=query)
            results[name] = response['Rows']
            print(f"✓ {name}: {len(response['Rows'])} rows")
        except Exception as e:
            print(f"✗ {name}: {e}")
    
    return results

# Uso
analytics = time_series_analytics('IoTDatabase', 'SensorData')
```

## Integração

### Lambda Function

```python
import json
import boto3
import os

timestream_write = boto3.client('timestream-write')

DATABASE_NAME = os.environ['DATABASE_NAME']
TABLE_NAME = os.environ['TABLE_NAME']

def lambda_handler(event, context):
    """Process IoT events"""
    records = []
    
    for record in event['Records']:
        payload = json.loads(record['body'])
        
        records.append({
            'Dimensions': [
                {'Name': 'device_id', 'Value': payload['device_id']},
                {'Name': 'location', 'Value': payload['location']}
            ],
            'MeasureName': payload['measure_name'],
            'MeasureValue': str(payload['value']),
            'MeasureValueType': 'DOUBLE',
            'Time': str(payload['timestamp']),
            'TimeUnit': 'MILLISECONDS'
        })
    
    # Batch write
    result = timestream_write.write_records(
        DatabaseName=DATABASE_NAME,
        TableName=TABLE_NAME,
        Records=records
    )
    
    return {
        'statusCode': 200,
        'body': json.dumps({
            'records_ingested': result['RecordsIngested']
        })
    }
```

### Grafana Integration

```yaml
# datasource.yaml
apiVersion: 1

datasources:
  - name: Amazon Timestream
    type: grafana-timestream-datasource
    access: proxy
    jsonData:
      authType: default
      defaultRegion: us-east-1
      defaultDatabase: IoTDatabase
      defaultTable: SensorData
```

## Melhores Práticas

### Performance
1. **Batch writes** (até 100 records por batch)
2. **Appropriate data retention** (memory vs magnetic)
3. **Use dimensões** para filtering eficiente
4. **Query optimization** (filter cedo, limit results)
5. **Multi-measure records** para related data

### Custo
1. **Right-size retention** policies
2. **Use magnetic store** para historical data
3. **Efficient queries** (evite full scans)
4. **Batch operations** reduzem API calls
5. **Monitor ingest/query costs**

### Modeling
1. **Common dimensions** para grouping
2. **Meaningful measure names**
3. **Consistent timestamps**
4. **Appropriate granularity**
5. **Schema evolution** strategy

## Limitações

- Max dimensions per record: 128
- Max measures per record: 100 (multi-measure)
- Max batch size: 100 records
- Query timeout: 15 minutes
- Max result set size: 100 MB

## Perguntas e Respostas

### P: Timestream vs CloudWatch Metrics?
**R:** CloudWatch: métricas AWS, simple aggregations, AWS-centric. Timestream: custom time-series, SQL queries complexas, analytics avançados, IoT/DevOps scale. Use Timestream para requirements beyond CloudWatch.

### P: Quando usar memory vs magnetic store?
**R:** Memory: recent data, low latency queries, mais caro. Magnetic: historical data, acceptable latency, 1/10 custo. Typical: 1-7 dias em memory, resto em magnetic. Auto-tiering transparente.

### P: Como funciona data tiering?
**R:** Dados movem automaticamente de memory para magnetic baseado em retention policy. Queries span ambos stores automaticamente. Transparent para application. Configure baseado em access patterns.

### P: Timestream vs InfluxDB?
**R:** Timestream: serverless, auto-scaling, AWS-managed. InfluxDB: self-managed ou cloud, mais control, ecosystem maior. Timestream easier setup, InfluxDB mais features/customization.

### P: Posso deletar dados?
**R:** Sim, mas limitado. TTL automático via retention policies (recommended). DELETE statements para selective deletes. Consider partition strategy para bulk deletes.

### P: Como otimizar query performance?
**R:** 1) Use time filters (critical), 2) Filter dimensions cedo, 3) Limit result sets, 4) Use appropriate aggregations, 5) Consider data granularity, 6) Monitor query metrics.

### P: Batch writes são necessários?
**R:** Recomendado para efficiency. Reduce API calls, improve throughput. Até 100 records per batch. Balance entre latency (real-time) e efficiency (batching).

### P: Como fazer backup?
**R:** Automated backups via magnetic store (até 200 anos retention). Export para S3 usando queries. Continuous replication não built-in. Plan DR strategy.

### P: Timestream vs DynamoDB para time-series?
**R:** Timestream: purpose-built, SQL queries, analytics, compression. DynamoDB: general purpose, NoSQL, mais flexible. Timestream ~1/10 custo para time-series. Use Timestream para time-series workloads.

### P: Como integrar com Grafana?
**R:** Use Grafana Timestream plugin. Configure datasource com AWS credentials. Create dashboards com Timestream queries. Real-time monitoring e alerting.

## Recursos de Aprendizado

- [Timestream Documentation](https://docs.aws.amazon.com/timestream/)
- [Timestream Best Practices](https://docs.aws.amazon.com/timestream/latest/developerguide/best-practices.html)
- [Time Series Query Language](https://docs.aws.amazon.com/timestream/latest/developerguide/reference.html)
- [AWS re:Invent Timestream Sessions](https://www.youtube.com/results?search_query=reinvent+timestream)
- [Timestream Pricing](https://aws.amazon.com/timestream/pricing/)
