# Amazon DynamoDB

## O que é DynamoDB?

Amazon DynamoDB é um banco de dados NoSQL totalmente gerenciado, serverless, que fornece performance consistente de milissegundos em qualquer escala.

## Conceitos Principais

### Tables
Container principal para dados.

### Items
Registros individuais na tabela (similar a rows em SQL).

### Attributes
Campos de dados em um item (similar a columns).

### Primary Key
Identificador único para cada item.

**Tipos:**
1. **Partition Key** (Simple): Apenas hash key
2. **Partition Key + Sort Key** (Composite): Hash key + range key

### Data Types

**Scalar:**
- String (S)
- Number (N)
- Binary (B)
- Boolean (BOOL)
- Null (NULL)

**Document:**
- List (L)
- Map (M)

**Set:**
- String Set (SS)
- Number Set (NS)
- Binary Set (BS)

## Modelo de Dados

### Exemplo: E-commerce

```json
{
  "PK": "USER#12345",
  "SK": "PROFILE",
  "name": "João Silva",
  "email": "joao@example.com",
  "created": "2024-01-01T00:00:00Z"
}

{
  "PK": "USER#12345",
  "SK": "ORDER#001",
  "orderId": "ORDER#001",
  "total": 150.00,
  "status": "shipped",
  "items": [
    {"productId": "PROD#001", "quantity": 2}
  ]
}
```

## Indexes

### Global Secondary Index (GSI)
- Partition key e sort key diferentes da tabela base
- Queries em diferentes access patterns
- Própria capacidade de throughput
- Eventually consistent
- Pode ser criado depois da tabela

### Local Secondary Index (LSI)
- Mesma partition key, sort key diferente
- Compartilha throughput com tabela base
- Strongly ou eventually consistent
- Deve ser criado junto com tabela
- Máximo 5 LSIs por tabela

## Capacity Modes

### On-Demand
- Paga por request
- Scaling automático
- Sem planejamento de capacidade
- Ideal para workloads imprevisíveis

**Pricing:**
- Write: $1.25 por milhão WRUs
- Read: $0.25 por milhão RRUs

### Provisioned
- Define WCUs/RCUs
- Auto Scaling disponível
- Mais econômico para workloads previsíveis

**Capacity Units:**
- **1 WCU** = 1 write de até 1 KB/segundo
- **1 RCU** = 1 strongly consistent read de até 4 KB/segundo
- **1 RCU** = 2 eventually consistent reads de até 4 KB/segundo

**Exemplo:**
- Item de 3 KB → 1 WCU
- Item de 5 KB → 2 WCUs
- Read item 10 KB (strong) → 3 RCUs
- Read item 10 KB (eventual) → 1.5 RCUs

## Operações

### PutItem
Cria ou substitui item.

```python
table.put_item(
    Item={
        'userId': '12345',
        'name': 'João Silva',
        'email': 'joao@example.com'
    }
)
```

### GetItem
Recupera item por primary key.

```python
response = table.get_item(
    Key={
        'userId': '12345'
    }
)
item = response['Item']
```

### UpdateItem
Atualiza atributos específicos.

```python
table.update_item(
    Key={'userId': '12345'},
    UpdateExpression='SET email = :email',
    ExpressionAttributeValues={
        ':email': 'newemail@example.com'
    }
)
```

### DeleteItem
Remove item.

```python
table.delete_item(
    Key={'userId': '12345'}
)
```

### Query
Busca items com mesma partition key.

```python
response = table.query(
    KeyConditionExpression=Key('userId').eq('12345')
)
```

### Scan
Lê toda a tabela (cuidado com custo!).

```python
response = table.scan(
    FilterExpression=Attr('status').eq('active')
)
```

### BatchGetItem / BatchWriteItem
Operações em batch (até 25 items).

## Recursos Avançados

### DynamoDB Streams
Capture mudanças em tempo real.

**Eventos:**
- INSERT
- MODIFY
- REMOVE

**Casos de Uso:**
- Replicação
- Triggers (Lambda)
- Audit logging
- Real-time analytics

**View Types:**
- KEYS_ONLY
- NEW_IMAGE
- OLD_IMAGE
- NEW_AND_OLD_IMAGES

### Global Tables
Multi-region, multi-active replication.

**Características:**
- Active-active replication
- Conflict resolution (last write wins)
- Baixa latência global
- DR automático

### Point-in-Time Recovery (PITR)
Backup contínuo dos últimos 35 dias.

**Recursos:**
- Restore para qualquer momento
- Sem impacto em performance
- Restore em nova tabela

### On-Demand Backup
Backups manuais.

**Características:**
- Retenção ilimitada
- Restore em nova tabela
- Sem impacto em performance

### DynamoDB Accelerator (DAX)
Cache in-memory.

**Características:**
- Microsegundos de latência
- Fully managed
- Compatible com DynamoDB API
- Write-through cache
- Até 10x performance improvement

**Casos de Uso:**
- Read-heavy workloads
- Latência crítica
- Reduzir RCUs

### Transactions
ACID transactions em até 100 items.

```python
response = dynamodb.transact_write_items(
    TransactItems=[
        {
            'Put': {
                'TableName': 'Orders',
                'Item': {'orderId': '001', 'status': 'pending'}
            }
        },
        {
            'Update': {
                'TableName': 'Inventory',
                'Key': {'productId': 'PROD001'},
                'UpdateExpression': 'SET quantity = quantity - :val',
                'ExpressionAttributeValues': {':val': 1}
            }
        }
    ]
)
```

### Time To Live (TTL)
Delete automático de items expirados.

```python
table.update_item(
    Key={'userId': '12345'},
    UpdateExpression='SET expireAt = :ttl',
    ExpressionAttributeValues={
        ':ttl': int(time.time()) + 86400  # 24 horas
    }
)
```

## Design Patterns

### Single Table Design
Armazene múltiplas entidades em uma tabela.

**Benefícios:**
- Menos tabelas para gerenciar
- Joins via queries (não scan)
- Melhor performance
- Custo reduzido

**Exemplo:**
```
PK                  SK              Attributes
USER#12345         PROFILE         name, email
USER#12345         ORDER#001       total, status
PRODUCT#001        METADATA        name, price
PRODUCT#001        REVIEW#001      rating, comment
```

### Adjacency List
Relacionamentos usando PK e SK.

### Composite Keys
Multiple attributes em chave.

### Sparse Indexes
GSI com subset de items.

## Melhores Práticas

### Data Modeling
- Design para access patterns, não normalização
- Use single table design quando possível
- Partition key com alta cardinalidade
- Evite hot partitions
- Use sort keys para range queries

### Performance
- Use eventually consistent reads quando possível
- Batch operations
- Use projection expressions (limitar atributos)
- Cache com DAX para reads frequentes
- Parallel scans para grandes datasets

### Custo
- On-demand para workloads imprevisíveis
- Provisioned com auto-scaling para previsíveis
- Use Reserved Capacity para savings
- Delete unused indexes
- Implement TTL para cleanup automático
- Compress large attributes

### Segurança
- Use IAM roles
- Fine-grained access control
- Encryption at rest (default)
- VPC Endpoints para acesso privado
- Enable CloudTrail logging
- Backup regular

## Casos de Uso

1. **Gaming Leaderboards**: Sort key com scores
2. **Session Management**: TTL para cleanup
3. **IoT Telemetry**: Time-series data
4. **User Profiles**: Fast read/write
5. **Shopping Carts**: Real-time updates
6. **Mobile Backends**: Offline sync
7. **Metadata Storage**: Key-value pairs
8. **Real-time Voting**: Atomic counters

## Comparação com Outros Databases

| Característica | DynamoDB | RDS | MongoDB |
|---------------|----------|-----|---------|
| **Type** | NoSQL | SQL | NoSQL |
| **Management** | Fully managed | Managed | Self/Managed |
| **Scaling** | Horizontal | Vertical | Horizontal |
| **Schema** | Flexible | Fixed | Flexible |
| **Transactions** | Limited | Full ACID | Full ACID |
| **Joins** | No | Yes | Lookup |
| **Use Case** | Web scale | Complex queries | Document store |

## Limitações

- Item size: Máximo 400 KB
- Partition key max 2048 bytes
- Sort key max 1024 bytes
- No joins nativos
- Batch operations: 25 items max
- Transaction: 100 items max
- Query results: 1 MB max
- Local Secondary Index: 5 max

## Exemplo Completo: Python

```python
import boto3
from boto3.dynamodb.conditions import Key, Attr
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Create
table.put_item(
    Item={
        'userId': '12345',
        'name': 'João Silva',
        'email': 'joao@example.com',
        'balance': Decimal('100.50'),
        'tags': ['premium', 'verified']
    }
)

# Read
response = table.get_item(Key={'userId': '12345'})
user = response.get('Item')

# Update
table.update_item(
    Key={'userId': '12345'},
    UpdateExpression='ADD balance :val',
    ExpressionAttributeValues={':val': Decimal('50')}
)

# Query
response = table.query(
    IndexName='email-index',
    KeyConditionExpression=Key('email').eq('joao@example.com')
)

# Scan with filter
response = table.scan(
    FilterExpression=Attr('tags').contains('premium')
)

# Delete
table.delete_item(Key={'userId': '12345'})

# Batch write
with table.batch_writer() as batch:
    for i in range(100):
        batch.put_item(Item={'userId': str(i), 'name': f'User {i}'})

# Transaction
dynamodb_client = boto3.client('dynamodb')
response = dynamodb_client.transact_write_items(
    TransactItems=[
        {
            'Update': {
                'TableName': 'Users',
                'Key': {'userId': {'S': '12345'}},
                'UpdateExpression': 'ADD balance :val',
                'ExpressionAttributeValues': {':val': {'N': '-50'}}
            }
        },
        {
            'Put': {
                'TableName': 'Transactions',
                'Item': {
                    'transactionId': {'S': 'TXN001'},
                    'userId': {'S': '12345'},
                    'amount': {'N': '-50'}
                }
            }
        }
    ]
)
```

## Comandos AWS CLI

```bash
# Create table
aws dynamodb create-table \
    --table-name Users \
    --attribute-definitions \
        AttributeName=userId,AttributeType=S \
    --key-schema \
        AttributeName=userId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST

# Put item
aws dynamodb put-item \
    --table-name Users \
    --item '{"userId":{"S":"12345"},"name":{"S":"João Silva"}}'

# Get item
aws dynamodb get-item \
    --table-name Users \
    --key '{"userId":{"S":"12345"}}'

# Query
aws dynamodb query \
    --table-name Users \
    --key-condition-expression "userId = :userId" \
    --expression-attribute-values '{":userId":{"S":"12345"}}'

# Scan
aws dynamodb scan --table-name Users

# Enable streams
aws dynamodb update-table \
    --table-name Users \
    --stream-specification \
        StreamEnabled=true,StreamViewType=NEW_AND_OLD_IMAGES
```

## Monitoramento

### CloudWatch Metrics
- ConsumedReadCapacityUnits
- ConsumedWriteCapacityUnits
- UserErrors
- SystemErrors
- ThrottledRequests

### Alarms Recomendados
- High throttles
- High user errors
- Capacity utilization > 80%

## Recursos de Aprendizado

- [DynamoDB Developer Guide](https://docs.aws.amazon.com/dynamodb/)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [The DynamoDB Book](https://www.dynamodbbook.com/)
- [AWS re:Invent DynamoDB Sessions](https://www.youtube.com/results?search_query=reinvent+dynamodb)
