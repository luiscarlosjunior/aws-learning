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

## Troubleshooting Comum

### ProvisionedThroughputExceededException
**Problema**: Throttling de requests
**Soluções**:
- Habilitar Auto Scaling
- Aumentar WCUs/RCUs provisionados
- Mudar para On-Demand mode
- Implementar exponential backoff
- Distribuir carga uniformemente (evitar hot partitions)
- Usar cache (DAX)

### ValidationException
**Problema**: Erro de validação em operações
**Soluções**:
- Verificar tipos de dados (String, Number, Binary)
- Confirmar que chaves estão corretas
- Validar expression syntax
- Verificar tamanho do item (max 400 KB)

### ItemCollectionSizeLimitExceededException
**Problema**: Collection size > 10 GB
**Soluções**:
- Redesenhar partition key para melhor distribuição
- Usar composite keys diferentes
- Considerar múltiplas tabelas
- Remover dados desnecessários

### ConditionalCheckFailedException
**Problema**: Condição de escrita falhou
**Soluções**:
- Verificar se condition expression está correta
- Confirmar que item existe (para updates)
- Implementar retry logic se apropriado
- Usar transactions se precisa de atomicidade

## Perguntas e Respostas (Q&A)

### Conceitos Fundamentais

**P: Quando devo usar DynamoDB vs RDS?**
R: Use DynamoDB quando:
- Precisa de escala horizontal massiva (milhões de requests/segundo)
- Latência consistente de milissegundos é crítica
- Modelo serverless é preferido
- Schema flexível é necessário
- Carga de trabalho é key-value ou document-based

Use RDS quando:
- Precisa de queries SQL complexas com JOINs
- ACID transactions são críticas
- Schema relacional bem definido
- Já tem aplicação SQL existente
- Ferramentas de BI precisam conectar diretamente

**P: Como funciona o partitioning no DynamoDB?**
R: DynamoDB usa a partition key para distribuir dados. Hash da partition key determina em qual partition física o item é armazenado. Por isso é crucial escolher uma partition key com alta cardinalidade para distribuir carga uniformemente.

**P: Diferença entre partition key e sort key?**
R: 
- Partition key (hash key): Determina a partition onde o item é armazenado
- Sort key (range key): Ordena items dentro da mesma partition
- Juntas formam uma composite primary key
- Partition key sozinha = simple primary key

**P: DynamoDB é eventually consistent ou strongly consistent?**
R: Por padrão é eventually consistent (mais rápido, usa menos RCUs). Você pode solicitar strongly consistent reads quando necessário, mas isso usa 2x mais RCUs e tem latência ligeiramente maior.

### Modelagem de Dados

**P: Devo usar single table design ou múltiplas tabelas?**
R: Single table design é recomendado para:
- Aplicações complexas com múltiplas entidades relacionadas
- Quando precisa de queries eficientes entre entidades
- Para minimizar custos (menos tabelas = menos WCUs/RCUs totais)

Múltiplas tabelas fazem sentido para:
- Aplicações simples com entidades independentes
- Quando equipes diferentes gerenciam diferentes dados
- Para compliance/segurança (separação de dados sensíveis)

**P: Como fazer JOINs no DynamoDB?**
R: DynamoDB não tem JOINs nativos. Estratégias:
1. Denormalizar dados (duplicar onde necessário)
2. Usar single table design com composite keys
3. Fazer múltiplas queries na aplicação
4. Usar adjacency list pattern
5. Usar GSIs para diferentes access patterns

**P: Como escolher uma boa partition key?**
R: Características de uma boa partition key:
- Alta cardinalidade (muitos valores únicos)
- Distribuição uniforme de acesso
- Não concentra hot spots
- Permite queries eficientes

Exemplos:
- ✅ BOM: userId, orderId, deviceId
- ❌ RUIM: status (poucos valores), country (hot spots), date (acesso recente concentrado)

**P: Quantos GSIs devo criar?**
R: Crie um GSI para cada access pattern diferente que sua aplicação precisa. Máximo 20 GSIs por tabela. Considere:
- Cada GSI tem custo (storage + throughput)
- GSIs aumentam latência de writes (eventual consistency)
- Sparse indexes (GSIs com subset de items) são eficientes

### Performance e Custos

**P: On-Demand vs Provisioned - qual escolher?**
R: 
**On-Demand**: 
- Workloads imprevisíveis ou spiky
- Novas tabelas sem histórico
- Aplicações que aceitam até 2.5x custo extra
- Quando não quer gerenciar capacidade

**Provisioned**:
- Workloads previsíveis e consistentes
- Alto volume (mais cost-effective)
- Pode usar Reserved Capacity
- Com Auto Scaling bem configurado

**P: Como calcular RCUs e WCUs necessários?**
R: 
**WCUs**: 1 WCU = 1 write de até 1 KB/segundo
- Item 3.5 KB = 4 WCUs
- 100 writes/segundo de 2 KB = 200 WCUs

**RCUs** (strongly consistent): 1 RCU = 1 read de até 4 KB/segundo
- Item 10 KB = 3 RCUs
- 100 reads/segundo de 5 KB = 200 RCUs

**RCUs** (eventually consistent): 2 reads por RCU
- 100 reads/segundo de 5 KB = 100 RCUs

**P: DAX vale a pena? Quando usar?**
R: Use DAX quando:
- Workload read-heavy (>80% reads)
- Precisa de latência de microsegundos
- Muitas queries repetidas
- Pode tolerar eventually consistent data
- Custo adicional é aceitável ($0.12/hora por nó small)

Não use se:
- Workload write-heavy
- Dados mudam muito frequentemente
- Strongly consistent reads são críticos
- Budget limitado

**P: Como reduzir custos no DynamoDB?**
R: 
1. Use on-demand apenas se realmente necessário
2. Configure Auto Scaling em provisioned mode
3. Use Reserved Capacity (1-3 anos, até 77% desconto)
4. Delete GSIs não utilizados
5. Implemente TTL para cleanup automático
6. Use projection expressions (buscar apenas campos necessários)
7. Batch operations quando possível
8. Comprima large attributes
9. Use eventually consistent reads quando apropriado
10. Archive dados antigos para S3

### Transações e Consistência

**P: DynamoDB Transactions são ACID compliant?**
R: Sim, transactions (TransactWriteItems/TransactReadItems) são ACID:
- Atomicidade: Todas operações succedem ou falham juntas
- Consistência: Constraints são mantidas
- Isolamento: Serializable isolation
- Durabilidade: Writes são persistentes

Limitações: Max 100 items, 4 MB total, custo 2x dos writes normais

**P: Como implementar contador atômico?**
R: Use UpdateExpression com ADD:

```python
table.update_item(
    Key={'userId': '12345'},
    UpdateExpression='ADD views :inc',
    ExpressionAttributeValues={':inc': 1}
)
```

Isso é atômico mesmo com requests concorrentes.

**P: Como garantir idempotência em writes?**
R: Estratégias:
1. Use conditional writes com chave de idempotência
2. Use transactions
3. Gere IDs únicos (UUIDs) no client
4. Implemente deduplication window
5. Use version numbers ou timestamps

### Streams e Eventos

**P: DynamoDB Streams vs Kinesis Data Streams?**
R: 
**DynamoDB Streams**:
- Integrado nativamente
- 24 horas de retenção
- Não requer capacity planning
- Free (limitado a 2 readers simultâneos)

**Kinesis Data Streams para DynamoDB**:
- Até 1 ano de retenção
- Mais de 2 consumidores paralelos
- Integração com Kinesis ecosystem
- Custo adicional

**P: Como processar DynamoDB Streams com Lambda?**
R: Lambda processa Streams em batches:
- Polling automático
- Batch size configurável (1-10000 records)
- Retry automático em falhas
- Dead Letter Queue para falhas persistentes
- Filtros de eventos disponíveis

### Backup e Recuperação

**P: Qual a diferença entre PITR e On-Demand Backup?**
R: 
**PITR (Point-in-Time Recovery)**:
- Contínuo, últimos 35 dias
- Restore para qualquer segundo
- Custo: storage dos dados
- Recovery em nova tabela

**On-Demand Backup**:
- Manual ou agendado
- Retenção ilimitada
- Restore exato do momento
- Útil para compliance

**P: Backups afetam performance?**
R: Não! Tanto PITR quanto on-demand backups não consomem throughput provisionado e não impactam latência de requests.

### Segurança

**P: Como implementar multi-tenancy seguro?**
R: Estratégias:
1. **Tabela por tenant**: Isolamento máximo, gestão complexa
2. **Partition key = tenantId**: Eficiente, usar IAM condition keys
3. **Tenant ID como prefixo**: Flexível, requer validação na app
4. **Fine-grained access control**: IAM policy com condições

**P: Como usar IAM para acesso granular?**
R: Use IAM conditions:

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "dynamodb:GetItem",
      "dynamodb:Query"
    ],
    "Resource": "arn:aws:dynamodb:region:account:table/Users",
    "Condition": {
      "ForAllValues:StringEquals": {
        "dynamodb:LeadingKeys": ["${aws:userid}"]
      }
    }
  }]
}
```

Usuário só acessa items onde partition key = seu user ID.

## Exemplos Avançados

### Exemplo 1: E-commerce Order System com Single Table Design

```python
import boto3
from datetime import datetime
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('EcommerceApp')

class OrderService:
    def create_order(self, user_id, items):
        """Cria pedido com transaction para garantir atomicidade"""
        order_id = f"ORDER#{datetime.now().strftime('%Y%m%d%H%M%S')}"
        timestamp = datetime.now().isoformat()
        
        # Calcular total
        total = sum(item['price'] * item['quantity'] for item in items)
        
        # Transaction items
        transact_items = []
        
        # 1. Criar order
        transact_items.append({
            'Put': {
                'TableName': 'EcommerceApp',
                'Item': {
                    'PK': {'S': f'USER#{user_id}'},
                    'SK': {'S': order_id},
                    'EntityType': {'S': 'ORDER'},
                    'OrderId': {'S': order_id},
                    'Status': {'S': 'PENDING'},
                    'Total': {'N': str(total)},
                    'Items': {'L': [
                        {'M': {
                            'ProductId': {'S': item['product_id']},
                            'Quantity': {'N': str(item['quantity'])},
                            'Price': {'N': str(item['price'])}
                        }} for item in items
                    ]},
                    'CreatedAt': {'S': timestamp}
                }
            }
        })
        
        # 2. Decrementar inventory para cada item
        for item in items:
            transact_items.append({
                'Update': {
                    'TableName': 'EcommerceApp',
                    'Key': {
                        'PK': {'S': f"PRODUCT#{item['product_id']}"},
                        'SK': {'S': 'INVENTORY'}
                    },
                    'UpdateExpression': 'SET Stock = Stock - :qty',
                    'ConditionExpression': 'Stock >= :qty',
                    'ExpressionAttributeValues': {
                        ':qty': {'N': str(item['quantity'])}
                    }
                }
            })
        
        # 3. Criar index entry por status para queries
        transact_items.append({
            'Put': {
                'TableName': 'EcommerceApp',
                'Item': {
                    'PK': {'S': 'STATUS#PENDING'},
                    'SK': {'S': f"{timestamp}#{order_id}"},
                    'EntityType': {'S': 'ORDER_INDEX'},
                    'UserId': {'S': user_id},
                    'OrderId': {'S': order_id},
                    'Total': {'N': str(total)}
                }
            }
        })
        
        # Executar transaction
        client = boto3.client('dynamodb')
        try:
            client.transact_write_items(TransactItems=transact_items)
            return {'success': True, 'order_id': order_id}
        except Exception as e:
            if 'TransactionCanceledException' in str(e):
                return {'success': False, 'error': 'Stock insuficiente ou outro erro'}
            raise
    
    def get_user_orders(self, user_id):
        """Busca todos os pedidos de um usuário"""
        response = table.query(
            KeyConditionExpression='PK = :pk AND begins_with(SK, :sk)',
            ExpressionAttributeValues={
                ':pk': f'USER#{user_id}',
                ':sk': 'ORDER#'
            }
        )
        return response['Items']
    
    def get_orders_by_status(self, status):
        """Busca pedidos por status (usa GSI ou secondary index)"""
        response = table.query(
            KeyConditionExpression='PK = :pk',
            ExpressionAttributeValues={
                ':pk': f'STATUS#{status}'
            },
            ScanIndexForward=False,  # Mais recentes primeiro
            Limit=50
        )
        return response['Items']
    
    def update_order_status(self, user_id, order_id, new_status):
        """Atualiza status do pedido"""
        old_status_needed = True
        
        # Primeiro, get para saber o status antigo
        response = table.get_item(
            Key={
                'PK': f'USER#{user_id}',
                'SK': order_id
            }
        )
        
        if 'Item' not in response:
            return {'success': False, 'error': 'Order not found'}
        
        old_status = response['Item']['Status']
        timestamp = response['Item']['CreatedAt']
        
        # Transaction para update status em múltiplos places
        client = boto3.client('dynamodb')
        client.transact_write_items(
            TransactItems=[
                # Update main order record
                {
                    'Update': {
                        'TableName': 'EcommerceApp',
                        'Key': {
                            'PK': {'S': f'USER#{user_id}'},
                            'SK': {'S': order_id}
                        },
                        'UpdateExpression': 'SET #status = :new_status, UpdatedAt = :now',
                        'ExpressionAttributeNames': {'#status': 'Status'},
                        'ExpressionAttributeValues': {
                            ':new_status': {'S': new_status},
                            ':now': {'S': datetime.now().isoformat()}
                        }
                    }
                },
                # Delete old status index
                {
                    'Delete': {
                        'TableName': 'EcommerceApp',
                        'Key': {
                            'PK': {'S': f'STATUS#{old_status}'},
                            'SK': {'S': f"{timestamp}#{order_id}"}
                        }
                    }
                },
                # Create new status index
                {
                    'Put': {
                        'TableName': 'EcommerceApp',
                        'Item': {
                            'PK': {'S': f'STATUS#{new_status}'},
                            'SK': {'S': f"{timestamp}#{order_id}"},
                            'EntityType': {'S': 'ORDER_INDEX'},
                            'UserId': {'S': user_id},
                            'OrderId': {'S': order_id}
                        }
                    }
                }
            ]
        )
        
        return {'success': True}

# Uso
service = OrderService()

# Criar pedido
result = service.create_order(
    user_id='USER123',
    items=[
        {'product_id': 'PROD001', 'quantity': 2, 'price': Decimal('29.99')},
        {'product_id': 'PROD002', 'quantity': 1, 'price': Decimal('49.99')}
    ]
)

# Buscar pedidos do usuário
orders = service.get_user_orders('USER123')

# Buscar pedidos pendentes
pending = service.get_orders_by_status('PENDING')

# Atualizar status
service.update_order_status('USER123', 'ORDER#20241204120000', 'SHIPPED')
```

### Exemplo 2: Time-Series Data com TTL

```python
import boto3
from datetime import datetime, timedelta
import time

class IoTDataService:
    def __init__(self):
        self.dynamodb = boto3.resource('dynamodb')
        self.table = self.dynamodb.Table('IoTMetrics')
    
    def setup_table(self):
        """Criar tabela com TTL habilitado"""
        client = boto3.client('dynamodb')
        
        # Criar tabela
        client.create_table(
            TableName='IoTMetrics',
            KeySchema=[
                {'AttributeName': 'DeviceId', 'KeyType': 'HASH'},
                {'AttributeName': 'Timestamp', 'KeyType': 'RANGE'}
            ],
            AttributeDefinitions=[
                {'AttributeName': 'DeviceId', 'AttributeType': 'S'},
                {'AttributeName': 'Timestamp', 'AttributeType': 'N'}
            ],
            BillingMode='PAY_PER_REQUEST'
        )
        
        # Aguardar tabela estar ativa
        waiter = client.get_waiter('table_exists')
        waiter.wait(TableName='IoTMetrics')
        
        # Habilitar TTL
        client.update_time_to_live(
            TableName='IoTMetrics',
            TimeToLiveSpecification={
                'Enabled': True,
                'AttributeName': 'ExpiresAt'
            }
        )
        
        print("Tabela criada com TTL habilitado")
    
    def record_metric(self, device_id, metric_type, value, retention_days=7):
        """Registra métrica com TTL automático"""
        timestamp = int(time.time())
        expires_at = timestamp + (retention_days * 86400)
        
        self.table.put_item(
            Item={
                'DeviceId': device_id,
                'Timestamp': timestamp,
                'MetricType': metric_type,
                'Value': value,
                'RecordedAt': datetime.now().isoformat(),
                'ExpiresAt': expires_at  # TTL attribute
            }
        )
    
    def batch_record_metrics(self, metrics):
        """Registro em batch para eficiência"""
        with self.table.batch_writer() as batch:
            for metric in metrics:
                timestamp = int(time.time())
                expires_at = timestamp + (metric.get('retention_days', 7) * 86400)
                
                batch.put_item(Item={
                    'DeviceId': metric['device_id'],
                    'Timestamp': timestamp,
                    'MetricType': metric['type'],
                    'Value': metric['value'],
                    'RecordedAt': datetime.now().isoformat(),
                    'ExpiresAt': expires_at
                })
    
    def query_device_metrics(self, device_id, hours_back=24):
        """Query métricas recentes de um device"""
        start_time = int(time.time()) - (hours_back * 3600)
        
        response = self.table.query(
            KeyConditionExpression='DeviceId = :device AND #ts >= :start',
            ExpressionAttributeNames={'#ts': 'Timestamp'},
            ExpressionAttributeValues={
                ':device': device_id,
                ':start': start_time
            },
            ScanIndexForward=False  # Mais recentes primeiro
        )
        
        return response['Items']
    
    def get_latest_metric(self, device_id):
        """Pega métrica mais recente"""
        response = self.table.query(
            KeyConditionExpression='DeviceId = :device',
            ExpressionAttributeValues={':device': device_id},
            ScanIndexForward=False,
            Limit=1
        )
        
        if response['Items']:
            return response['Items'][0]
        return None
    
    def aggregate_metrics(self, device_id, hours_back=1):
        """Agrega métricas (exemplo: média)"""
        metrics = self.query_device_metrics(device_id, hours_back)
        
        if not metrics:
            return None
        
        # Calcular agregações
        values = [m['Value'] for m in metrics]
        return {
            'device_id': device_id,
            'count': len(values),
            'avg': sum(values) / len(values),
            'min': min(values),
            'max': max(values),
            'period': f'{hours_back}h'
        }

# Uso
service = IoTDataService()

# Registrar métricas individuais
service.record_metric('DEVICE001', 'temperature', 23.5, retention_days=30)
service.record_metric('DEVICE001', 'humidity', 65.2, retention_days=30)

# Registro em batch (mais eficiente)
metrics = [
    {'device_id': 'DEVICE001', 'type': 'temperature', 'value': 24.1, 'retention_days': 7},
    {'device_id': 'DEVICE001', 'type': 'humidity', 'value': 64.8, 'retention_days': 7},
    {'device_id': 'DEVICE002', 'type': 'temperature', 'value': 22.3, 'retention_days': 7}
]
service.batch_record_metrics(metrics)

# Query métricas
recent = service.query_device_metrics('DEVICE001', hours_back=24)
latest = service.get_latest_metric('DEVICE001')
aggregated = service.aggregate_metrics('DEVICE001', hours_back=1)

print(f"Agregação: {aggregated}")
```

### Exemplo 3: Caching com DAX

```python
import boto3
from amazondax import AmazonDaxClient

class CachedDataService:
    def __init__(self, use_dax=True):
        if use_dax:
            # Conectar ao DAX cluster
            self.client = AmazonDaxClient(endpoint_url='dax://my-cluster.abc123.dax-clusters.us-east-1.amazonaws.com')
        else:
            # Fallback para DynamoDB direto
            self.client = boto3.client('dynamodb')
        
        self.table_name = 'Products'
    
    def get_product(self, product_id):
        """
        Get com DAX - primeira chamada vai para DynamoDB,
        chamadas subsequentes vêm do cache DAX (microsegundos)
        """
        response = self.client.get_item(
            TableName=self.table_name,
            Key={'ProductId': {'S': product_id}}
        )
        return response.get('Item')
    
    def get_products_batch(self, product_ids):
        """Batch get com DAX"""
        keys = [{'ProductId': {'S': pid}} for pid in product_ids]
        
        response = self.client.batch_get_item(
            RequestItems={
                self.table_name: {
                    'Keys': keys
                }
            }
        )
        
        return response['Responses'].get(self.table_name, [])
    
    def query_category(self, category):
        """
        Query com DAX - resultados são cached
        Ideal para queries frequentemente repetidas
        """
        response = self.client.query(
            TableName=self.table_name,
            IndexName='CategoryIndex',
            KeyConditionExpression='Category = :cat',
            ExpressionAttributeValues={
                ':cat': {'S': category}
            }
        )
        
        return response.get('Items', [])
    
    def update_product(self, product_id, updates):
        """
        Writes invalidam cache automaticamente no DAX
        """
        update_expression = 'SET ' + ', '.join([f'{k} = :{k}' for k in updates.keys()])
        expression_values = {f':{k}': {'S': str(v)} for k, v in updates.items()}
        
        self.client.update_item(
            TableName=self.table_name,
            Key={'ProductId': {'S': product_id}},
            UpdateExpression=update_expression,
            ExpressionAttributeValues=expression_values
        )

# Comparação de performance
import time

def benchmark_with_without_dax():
    # Sem DAX
    service_no_dax = CachedDataService(use_dax=False)
    start = time.time()
    for _ in range(100):
        service_no_dax.get_product('PROD001')
    no_dax_time = time.time() - start
    
    # Com DAX (após cache warming)
    service_dax = CachedDataService(use_dax=True)
    service_dax.get_product('PROD001')  # Warm up cache
    start = time.time()
    for _ in range(100):
        service_dax.get_product('PROD001')
    dax_time = time.time() - start
    
    print(f"Sem DAX: {no_dax_time:.3f}s")
    print(f"Com DAX: {dax_time:.3f}s")
    print(f"Speedup: {no_dax_time/dax_time:.1f}x")
```

## Recursos de Aprendizado

- [DynamoDB Developer Guide](https://docs.aws.amazon.com/dynamodb/)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
- [The DynamoDB Book](https://www.dynamodbbook.com/) by Alex DeBrie
- [AWS re:Invent DynamoDB Sessions](https://www.youtube.com/results?search_query=reinvent+dynamodb)
- [DynamoDB Guide](https://www.dynamodbguide.com/)
- [AWS Workshops - DynamoDB](https://workshops.aws/categories/DynamoDB)
- [DynamoDB Toolbox](https://github.com/jeremydaly/dynamodb-toolbox) - Library to simplify operations
