# Amazon MSK (Managed Streaming for Apache Kafka)

## O que é Amazon MSK?

Amazon MSK (Managed Streaming for Apache Kafka) é um serviço totalmente gerenciado que facilita a construção e execução de aplicações que usam Apache Kafka para processar streaming data. O MSK gerencia a configuração, provisionamento e manutenção de clusters Apache Kafka.

## Conceitos Principais

### Apache Kafka
Sistema de streaming distribuído para:
- **Message Broker**: Pub/Sub messaging
- **Event Streaming**: Processamento de eventos em tempo real
- **Data Pipeline**: Movimentação de dados entre sistemas
- **Storage**: Armazenamento durável de streams

### Componentes do Kafka

#### 1. Topics
Categorias onde records são publicados.
- Particionados para paralelismo
- Retidos por período configurável
- Replicados para durabilidade

#### 2. Producers
Aplicações que publicam dados em topics.
- Escolhem partition (round-robin, key-based, custom)
- Configuram acks para durabilidade
- Suporte a batching para performance

#### 3. Consumers
Aplicações que leem dados de topics.
- Organizados em consumer groups
- Partition assignment automático
- Controle de offset (at-least-once, exactly-once)

#### 4. Brokers
Servidores Kafka que armazenam dados.
- Gerenciam partitions
- Replicam dados
- Servem requests de producers/consumers

#### 5. ZooKeeper / KRaft
Coordenação de cluster.
- **ZooKeeper**: Modo tradicional (legado)
- **KRaft**: Modo nativo (recomendado para novos clusters)

## Arquitetura MSK

### Deployment

```
┌─────────────────────────────────────┐
│         MSK Cluster                 │
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐
│  │Broker 1 │  │Broker 2 │  │Broker 3 │
│  │  AZ 1   │  │  AZ 2   │  │  AZ 3   │
│  └─────────┘  └─────────┘  └─────────┘
│                                     │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐
│  │  ZK 1   │  │  ZK 2   │  │  ZK 3   │
│  │  AZ 1   │  │  AZ 2   │  │  AZ 3   │
│  └─────────┘  └─────────┘  └─────────┘
└─────────────────────────────────────┘
        ↑                    ↓
    Producers            Consumers
```

### Tipos de Deployment

#### 1. MSK Provisioned
- Você escolhe broker type e storage
- Gerencia scaling manualmente
- Maior controle sobre configuração
- Preço baseado em broker hours

#### 2. MSK Serverless
- Automaticamente escala compute e storage
- Pay-per-use (GB/hour de storage + partition/hour)
- Sem gerenciamento de brokers
- Ideal para workloads variáveis

## Recursos Principais

### 1. Fully Managed
- **Setup Automatizado**: Cluster pronto em minutos
- **Patching**: Atualizações automáticas
- **Recovery**: Substituição automática de brokers
- **Monitoring**: Integração com CloudWatch
- **Scaling**: Vertical e horizontal

### 2. High Availability
- **Multi-AZ**: Deploy em 2 ou 3 AZs
- **Replication**: Dados replicados entre AZs
- **Auto Recovery**: Detecção e recovery automático
- **No Downtime**: Scaling sem interrupção

### 3. Security
- **Encryption in Transit**: TLS 1.2
- **Encryption at Rest**: KMS
- **Authentication**: IAM, SASL/SCRAM, mTLS
- **Authorization**: Kafka ACLs
- **Private Connectivity**: VPC
- **Compliance**: PCI DSS, HIPAA eligible

### 4. MSK Connect
- Kafka Connect totalmente gerenciado
- Conectores para S3, databases, etc.
- Auto-scaling de workers
- Monitoring integrado

### 5. MSK Serverless
- Zero gerenciamento
- Auto-scaling automático
- Pay per use
- Compatible com Kafka clients

## Casos de Uso

### 1. Event Streaming
Processamento de eventos em tempo real.

```java
// Producer example
Properties props = new Properties();
props.put("bootstrap.servers", "b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

Producer<String, String> producer = new KafkaProducer<>(props);
producer.send(new ProducerRecord<>("events", "key", "value"));
producer.close();
```

### 2. Log Aggregation
Centralizar logs de múltiplas aplicações.

### 3. Change Data Capture (CDC)
Capturar mudanças em databases.

### 4. Microservices Communication
Event-driven architecture entre microservices.

### 5. Real-time Analytics
Stream processing com Kafka Streams ou Flink.

```java
// Kafka Streams example
StreamsBuilder builder = new StreamsBuilder();
KStream<String, String> source = builder.stream("input-topic");

source
    .filter((key, value) -> value.contains("error"))
    .to("error-topic");

KafkaStreams streams = new KafkaStreams(builder.build(), props);
streams.start();
```

### 6. Data Integration
Pipeline entre sistemas usando MSK Connect.

## Melhores Práticas

### Cluster Configuration

#### 1. Broker Size
```
t3.small: Dev/Test (até 100 MB/s)
m5.large: Produção light (até 450 MB/s)
m5.xlarge: Produção standard (até 900 MB/s)
m5.2xlarge: High throughput (até 1800 MB/s)
```

#### 2. Storage
- **EBS Volume Type**: Use gp3 para melhor performance
- **Storage Size**: Plan para retention period + growth
- **Auto Scaling**: Enable para storage expansion

```bash
# Exemplo de configuração via CLI
aws kafka create-cluster \
    --cluster-name my-cluster \
    --broker-node-group-info '{
        "InstanceType": "kafka.m5.large",
        "ClientSubnets": ["subnet-1", "subnet-2", "subnet-3"],
        "StorageInfo": {
            "EbsStorageInfo": {
                "ProvisionedThroughput": {
                    "Enabled": true,
                    "VolumeThroughput": 250
                },
                "VolumeSize": 1000
            }
        }
    }' \
    --kafka-version "3.5.1" \
    --number-of-broker-nodes 3
```

### Topic Configuration

#### 1. Partitions
```bash
# Calcular número de partitions
# throughput_target / producer_throughput = número de partitions
# Exemplo: 100 MB/s / 10 MB/s = 10 partitions

# Criar topic com partitions
kafka-topics.sh --create \
    --bootstrap-server $BOOTSTRAP_SERVERS \
    --topic my-topic \
    --partitions 10 \
    --replication-factor 3 \
    --config retention.ms=604800000
```

#### 2. Replication Factor
- **Production**: 3 (minimum)
- **Multi-AZ**: Replicas distribuídas
- **Trade-off**: Durabilidade vs. throughput

#### 3. Retention
```bash
# Configurar retention
kafka-configs.sh --alter \
    --entity-type topics \
    --entity-name my-topic \
    --add-config retention.ms=86400000 \
    --bootstrap-server $BOOTSTRAP_SERVERS
```

### Producer Best Practices

#### 1. Idempotence
```java
props.put("enable.idempotence", true);
props.put("acks", "all");
props.put("retries", Integer.MAX_VALUE);
```

#### 2. Batching
```java
props.put("batch.size", 16384);
props.put("linger.ms", 10);
props.put("compression.type", "snappy");
```

#### 3. Error Handling
```java
producer.send(record, (metadata, exception) -> {
    if (exception != null) {
        // Log and retry
        logger.error("Error sending message", exception);
    }
});
```

### Consumer Best Practices

#### 1. Consumer Groups
```java
props.put("group.id", "my-consumer-group");
props.put("enable.auto.commit", false);
props.put("auto.offset.reset", "earliest");

// Manual commit
consumer.commitSync();
```

#### 2. Parallelism
- Número de consumers ≤ número de partitions
- Scale consumers baseado em lag
- Use consumer groups para load balancing

#### 3. Offset Management
```java
// Store offsets in external system
consumer.seek(partition, offset);

// Exactly-once semantics
props.put("isolation.level", "read_committed");
props.put("enable.auto.commit", false);
```

### Security

#### 1. IAM Authentication
```java
// Configure IAM
props.put("security.protocol", "SASL_SSL");
props.put("sasl.mechanism", "AWS_MSK_IAM");
props.put("sasl.jaas.config", "software.amazon.msk.auth.iam.IAMLoginModule required;");
props.put("sasl.client.callback.handler.class", 
    "software.amazon.msk.auth.iam.IAMClientCallbackHandler");
```

#### 2. Encryption
```bash
# Enable encryption in transit
aws kafka update-security \
    --cluster-arn $CLUSTER_ARN \
    --encryption-info '{
        "EncryptionInTransit": {
            "ClientBroker": "TLS",
            "InCluster": true
        }
    }'
```

#### 3. VPC Configuration
- Coloque cluster em private subnets
- Use security groups para controle de acesso
- VPC endpoints para acesso privado

## MSK Connect

Kafka Connect gerenciado para integração de dados.

### Conectores Populares

#### 1. S3 Sink Connector
```json
{
    "name": "s3-sink-connector",
    "config": {
        "connector.class": "io.confluent.connect.s3.S3SinkConnector",
        "tasks.max": "2",
        "topics": "my-topic",
        "s3.bucket.name": "my-bucket",
        "s3.region": "us-east-1",
        "format.class": "io.confluent.connect.s3.format.json.JsonFormat",
        "flush.size": "1000",
        "storage.class": "io.confluent.connect.s3.storage.S3Storage"
    }
}
```

#### 2. Database Source Connector (Debezium)
```json
{
    "name": "mysql-source",
    "config": {
        "connector.class": "io.debezium.connector.mysql.MySqlConnector",
        "database.hostname": "mysql.example.com",
        "database.port": "3306",
        "database.user": "debezium",
        "database.password": "${file:/secrets/mysql-password}",
        "database.server.id": "184054",
        "database.server.name": "mydb",
        "table.include.list": "inventory.customers,inventory.orders"
    }
}
```

### Deploy Connector
```bash
# Via CLI
aws kafkaconnect create-connector \
    --connector-name my-connector \
    --capacity '{
        "ProvisionedCapacity": {
            "McuCount": 2,
            "WorkerCount": 2
        }
    }' \
    --kafka-cluster '{
        "ApacheKafkaCluster": {
            "BootstrapServers": "b-1.msk.kafka.us-east-1.amazonaws.com:9092",
            "Vpc": {
                "Subnets": ["subnet-1", "subnet-2"],
                "SecurityGroups": ["sg-123"]
            }
        }
    }' \
    --connector-configuration file://connector-config.json
```

## MSK Serverless

Cluster Kafka serverless com auto-scaling.

### Quando Usar
- Workloads variáveis
- Começando com Kafka
- Não quer gerenciar capacity
- Pay-per-use preferível

### Criar Cluster Serverless
```bash
aws kafka create-cluster-v2 \
    --cluster-name my-serverless-cluster \
    --serverless '{
        "VpcConfigs": [{
            "SubnetIds": ["subnet-1", "subnet-2"],
            "SecurityGroupIds": ["sg-123"]
        }],
        "ClientAuthentication": {
            "Sasl": {
                "Iam": {
                    "Enabled": true
                }
            }
        }
    }'
```

### Limitações
- Não suporta public access
- Apenas IAM authentication
- Algumas configurações Kafka não disponíveis
- Replication factor fixo em 3

## Monitoring e Troubleshooting

### CloudWatch Metrics

#### Broker Metrics
```
- CPUUser
- KafkaDataLogsDiskUsed
- MemoryUsed
- NetworkRxPackets/NetworkTxPackets
- BytesInPerSec/BytesOutPerSec
```

#### Topic Metrics
```
- PartitionCount
- MessagesInPerSec
- FetchConsumerTotalTimeMs
- ProduceTotalTimeMs
```

### CloudWatch Logs
```bash
# Enable broker logs
aws kafka update-monitoring \
    --cluster-arn $CLUSTER_ARN \
    --logging-info '{
        "BrokerLogs": {
            "CloudWatchLogs": {
                "Enabled": true,
                "LogGroup": "/aws/msk/my-cluster"
            }
        }
    }'
```

### Common Issues

#### 1. High Consumer Lag
```bash
# Check consumer lag
kafka-consumer-groups.sh \
    --bootstrap-server $BOOTSTRAP_SERVERS \
    --describe \
    --group my-group

# Solutions:
# - Increase partitions
# - Add more consumers
# - Optimize consumer processing
```

#### 2. Connection Issues
```
- Verify security group rules
- Check IAM permissions
- Verify VPC configuration
- Check TLS/authentication settings
```

#### 3. Performance Issues
```
- Monitor CPU/Memory usage
- Check disk I/O
- Review partition distribution
- Optimize producer/consumer configs
```

## Exemplos de Código

### Python Producer
```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8'),
    security_protocol='SASL_SSL',
    sasl_mechanism='AWS_MSK_IAM',
    sasl_oauth_token_provider='software.amazon.msk.auth.iam.IAMOAuthBearerTokenProvider'
)

# Send message
producer.send('my-topic', {'key': 'value'})
producer.flush()
```

### Python Consumer
```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'my-topic',
    bootstrap_servers=['b-1.msk-cluster.kafka.us-east-1.amazonaws.com:9092'],
    group_id='my-group',
    value_deserializer=lambda m: json.loads(m.decode('utf-8')),
    auto_offset_reset='earliest',
    enable_auto_commit=False
)

for message in consumer:
    print(f"Topic: {message.topic}, Partition: {message.partition}, "
          f"Offset: {message.offset}, Value: {message.value}")
    consumer.commit()
```

### Lambda Consumer
```python
import json

def lambda_handler(event, context):
    for partition_key in event['records']:
        partition_values = event['records'][partition_key]
        for record in partition_values:
            # Decode base64 value
            payload = json.loads(record['value'])
            print(f"Received: {payload}")
    
    return {
        'statusCode': 200
    }
```

## Integration com AWS Services

### 1. Lambda
- Event source mapping
- Auto-scaling baseado em lag
- Batch processing

### 2. Kinesis Data Analytics
- SQL queries em streams
- Real-time analytics
- Anomaly detection

### 3. S3
- MSK Connect S3 sink
- Long-term storage
- Data lake integration

### 4. CloudWatch
- Metrics e alarms
- Log aggregation
- Dashboard

### 5. Glue Schema Registry
- Schema management
- Schema evolution
- Compatibility checking

## Migração para MSK

### De Kafka On-Premises

#### 1. MirrorMaker 2
```properties
# mm2.properties
clusters = source, target
source.bootstrap.servers = on-prem-kafka:9092
target.bootstrap.servers = msk-cluster.kafka.us-east-1.amazonaws.com:9092

source->target.enabled = true
source->target.topics = .*
```

#### 2. Strategy
1. Setup MSK cluster
2. Configure MirrorMaker 2
3. Start replication
4. Validate data
5. Switch producers
6. Switch consumers
7. Decommission old cluster

### De Kinesis Data Streams
- Use Lambda para bridge
- Kafka Connect Kinesis source
- Gradual migration

## Pricing

### MSK Provisioned
- **Broker Hours**: Por tipo de instância
- **Storage**: Por GB-month (gp3)
- **Data Transfer**: Cross-AZ e internet

### MSK Serverless
- **Partition Hour**: $0.025/partition/hour
- **Storage**: $0.10/GB-month
- **Data Transfer**: Same as provisioned

### MSK Connect
- **Worker Hours**: Por MCU-hour
- **Plugins**: Sem custo adicional

### Exemplo
```
Cluster: 3 x kafka.m5.large
Storage: 1000 GB gp3 per broker
Region: us-east-1

Monthly Cost:
- Brokers: 3 × $0.21/hour × 730 hours = $460
- Storage: 3 × 1000 GB × $0.10 = $300
- Total: ~$760/month
```

## Comparação com Alternativas

| Feature | MSK | Kinesis | EventBridge | SQS |
|---------|-----|---------|-------------|-----|
| Protocolo | Kafka | Proprietary | Event patterns | SQS |
| Throughput | Muito alto | Alto | Médio | Alto |
| Ordering | Per partition | Per shard | Não garantido | FIFO queues |
| Retention | Configurável | 1-365 dias | 24h replay | 14 dias |
| Replay | Sim | Sim | Sim | Não |
| Managed | Sim | Totalmente | Totalmente | Totalmente |
| Complexity | Médio | Baixo | Baixo | Baixo |

## Recursos de Aprendizado

### Documentação
- [MSK Developer Guide](https://docs.aws.amazon.com/msk/)
- [MSK Connect Guide](https://docs.aws.amazon.com/msk/latest/developerguide/msk-connect.html)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)

### Workshops
- [MSK Labs](https://msk-labs.workshop.aws/)
- [Event-driven Architecture](https://aws.amazon.com/event-driven-architecture/workshops/)

### Ferramentas
- [Kafka CLI Tools](https://kafka.apache.org/quickstart)
- [Conduktor](https://www.conduktor.io/) - Kafka GUI
- [Kafka Manager](https://github.com/yahoo/CMAK) - Cluster management
- [kcat (kafkacat)](https://github.com/edenhill/kcat) - Debugging tool

### Blogs
- [AWS Big Data Blog - MSK](https://aws.amazon.com/blogs/big-data/tag/amazon-msk/)
- [Best Practices for MSK](https://aws.amazon.com/blogs/big-data/best-practices-for-right-sizing-your-apache-kafka-clusters-to-optimize-performance-and-cost/)

---

**Amazon MSK** é ideal para event streaming, log aggregation, e real-time analytics com Apache Kafka totalmente gerenciado. Use MSK Serverless para começar sem gerenciamento de capacity! 🚀
