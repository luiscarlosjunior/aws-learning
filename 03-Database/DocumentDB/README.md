# Amazon DocumentDB

## O que é DocumentDB?

Amazon DocumentDB é um banco de dados de documentos totalmente gerenciado, compatível com MongoDB, que facilita armazenar, consultar e indexar dados JSON. Oferece a performance, escalabilidade e disponibilidade necessárias para aplicações críticas.

## Compatibilidade MongoDB

**Versões suportadas:**
- MongoDB 3.6
- MongoDB 4.0
- MongoDB 5.0

**APIs compatíveis:**
- find, insert, update, delete
- Aggregation framework
- Indexes (single field, compound, text)
- Transactions
- Change streams

**Limitações:**
- Algumas operations não suportadas (ex: map-reduce)
- Algumas aggregation stages limitadas
- Client-side field-level encryption não suportado

## Arquitetura

```
┌─────────────────────────────────────┐
│      DocumentDB Cluster             │
│                                     │
│  Primary Instance                   │
│  (Read + Write)                     │
│         │                           │
│         ├─ Replica 1 (Read)        │
│         ├─ Replica 2 (Read)        │
│         └─ Replica 3 (Read)        │
│                                     │
│  ┌────────────────────────────┐   │
│  │  Shared Storage Volume      │   │
│  │  (6 copies across 3 AZs)   │   │
│  └────────────────────────────┘   │
└─────────────────────────────────────┘
```

## Conceitos Principais

### Collections e Documents

```javascript
// Document exemplo
{
  "_id": ObjectId("507f1f77bcf86cd799439011"),
  "name": "João Silva",
  "email": "joao@example.com",
  "age": 30,
  "address": {
    "street": "Rua das Flores",
    "city": "São Paulo",
    "state": "SP"
  },
  "tags": ["premium", "verified"],
  "created_at": ISODate("2024-01-01T10:00:00Z")
}
```

### Indexes

```javascript
// Single field index
db.users.createIndex({ email: 1 })

// Compound index
db.orders.createIndex({ user_id: 1, created_at: -1 })

// Text index
db.products.createIndex({ description: "text" })

// Unique index
db.users.createIndex({ email: 1 }, { unique: true })

// TTL index (auto-delete após expiry)
db.sessions.createIndex({ created_at: 1 }, { expireAfterSeconds: 3600 })
```

## Exemplos Práticos

### 1. Python (PyMongo)

```python
from pymongo import MongoClient
from datetime import datetime

# Conectar
client = MongoClient(
    'mongodb://admin:password@my-docdb.cluster-xxx.us-east-1.docdb.amazonaws.com:27017/?tls=true&tlsCAFile=rds-combined-ca-bundle.pem&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false'
)

db = client.myapp
users = db.users

# Insert
user_id = users.insert_one({
    'name': 'João Silva',
    'email': 'joao@example.com',
    'age': 30,
    'created_at': datetime.utcnow()
}).inserted_id

print(f"User created: {user_id}")

# Find
user = users.find_one({'email': 'joao@example.com'})
print(user)

# Update
users.update_one(
    {'_id': user_id},
    {'$set': {'age': 31, 'updated_at': datetime.utcnow()}}
)

# Delete
users.delete_one({'_id': user_id})

# Bulk operations
users.insert_many([
    {'name': 'Alice', 'age': 25},
    {'name': 'Bob', 'age': 28},
    {'name': 'Charlie', 'age': 35}
])

# Query com filtros
adults = users.find({'age': {'$gte': 18}})
for user in adults:
    print(user)

client.close()
```

### 2. Aggregation Pipeline

```python
from pymongo import MongoClient

client = MongoClient('mongodb://...')
db = client.analytics

# Pipeline de agregação
pipeline = [
    # Filter
    {
        '$match': {
            'created_at': {
                '$gte': datetime(2024, 1, 1)
            }
        }
    },
    # Group e agregação
    {
        '$group': {
            '_id': '$category',
            'total_sales': {'$sum': '$amount'},
            'avg_sale': {'$avg': '$amount'},
            'count': {'$sum': 1}
        }
    },
    # Sort
    {
        '$sort': {'total_sales': -1}
    },
    # Limit
    {
        '$limit': 10
    }
]

results = list(db.sales.aggregate(pipeline))

for result in results:
    print(f"Category: {result['_id']}")
    print(f"Total Sales: ${result['total_sales']:,.2f}")
    print(f"Avg Sale: ${result['avg_sale']:,.2f}")
    print(f"Count: {result['count']}")
    print("---")
```

### 3. Node.js (MongoDB Driver)

```javascript
const { MongoClient } = require('mongodb');

const uri = 'mongodb://admin:password@my-docdb.cluster-xxx.us-east-1.docdb.amazonaws.com:27017/?tls=true&replicaSet=rs0&readPreference=secondaryPreferred&retryWrites=false';

async function main() {
  const client = new MongoClient(uri, {
    tlsCAFile: './rds-combined-ca-bundle.pem'
  });

  try {
    await client.connect();
    console.log('Connected to DocumentDB');

    const db = client.db('myapp');
    const users = db.collection('users');

    // Insert
    const result = await users.insertOne({
      name: 'João Silva',
      email: 'joao@example.com',
      age: 30,
      created_at: new Date()
    });
    console.log(`User created: ${result.insertedId}`);

    // Find
    const user = await users.findOne({ email: 'joao@example.com' });
    console.log('User:', user);

    // Update
    await users.updateOne(
      { _id: result.insertedId },
      { $set: { age: 31, updated_at: new Date() } }
    );

    // Find with projection
    const projection = { name: 1, email: 1, _id: 0 };
    const userInfo = await users.findOne(
      { email: 'joao@example.com' },
      { projection }
    );
    console.log('User info:', userInfo);

    // Complex query
    const activeUsers = await users.find({
      age: { $gte: 18, $lte: 65 },
      status: 'active',
      'address.city': 'São Paulo'
    }).toArray();
    console.log(`Found ${activeUsers.length} active users`);

  } finally {
    await client.close();
  }
}

main().catch(console.error);
```

### 4. E-commerce Application

```python
from pymongo import MongoClient
from bson import ObjectId
from datetime import datetime

client = MongoClient('mongodb://...')
db = client.ecommerce

class EcommerceDB:
    def __init__(self):
        self.products = db.products
        self.orders = db.orders
        self.customers = db.customers
    
    def create_product(self, name, price, category, stock):
        """Criar produto"""
        return self.products.insert_one({
            'name': name,
            'price': price,
            'category': category,
            'stock': stock,
            'created_at': datetime.utcnow()
        }).inserted_id
    
    def create_order(self, customer_id, items):
        """Criar pedido com validação de estoque"""
        # Verificar estoque
        for item in items:
            product = self.products.find_one({'_id': item['product_id']})
            if not product or product['stock'] < item['quantity']:
                raise ValueError(f"Insufficient stock for {item['product_id']}")
        
        # Calcular total
        total = 0
        order_items = []
        for item in items:
            product = self.products.find_one({'_id': item['product_id']})
            item_total = product['price'] * item['quantity']
            total += item_total
            
            order_items.append({
                'product_id': item['product_id'],
                'product_name': product['name'],
                'quantity': item['quantity'],
                'unit_price': product['price'],
                'subtotal': item_total
            })
        
        # Criar ordem
        order_id = self.orders.insert_one({
            'customer_id': customer_id,
            'items': order_items,
            'total': total,
            'status': 'pending',
            'created_at': datetime.utcnow()
        }).inserted_id
        
        # Atualizar estoque
        for item in items:
            self.products.update_one(
                {'_id': item['product_id']},
                {'$inc': {'stock': -item['quantity']}}
            )
        
        return order_id
    
    def get_customer_orders(self, customer_id):
        """Buscar pedidos do cliente"""
        return list(self.orders.find(
            {'customer_id': customer_id}
        ).sort('created_at', -1))
    
    def get_sales_report(self, start_date, end_date):
        """Relatório de vendas"""
        pipeline = [
            {
                '$match': {
                    'created_at': {'$gte': start_date, '$lte': end_date},
                    'status': {'$in': ['completed', 'shipped']}
                }
            },
            {
                '$group': {
                    '_id': {
                        'year': {'$year': '$created_at'},
                        'month': {'$month': '$created_at'}
                    },
                    'total_orders': {'$sum': 1},
                    'total_revenue': {'$sum': '$total'},
                    'avg_order_value': {'$avg': '$total'}
                }
            },
            {
                '$sort': {'_id': 1}
            }
        ]
        
        return list(self.orders.aggregate(pipeline))

# Uso
store = EcommerceDB()

# Criar produtos
product1 = store.create_product('Laptop', 1200.00, 'Electronics', 50)
product2 = store.create_product('Mouse', 25.00, 'Accessories', 200)

# Criar pedido
customer_id = ObjectId()
order_id = store.create_order(customer_id, [
    {'product_id': product1, 'quantity': 1},
    {'product_id': product2, 'quantity': 2}
])

print(f"Order created: {order_id}")

# Relatório
report = store.get_sales_report(datetime(2024, 1, 1), datetime(2024, 12, 31))
for month_data in report:
    print(f"Month: {month_data['_id']}")
    print(f"Revenue: ${month_data['total_revenue']:,.2f}")
```

### 5. Change Streams

```python
from pymongo import MongoClient

client = MongoClient('mongodb://...')
db = client.myapp

# Watch changes em uma collection
with db.orders.watch() as stream:
    for change in stream:
        print(f"Change detected:")
        print(f"Operation: {change['operationType']}")
        
        if change['operationType'] == 'insert':
            print(f"New document: {change['fullDocument']}")
        elif change['operationType'] == 'update':
            print(f"Updated fields: {change['updateDescription']}")
        elif change['operationType'] == 'delete':
            print(f"Deleted document ID: {change['documentKey']}")
```

### 6. Transactions

```python
from pymongo import MongoClient

client = MongoClient('mongodb://...')
db = client.banking

def transfer_money(from_account, to_account, amount):
    """Transferência com transaction"""
    with client.start_session() as session:
        with session.start_transaction():
            # Debitar origem
            result = db.accounts.update_one(
                {'_id': from_account, 'balance': {'$gte': amount}},
                {'$inc': {'balance': -amount}},
                session=session
            )
            
            if result.modified_count == 0:
                session.abort_transaction()
                raise ValueError("Insufficient funds")
            
            # Creditar destino
            db.accounts.update_one(
                {'_id': to_account},
                {'$inc': {'balance': amount}},
                session=session
            )
            
            # Log transação
            db.transactions.insert_one({
                'from': from_account,
                'to': to_account,
                'amount': amount,
                'timestamp': datetime.utcnow()
            }, session=session)
            
            print("Transfer completed successfully")

# Uso
try:
    transfer_money(
        from_account=ObjectId('507f1f77bcf86cd799439011'),
        to_account=ObjectId('507f1f77bcf86cd799439012'),
        amount=100.00
    )
except ValueError as e:
    print(f"Transfer failed: {e}")
```

## High Availability

### Read Replicas

```bash
# Criar replica
aws docdb create-db-instance \
    --db-instance-identifier docdb-replica-1 \
    --db-instance-class db.r5.large \
    --engine docdb \
    --db-cluster-identifier my-docdb-cluster
```

**Connection string com replicas:**
```
mongodb://admin:password@instance1:27017,instance2:27017,instance3:27017/?replicaSet=rs0&readPreference=secondaryPreferred
```

### Failover Automático

- Detecta falha do primary
- Promove replica automaticamente
- RTO: ~30 segundos
- Zero data loss (replicação síncrona)

## Backup e Recovery

```bash
# Snapshot manual
aws docdb create-db-cluster-snapshot \
    --db-cluster-identifier my-docdb \
    --db-cluster-snapshot-identifier my-snapshot-20240101

# Restore
aws docdb restore-db-cluster-from-snapshot \
    --db-cluster-identifier my-docdb-restored \
    --snapshot-identifier my-snapshot-20240101 \
    --engine docdb
```

## Migração de MongoDB

### Usando mongodump/mongorestore

```bash
# Export de MongoDB existente
mongodump --uri="mongodb://localhost:27017/myapp" --out=/tmp/dump

# Import para DocumentDB
mongorestore \
    --uri="mongodb://admin:password@my-docdb.cluster-xxx.us-east-1.docdb.amazonaws.com:27017/?tls=true&replicaSet=rs0&retryWrites=false" \
    --tlsCAFile=rds-combined-ca-bundle.pem \
    /tmp/dump
```

### Usando AWS DMS

```bash
# Replicação contínua com DMS
aws dms create-replication-task \
    --replication-task-identifier mongodb-to-docdb \
    --source-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:MONGODB_SOURCE \
    --target-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:DOCDB_TARGET \
    --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:INSTANCE \
    --migration-type full-load-and-cdc
```

## Performance

### Indexing Strategy

```javascript
// Analyze query performance
db.users.explain('executionStats').find({ email: 'joao@example.com' })

// Compound index para queries múltiplas
db.orders.createIndex({ user_id: 1, created_at: -1, status: 1 })

// Partial index (economia de espaço)
db.orders.createIndex(
  { status: 1 },
  { partialFilterExpression: { status: { $eq: 'pending' } } }
)
```

### Monitoring

```python
# CloudWatch metrics via boto3
import boto3

cloudwatch = boto3.client('cloudwatch')

# Get CPU utilization
response = cloudwatch.get_metric_statistics(
    Namespace='AWS/DocDB',
    MetricName='CPUUtilization',
    Dimensions=[
        {'Name': 'DBInstanceIdentifier', 'Value': 'my-docdb-instance'}
    ],
    StartTime=datetime.utcnow() - timedelta(hours=1),
    EndTime=datetime.utcnow(),
    Period=300,
    Statistics=['Average']
)

for datapoint in response['Datapoints']:
    print(f"Time: {datapoint['Timestamp']}, CPU: {datapoint['Average']:.2f}%")
```

## Segurança

```bash
# VPC, encryption, TLS
aws docdb create-db-cluster \
    --db-cluster-identifier my-docdb \
    --engine docdb \
    --master-username admin \
    --master-user-password SecurePass123! \
    --vpc-security-group-ids sg-12345678 \
    --db-subnet-group-name my-subnet-group \
    --storage-encrypted \
    --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678 \
    --enable-cloudwatch-logs-exports audit profiler
```

## Melhores Práticas

1. **Use índices apropriados** - analyze queries com explain()
2. **Connection pooling** - reuse connections
3. **Read replicas** para distribuir leitura
4. **Appropriate read preference** - secondaryPreferred para analytics
5. **Monitor performance** - CloudWatch metrics
6. **Regular backups** - automated + manual snapshots
7. **Schema design** - embed vs reference baseado em access patterns

## Limitações

- Max document size: 16 MB
- Max index keys per document: 1000
- Max indexes per collection: 64
- Algumas operações MongoDB não suportadas
- retryWrites deve ser false

## Perguntas e Respostas

### P: DocumentDB vs DynamoDB?
**R:** DocumentDB: queries flexíveis, aggregations complexas, MongoDB compatibility. DynamoDB: serverless, horizontal scale ilimitado, key-value simples. Use DocumentDB para queries ad-hoc e schema flexível.

### P: DocumentDB é 100% compatível com MongoDB?
**R:** Compatível com MongoDB 3.6, 4.0, 5.0 APIs mas não 100%. Principais diferenças: map-reduce não suportado, algumas aggregation stages limitadas, client-side encryption não disponível. Test aplicação antes de migrar.

### P: Como escalar DocumentDB?
**R:** Vertical: aumentar instance class. Horizontal: adicionar read replicas (até 15). Storage auto-scale. Considere sharding no application layer se necessário write scaling.

### P: Change Streams vs DynamoDB Streams?
**R:** Similar concept: capture changes em tempo real. DocumentDB change streams seguem MongoDB API. DynamoDB Streams AWS-specific. Ambos úteis para triggers e replicação.

### P: Quando usar embed vs reference?
**R:** Embed: dados acessados juntos, relacionamento 1-to-few, update patterns simples. Reference: dados grandes, relacionamento many-to-many, dados compartilhados. Balance entre performance e consistency.

### P: DocumentDB vs RDS PostgreSQL com JSONB?
**R:** DocumentDB: schema flexível nativo, MongoDB API, horizontal scaling easier. PostgreSQL: ACID forte, SQL + JSON, complex joins. Escolha baseado em workload predominante.

### P: Como migrar de MongoDB Atlas?
**R:** mongodump/mongorestore para small datasets. AWS DMS para continuous replication (large/live). Test connectivity e compatibility. Plan cutover window.

### P: Performance de aggregations?
**R:** Use índices para $match stages. Limit results cedo. Use $project para reduzir data transferido. Consider materialized views pattern. Monitor query profiler.

### P: Connection pooling é necessário?
**R:** Sim! Criar connections é caro. Use connection pools em drivers. Configure size baseado em concurrent users. Monitor connection count no CloudWatch.

### P: Como fazer backup incremental?
**R:** DocumentDB não suporta incremental backups direto. Use change streams para CDC. Ou automated backups (point-in-time recovery). Copy snapshots para DR.

## Recursos de Aprendizado

- [DocumentDB Documentation](https://docs.aws.amazon.com/documentdb/)
- [DocumentDB Best Practices](https://docs.aws.amazon.com/documentdb/latest/developerguide/best-practices.html)
- [MongoDB Documentation](https://docs.mongodb.com/)
- [AWS re:Invent DocumentDB Sessions](https://www.youtube.com/results?search_query=reinvent+documentdb)
- [DocumentDB Pricing](https://aws.amazon.com/documentdb/pricing/)
