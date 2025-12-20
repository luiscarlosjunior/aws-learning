# Capítulo 3: Nonrelational Stores (Armazenamento Não Relacional)

O armazenamento não relacional, também conhecido como NoSQL (Not Only SQL), representa uma mudança de paradigma fundamental no gerenciamento de dados. Enquanto bancos de dados relacionais dominaram por décadas, a explosão de dados da web, IoT, redes sociais e aplicações distribuídas criou necessidades que os sistemas relacionais tradicionais lutavam para atender. Bancos de dados NoSQL emergiram para resolver problemas específicos de escalabilidade, flexibilidade e performance em escala massiva.

Como observa Pramod J. Sadalage em "NoSQL Distilled": "NoSQL databases are built to allow the insertion of data without a predefined schema" ("Bancos de dados NoSQL são construídos para permitir a inserção de dados sem um schema predefinido") (Sadalage & Fowler, 2012).

---

## Nonrelational Database Concepts (Conceitos de Banco de Dados Não Relacionais)

### Fundamentos Teóricos

Bancos de dados não relacionais surgiram para resolver limitações específicas dos sistemas relacionais em contextos de:
- **Escala Web**: Bilhões de usuários, petabytes de dados
- **Velocidade**: Latência de milissegundos ou menos
- **Variedade**: Dados estruturados, semi-estruturados e não estruturados
- **Volume**: Crescimento exponencial de dados

**Diferenças Fundamentais vs Bancos Relacionais:**

| Característica | Relacional (SQL) | Não Relacional (NoSQL) |
|----------------|------------------|------------------------|
| Schema | Fixo, definido antecipadamente | Flexível, schema-less ou schema-on-read |
| Escalabilidade | Vertical (scale-up) | Horizontal (scale-out) |
| Transações | ACID completo | BASE (eventual consistency) |
| Queries | SQL padronizado, JOINs complexos | APIs específicas, sem JOINs nativos |
| Data Model | Tabelas relacionadas | Documentos, Key-Value, Colunas, Grafos |
| Consistência | Strong consistency | Eventual consistency (tunable) |

### Teorema CAP

O Teorema CAP (Consistency, Availability, Partition Tolerance) é fundamental para entender trade-offs em sistemas distribuídos.

**Conceito:**
Em um sistema distribuído, é impossível garantir simultaneamente:
- **C (Consistency)**: Todos os nós veem os mesmos dados ao mesmo tempo
- **A (Availability)**: Toda requisição recebe resposta (sucesso ou falha)
- **P (Partition Tolerance)**: Sistema continua operando apesar de partições de rede

**Importante**: Em sistemas distribuídos modernos, partições de rede são inevitáveis, então **deve-se escolher entre Consistency e Availability**.

```python
# Exemplos de trade-offs CAP:

# CP System (Consistency + Partition Tolerance)
# Sacrifica Availability durante partições
class CPDatabase:
    """
    Exemplo: HBase, MongoDB (com WriteConcern=majority)
    
    Durante partição de rede:
    - Nodes minoritários rejeitam writes/reads
    - Garante consistency mas perde availability
    """
    def write(self, key, value):
        # Requer quorum (maioria dos nodes)
        if not self.has_quorum():
            raise UnavailableException("No quorum - rejecting write")
        
        # Escreve em maioria dos nodes
        self.replicate_to_majority(key, value)
        return "OK"
    
    # Trade-off: Durante partição, minority partition fica indisponível
    # Benefício: Zero inconsistências, todos leem mesmos dados

# AP System (Availability + Partition Tolerance)
# Sacrifica Consistency durante partições
class APDatabase:
    """
    Exemplo: Cassandra, DynamoDB (eventual consistency)
    
    Durante partição de rede:
    - Todos os nodes aceitam writes/reads
    - Pode haver inconsistências temporárias
    """
    def write(self, key, value):
        # Aceita write mesmo sem quorum
        self.write_local(key, value)
        
        # Async: Tenta replicar para outros nodes
        self.async_replicate(key, value)
        
        return "OK"
    
    def read(self, key):
        # Lê do node local (pode estar desatualizado)
        value = self.read_local(key)
        
        # Background: Anti-entropy repair reconcilia diferenças
        return value
    
    # Trade-off: Pode ler dados desatualizados
    # Benefício: Sistema sempre disponível, mesmo durante partições

# Exemplo Real - Partição de Rede:
# 
# Sistema com 3 nodes: A, B, C
# Partição: A fica isolado, B e C se comunicam
#
# CP System (HBase):
# - A rejeita writes/reads (minority)
# - B e C aceitam writes/reads (majority)
# - Result: 66% availability, 100% consistency
#
# AP System (Cassandra):
# - A, B, C aceitam writes/reads
# - Pode haver versões conflitantes
# - Result: 100% availability, eventual consistency
```

**Prova do Teorema CAP:**

```
Cenário: Sistema com 2 nodes (A e B), partição de rede os separa

1. Cliente escreve X=1 no node A
2. Partição de rede: A e B não se comunicam
3. Cliente lê X do node B

Impossível ter C + A + P:
- Se queremos C: B deve retornar X=1, mas não recebeu update (requer comunicação)
- Se queremos A: B deve responder, mas só pode retornar valor antigo (inconsistente)
- P é assumido (partição existe)

Conclusão: Deve escolher entre C ou A quando P acontece
```

**Referência:** Brewer, E. (2000). "Towards Robust Distributed Systems". *PODC Keynote*.

### PACELC Theorem

Extensão do CAP que considera latência.

**Conceito:**
- **if Partition**: escolha entre **A**vailability e **C**onsistency
- **else (no partition)**: escolha entre **L**atency e **C**onsistency

```python
# Exemplos de classificação PACELC:

# DynamoDB: PA/EL
# - Durante partição (P): Availability sobre Consistency
# - Normal (E): Latency sobre Consistency
# Trade-off: Máxima disponibilidade e performance, eventual consistency

# MongoDB: PC/EC
# - Durante partição (P): Consistency sobre Availability
# - Normal (E): Consistency sobre Latency
# Trade-off: Strong consistency, menor performance

# Cassandra (tunable): PA/EL ou PC/EC
# - Configurável via Consistency Level
cassandra_config = {
    'write_consistency': 'QUORUM',  # PC/EC
    'read_consistency': 'ONE',      # PA/EL
    # Mix: Strong writes, fast reads
}
```

**Referência:** Abadi, D. (2012). "Consistency Tradeoffs in Modern Distributed Database System Design". *IEEE Computer*, 45(2), 37-42.

### Vantagens dos Bancos Não Relacionais

**1. Escalabilidade Horizontal Nativa**

```python
# Problema com bancos relacionais:
# Escalar horizontalmente requer sharding manual complexo

# Solução NoSQL: Sharding/partitioning automático
class DistributedNoSQLCluster:
    """
    DynamoDB, Cassandra: Auto-sharding transparente
    """
    def __init__(self, nodes):
        self.nodes = nodes
        self.consistent_hash_ring = ConsistentHashRing(nodes)
    
    def write(self, key, value):
        # Determina nodes automaticamente via hash
        primary_nodes = self.consistent_hash_ring.get_nodes(key, count=3)
        
        # Escreve em múltiplos nodes para replicação
        for node in primary_nodes:
            node.write(key, value)
    
    def add_node(self, new_node):
        # Adiciona node ao cluster
        self.consistent_hash_ring.add_node(new_node)
        
        # Rebalanceamento automático
        # Apenas 1/N dos dados movem (N = número de nodes)
        self.rebalance()
    
# Benefício:
# - Adicionar node: Minutos (vs dias/semanas com SQL sharding)
# - Transparente para aplicação
# - Linear scalability: 10 nodes = 10x capacity

# Exemplo real - DynamoDB:
# Capacidade: 20M requests/second, 100+ TB
# Nodes: Centenas (gerenciado pela AWS)
# Adicionar capacity: Clique em botão ou auto-scaling
```

**2. Performance para Workloads Específicos**

```python
# Key-Value Store: Ultra-baixa latência
class KeyValueStore:
    """
    Redis, DynamoDB: <1ms latency
    """
    def get(self, key):
        # O(1) lookup, sem query planning
        # Direto ao node via hash
        return self.hash_table[key]
    
    # vs SQL:
    # SELECT * FROM table WHERE id = key
    # - Query parsing
    # - Query planning
    # - Index lookup (even with B-tree)
    # - Result: 5-50ms
    
# Performance comparison:
# Redis GET: 0.1-0.5ms
# PostgreSQL SELECT by PK: 1-5ms
# Speedup: 10-50x

# Document Store: Queries flexíveis sem JOINs
class DocumentStore:
    """
    MongoDB, DocumentDB: Documentos completos em uma leitura
    """
    def get_user_with_orders(self, user_id):
        # Documento denormalizado
        return {
            'user_id': user_id,
            'name': 'John Doe',
            'email': 'john@example.com',
            'orders': [  # Embedded!
                {'order_id': 1, 'total': 100},
                {'order_id': 2, 'total': 200}
            ]
        }
        # 1 read operation, 1-5ms
    
    # vs SQL (normalized):
    # SELECT * FROM users WHERE user_id = ?
    # SELECT * FROM orders WHERE user_id = ?
    # - 2 queries, 2 network round-trips
    # - JOIN overhead
    # - Result: 10-30ms
```

**3. Flexibilidade de Schema**

```python
# Evolução de schema sem downtime
class SchemaEvolution:
    """
    MongoDB: Schema-less, evolução natural
    """
    # Versão 1: Estrutura inicial
    user_v1 = {
        '_id': 'user123',
        'name': 'John Doe',
        'email': 'john@example.com'
    }
    
    # Versão 2: Adicionar campo (sem migration!)
    user_v2 = {
        '_id': 'user456',
        'name': 'Jane Smith',
        'email': 'jane@example.com',
        'phone': '+1234567890',  # Novo campo!
        'address': {  # Novo objeto nested!
            'street': '123 Main St',
            'city': 'San Francisco'
        }
    }
    
    # Versão 3: Diferentes estruturas coexistem
    user_v3 = {
        '_id': 'user789',
        'name': 'Bob Johnson',
        'contacts': {  # Estrutura completamente diferente!
            'emails': ['bob1@example.com', 'bob2@example.com'],
            'phones': ['+1111111111', '+2222222222']
        },
        'preferences': {
            'language': 'en',
            'timezone': 'PST'
        }
    }
    
    # Application code: Handle múltiplas versões
    def get_user_email(self, user_doc):
        # Backward compatibility
        if 'email' in user_doc:
            return user_doc['email']
        elif 'contacts' in user_doc and 'emails' in user_doc['contacts']:
            return user_doc['contacts']['emails'][0]
        else:
            return None
    
# vs SQL: Requer ALTER TABLE, downtime ou complex migration
# 
# -- SQL migration (pode levar horas em tabelas grandes)
# ALTER TABLE users ADD COLUMN phone VARCHAR(20);
# ALTER TABLE users ADD COLUMN address_street VARCHAR(100);
# ALTER TABLE users ADD COLUMN address_city VARCHAR(50);
# -- Durante migration: Tabela locked ou degraded performance
```

### Desvantagens dos Bancos Não Relacionais

**1. Eventual Consistency Challenges**

```python
# Problema: Read-your-writes pode falhar
class EventualConsistencyIssue:
    """
    Cenário real: E-commerce checkout
    """
    def checkout_flow_problem(self, user_id):
        # 1. User atualiza endereço
        dynamodb.put_item(
            TableName='Users',
            Item={'user_id': user_id, 'address': 'New Address'}
        )
        
        # 2. Imediatamente cria order
        # Problem: Pode ler endereço antigo!
        user = dynamodb.get_item(
            TableName='Users',
            Key={'user_id': user_id}
        )
        
        address = user['address']  # Pode ser endereço antigo!
        
        # Order criado com endereço errado
        create_order(user_id, address)  # BUG!
    
    # Solução: Strongly consistent read
    def checkout_flow_fixed(self, user_id):
        # 1. User atualiza endereço
        dynamodb.put_item(
            TableName='Users',
            Item={'user_id': user_id, 'address': 'New Address'}
        )
        
        # 2. Lê com strong consistency
        user = dynamodb.get_item(
            TableName='Users',
            Key={'user_id': user_id},
            ConsistentRead=True  # Strong consistency!
        )
        
        address = user['address']  # Garantido correto
        create_order(user_id, address)  # OK!
    
    # Trade-off:
    # - Strong consistency: 2x latency (5-10ms vs 1-5ms)
    # - Menor availability durante partições
    # - Mas necessário para casos críticos
```

**2. Falta de JOINs Nativos**

```python
# Problema: Queries relacionais complexas
class NoJoinProblem:
    """
    Obter todos orders de users em uma cidade
    """
    # SQL (easy):
    sql_query = """
        SELECT o.order_id, o.total, u.name
        FROM orders o
        JOIN users u ON o.user_id = u.user_id
        WHERE u.city = 'San Francisco'
    """
    # 1 query, milliseconds
    
    # NoSQL (hard):
    def get_orders_by_city_nosql(self, city):
        # 1. Encontrar users na cidade
        users = dynamodb.query(
            TableName='Users',
            IndexName='CityIndex',
            KeyConditionExpression='city = :city',
            ExpressionAttributeValues={':city': city}
        )
        
        user_ids = [u['user_id'] for u in users]
        
        # 2. Para cada user, buscar orders (N queries!)
        all_orders = []
        for user_id in user_ids:  # N+1 query problem
            orders = dynamodb.query(
                TableName='Orders',
                KeyConditionExpression='user_id = :user_id',
                ExpressionAttributeValues={':user_id': user_id}
            )
            all_orders.extend(orders)
        
        return all_orders
        
        # Performance:
        # - 1000 users na cidade
        # - 1001 queries (1 para users + 1000 para orders)
        # - Latency: 1001 × 5ms = 5 seconds!
    
    # Solução 1: Denormalization
    order_denormalized = {
        'order_id': 'order123',
        'user_id': 'user456',
        'user_name': 'John Doe',  # Denormalized!
        'user_city': 'San Francisco',  # Denormalized!
        'total': 100
    }
    # Agora pode query orders diretamente por city
    # Trade-off: Redundância, inconsistência potencial
    
    # Solução 2: Application-level JOIN
    # Batch get para reduzir queries
    def batch_get_orders(self, user_ids):
        # DynamoDB BatchGetItem: até 100 items por request
        batches = [user_ids[i:i+100] for i in range(0, len(user_ids), 100)]
        
        all_orders = []
        for batch in batches:
            response = dynamodb.batch_get_item(
                RequestItems={
                    'Orders': {
                        'Keys': [{'user_id': uid} for uid in batch]
                    }
                }
            )
            all_orders.extend(response['Responses']['Orders'])
        
        # Performance:
        # - 1000 users = 10 batch requests (vs 1000 individual)
        # - Latency: 10 × 10ms = 100ms (vs 5 seconds)
        # Speedup: 50x
```

**3. Transações Limitadas**

```python
# Problema: Transações multi-item complexas
class LimitedTransactions:
    """
    Transferência bancária em NoSQL
    """
    # SQL: Trivial com ACID
    sql_transaction = """
        BEGIN TRANSACTION;
        UPDATE accounts SET balance = balance - 100 WHERE id = 'A';
        UPDATE accounts SET balance = balance + 100 WHERE id = 'B';
        COMMIT;
    """
    # Atomicidade garantida
    
    # NoSQL: Mais complexo
    def transfer_nosql(self, from_account, to_account, amount):
        # DynamoDB transactions: Limitações
        # - Máximo 25 items por transação
        # - Mesma região apenas
        # - 2x custo de throughput
        # - Latência maior (10-30ms vs 2-5ms)
        
        try:
            dynamodb_client.transact_write_items(
                TransactItems=[
                    {
                        'Update': {
                            'TableName': 'Accounts',
                            'Key': {'account_id': from_account},
                            'UpdateExpression': 'SET balance = balance - :amount',
                            'ConditionExpression': 'balance >= :amount',  # Prevent overdraft
                            'ExpressionAttributeValues': {':amount': amount}
                        }
                    },
                    {
                        'Update': {
                            'TableName': 'Accounts',
                            'Key': {'account_id': to_account},
                            'UpdateExpression': 'SET balance = balance + :amount',
                            'ExpressionAttributeValues': {':amount': amount}
                        }
                    }
                ]
            )
        except ClientError as e:
            if e.response['Error']['Code'] == 'TransactionCanceledException':
                # Condition failed ou conflict
                raise InsufficientFundsError()
    
    # Alternativa: Saga Pattern (eventual consistency)
    def transfer_saga(self, from_account, to_account, amount):
        # 1. Registrar transaction intent
        tx_id = create_transaction_log(from_account, to_account, amount)
        
        # 2. Debit from_account
        try:
            debit(from_account, amount, tx_id)
        except Exception:
            mark_transaction_failed(tx_id)
            return
        
        # 3. Credit to_account
        try:
            credit(to_account, amount, tx_id)
            mark_transaction_completed(tx_id)
        except Exception:
            # Compensating transaction: Refund
            compensate_debit(from_account, amount, tx_id)
            mark_transaction_failed(tx_id)
        
        # Trade-off:
        # - Eventual consistency (not immediate)
        # - Complexidade alta (error handling, compensation)
        # - Benefício: Escalabilidade, nenhuma limitação de items
```

---

## Schema Flexibility (Flexibilidade de Schema)

### Conceitos Fundamentais

Schema flexibility é uma das características mais distintivas de bancos NoSQL. Enquanto bancos relacionais requerem schema rígido definido antecipadamente, NoSQL permite evolução orgânica e heterogeneidade de estruturas.

### Schema-on-Write vs Schema-on-Read

**Schema-on-Write (Bancos Relacionais):**

```sql
-- Deve definir schema antes de inserir dados
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    name VARCHAR(200) NOT NULL,
    price DECIMAL(10,2) NOT NULL,
    category VARCHAR(50) NOT NULL
);

-- Inserção validada contra schema
INSERT INTO products VALUES (1, 'Laptop', 999.99, 'Electronics');

-- Mudança de schema requer ALTER TABLE
ALTER TABLE products ADD COLUMN weight_kg DECIMAL(5,2);
-- Em tabelas grandes: downtime ou performance degradation
```

**Schema-on-Read (Bancos NoSQL):**

```python
# MongoDB: Nenhum schema predefinido necessário
db.products.insert_one({
    'product_id': 1,
    'name': 'Laptop',
    'price': 999.99,
    'category': 'Electronics'
})

# Documentos diferentes coexistem naturalmente
db.products.insert_one({
    'product_id': 2,
    'name': 'T-Shirt',
    'price': 19.99,
    'category': 'Clothing',
    'size': 'M',  # Campo adicional!
    'color': 'Blue',  # Outro campo adicional!
    'material': 'Cotton'  # Mais um!
})

# Schema é enforced durante leitura (application code)
def get_product_details(product):
    details = {
        'name': product['name'],
        'price': product['price']
    }
    
    # Handle campos opcionais
    if 'size' in product:
        details['size'] = product['size']
    if 'color' in product:
        details['color'] = product['color']
    if 'weight_kg' in product:
        details['weight'] = f"{product['weight_kg']} kg"
    
    return details
```

### Padrões de Schema Flexibility

**1. Polymorphic Pattern (Documentos Heterogêneos)**

```python
# Diferentes tipos de products no mesmo collection
class PolymorphicProductCatalog:
    """
    Produtos de diferentes categorias têm atributos diferentes
    """
    # Livro
    book = {
        '_id': 'prod001',
        'type': 'book',
        'name': 'Designing Data-Intensive Applications',
        'price': 45.99,
        'author': 'Martin Kleppmann',
        'isbn': '978-1449373320',
        'pages': 616,
        'publisher': "O'Reilly"
    }
    
    # Eletrônico
    laptop = {
        '_id': 'prod002',
        'type': 'electronics',
        'name': 'MacBook Pro',
        'price': 2499.99,
        'brand': 'Apple',
        'processor': 'M2 Max',
        'ram_gb': 32,
        'storage_gb': 1024,
        'screen_inches': 16
    }
    
    # Roupa
    shirt = {
        '_id': 'prod003',
        'type': 'clothing',
        'name': 'Cotton T-Shirt',
        'price': 29.99,
        'brand': 'Nike',
        'sizes': ['S', 'M', 'L', 'XL'],
        'colors': ['black', 'white', 'blue'],
        'material': 'cotton'
    }
    
    # Query: Funciona em todos os tipos
    def search_products(self, max_price):
        return db.products.find({
            'price': {'$lt': max_price}
        })
    
    # Application code: Handle diferentes tipos
    def render_product(self, product):
        common = f"{product['name']} - ${product['price']}"
        
        if product['type'] == 'book':
            return f"{common} by {product['author']}"
        elif product['type'] == 'electronics':
            return f"{common} - {product['brand']} {product.get('processor', '')}"
        elif product['type'] == 'clothing':
            sizes = ', '.join(product['sizes'])
            return f"{common} - Sizes: {sizes}"
        
        return common

# Benefício: Schema evolui naturalmente com negócio
# Não precisa ALTER TABLE para cada novo tipo de produto
# Flexibilidade para adicionar atributos específicos
```

**2. Attribute Pattern (Atributos Dinâmicos)**

```python
# Produtos com atributos customizáveis
class AttributePattern:
    """
    E-commerce com filtros dinâmicos
    """
    # Laptop com specs técnicas
    laptop_with_attributes = {
        '_id': 'prod100',
        'name': 'Gaming Laptop',
        'price': 1599.99,
        'category': 'Electronics',
        'attributes': [  # Array de key-value pairs
            {'key': 'processor', 'value': 'Intel i9'},
            {'key': 'ram', 'value': '32GB'},
            {'key': 'gpu', 'value': 'RTX 4080'},
            {'key': 'storage', 'value': '2TB SSD'},
            {'key': 'screen', 'value': '17.3 inches'},
            {'key': 'refresh_rate', 'value': '240Hz'}
        ]
    }
    
    # Criar índice em attributes para queries eficientes
    def create_attribute_index(self):
        db.products.create_index([
            ('attributes.key', 1),
            ('attributes.value', 1)
        ])
    
    # Query: Filtrar por qualquer atributo
    def find_by_attribute(self, attr_key, attr_value):
        return db.products.find({
            'attributes': {
                '$elemMatch': {
                    'key': attr_key,
                    'value': attr_value
                }
            }
        })
    
    # Exemplo: Encontrar todos produtos com GPU RTX 4080
    rtx_4080_products = find_by_attribute('gpu', 'RTX 4080')
    
    # Benefício: Adicionar novos atributos sem schema change
    # Filtros dinâmicos: UI pode gerar filtros automaticamente
    # Indexação: Eficiente para queries em qualquer atributo

# Caso Real - Amazon Product Catalog:
# Milhões de produtos, cada um com atributos únicos
# Schema fixo seria impossível (100,000+ atributos possíveis)
# Attribute Pattern permite flexibilidade total
```

**3. Extended Reference Pattern (Denormalização Seletiva)**

```python
# Denormalizar campos frequentemente acessados
class ExtendedReferencePattern:
    """
    Otimizar reads com denormalização parcial
    """
    # Order com user info denormalizada
    order_with_extended_ref = {
        '_id': 'order12345',
        'order_date': '2024-01-15',
        'total': 299.99,
        'status': 'shipped',
        
        # Extended reference: Dados básicos do user
        'user': {
            'user_id': 'user789',  # Reference para doc completo
            'name': 'John Doe',    # Denormalizado
            'email': 'john@example.com'  # Denormalizado
            # Não inclui: password, full address, payment methods
        },
        
        'items': [
            {
                'product_id': 'prod456',
                'name': 'Wireless Mouse',  # Denormalizado
                'price': 29.99,  # Denormalizado
                'quantity': 2
            }
        ]
    }
    
    # Query otimizada: 1 read ao invés de 3
    def get_order_summary(self, order_id):
        order = db.orders.find_one({'_id': order_id})
        
        # Já tem todas informações necessárias!
        # Não precisa buscar user ou products separadamente
        return {
            'order_id': order['_id'],
            'date': order['order_date'],
            'customer_name': order['user']['name'],  # Já denormalizado
            'customer_email': order['user']['email'],  # Já denormalizado
            'items': [
                f"{item['name']} x{item['quantity']}"  # Já denormalizado
                for item in order['items']
            ],
            'total': order['total']
        }
        
        # vs Normalized (SQL):
        # SELECT * FROM orders WHERE id = ?
        # SELECT name, email FROM users WHERE id = ?
        # SELECT name, price FROM products WHERE id IN (...)
        # = 3 queries, 3 network round-trips
    
    # Trade-off Management:
    def update_user_name(self, user_id, new_name):
        # 1. Atualizar documento principal do user
        db.users.update_one(
            {'_id': user_id},
            {'$set': {'name': new_name}}
        )
        
        # 2. Atualizar todas as extended references
        # (pode ser async, eventual consistency é OK)
        db.orders.update_many(
            {'user.user_id': user_id},
            {'$set': {'user.name': new_name}}
        )
    
    # Decisão: O que denormalizar?
    # ✓ Denormalizar: Dados lidos frequentemente, mudam raramente
    # ✓ Exemplo: Nome, email (display info)
    # ❌ Não denormalizar: Dados sensíveis, mudam frequentemente
    # ❌ Exemplo: Password, balance, address completo

# Performance Impact:
# - Normalized: 3 queries × 5ms = 15ms
# - Extended Reference: 1 query × 5ms = 5ms
# Speedup: 3x
# Trade-off: Inconsistência temporária durante updates (acceptable)
```

### Schema Validation (Opcional)

Mesmo em NoSQL, pode-se enforcar validação de schema quando necessário.

**MongoDB Schema Validation:**

```javascript
// Criar collection com schema validation
db.createCollection("users", {
    validator: {
        $jsonSchema: {
            bsonType: "object",
            required: ["name", "email", "created_at"],
            properties: {
                name: {
                    bsonType: "string",
                    description: "must be a string and is required"
                },
                email: {
                    bsonType: "string",
                    pattern: "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}$",
                    description: "must be a valid email"
                },
                age: {
                    bsonType: "int",
                    minimum: 0,
                    maximum: 150,
                    description: "must be an integer between 0 and 150"
                },
                created_at: {
                    bsonType: "date",
                    description: "must be a date"
                }
            }
        }
    },
    validationLevel: "strict",  // ou "moderate"
    validationAction: "error"   // ou "warn"
})

// Insert válido:
db.users.insertOne({
    name: "John Doe",
    email: "john@example.com",
    age: 30,
    created_at: new Date()
})  // OK

// Insert inválido:
db.users.insertOne({
    name: "Jane Smith",
    email: "invalid-email",  // Não match pattern
    age: 30
})  // ERROR: Document failed validation

// Benefício: Best of both worlds
// - Flexibilidade de schema evolution
// - Validação quando necessária para dados críticos
```

**AWS DynamoDB Table Schema (via Application):**

```python
# DynamoDB não tem schema validation nativo
# Mas pode implementar na aplicação
class DynamoDBSchemaValidator:
    """
    Application-level schema validation
    """
    user_schema = {
        'user_id': {'type': str, 'required': True},
        'name': {'type': str, 'required': True, 'min_length': 2, 'max_length': 100},
        'email': {'type': str, 'required': True, 'pattern': r'^[\w\.-]+@[\w\.-]+\.\w+$'},
        'age': {'type': int, 'required': False, 'min': 0, 'max': 150},
        'created_at': {'type': str, 'required': True}  # ISO 8601 format
    }
    
    def validate_and_put(self, item):
        # Validate contra schema
        self._validate(item, self.user_schema)
        
        # Se passou, insert
        dynamodb.put_item(
            TableName='Users',
            Item=item
        )
    
    def _validate(self, item, schema):
        for field, rules in schema.items():
            # Check required
            if rules.get('required') and field not in item:
                raise ValidationError(f"Missing required field: {field}")
            
            if field in item:
                value = item[field]
                
                # Check type
                if not isinstance(value, rules['type']):
                    raise ValidationError(f"Field {field} must be {rules['type']}")
                
                # Check string length
                if rules['type'] == str:
                    if 'min_length' in rules and len(value) < rules['min_length']:
                        raise ValidationError(f"{field} too short")
                    if 'max_length' in rules and len(value) > rules['max_length']:
                        raise ValidationError(f"{field} too long")
                    if 'pattern' in rules and not re.match(rules['pattern'], value):
                        raise ValidationError(f"{field} doesn't match pattern")
                
                # Check numeric range
                if rules['type'] in [int, float]:
                    if 'min' in rules and value < rules['min']:
                        raise ValidationError(f"{field} below minimum")
                    if 'max' in rules and value > rules['max']:
                        raise ValidationError(f"{field} above maximum")

# Usage:
validator = DynamoDBSchemaValidator()

# Valid insert:
validator.validate_and_put({
    'user_id': 'user123',
    'name': 'John Doe',
    'email': 'john@example.com',
    'age': 30,
    'created_at': '2024-01-15T10:30:00Z'
})  # OK

# Invalid insert:
validator.validate_and_put({
    'user_id': 'user456',
    'name': 'J',  # Too short!
    'email': 'invalid',  # Invalid format!
    'age': 200  # Above maximum!
})  # Raises ValidationError
```


---

## Data Models (Modelos de Dados)

Bancos de dados NoSQL utilizam diferentes modelos de dados, cada um otimizado para casos de uso específicos.

### 1. Key-Value Stores

**Conceito:**
O modelo mais simples: cada item é armazenado como um par de chave única e valor opaco.

**Características:**
- **Simplicidade**: API minimal (GET, PUT, DELETE)
- **Performance**: O(1) lookups, latência ultra-baixa
- **Escalabilidade**: Particionamento trivial por key
- **Limitação**: Queries apenas por key, sem índices secundários

**AWS DynamoDB - Key-Value Mode:**

```python
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('SessionStore')

class SessionManager:
    """
    Session storage para aplicação web
    """
    def create_session(self, session_id, user_data):
        """
        Criar nova session
        """
        table.put_item(
            Item={
                'session_id': session_id,  # Partition key
                'user_id': user_data['user_id'],
                'username': user_data['username'],
                'login_time': datetime.now().isoformat(),
                'ttl': int((datetime.now() + timedelta(hours=24)).timestamp())
            }
        )
        # Performance: 1-5ms
    
    def get_session(self, session_id):
        """
        Recuperar session
        """
        response = table.get_item(
            Key={'session_id': session_id}
        )
        return response.get('Item')
        # Performance: 1-3ms (single-digit millisecond)
    
    def delete_session(self, session_id):
        """
        Logout: Delete session
        """
        table.delete_item(
            Key={'session_id': session_id}
        )
        # Performance: 1-3ms

# Use Cases Ideais:
# - Session stores
# - Cache layers
# - Shopping carts
# - User preferences
# - Real-time bidding data

# Performance Metrics:
# - Read: <1ms P50, 1-3ms P99
# - Write: 1-3ms P50, 3-10ms P99
# - Throughput: Milhões de requests por segundo
```

**Amazon ElastiCache (Redis) - In-Memory Key-Value:**

```python
import redis

r = redis.Redis(host='my-cluster.cache.amazonaws.com', port=6379)

class CacheLayer:
    """
    Caching layer com Redis
    """
    def cache_product(self, product_id, product_data):
        """
        Cache product details
        """
        key = f"product:{product_id}"
        
        # Store as JSON
        r.setex(
            key,
            3600,  # TTL: 1 hora
            json.dumps(product_data)
        )
        # Performance: 0.1-0.5ms
    
    def get_product_with_cache(self, product_id):
        """
        Get product with cache-aside pattern
        """
        key = f"product:{product_id}"
        
        # Try cache first
        cached = r.get(key)
        if cached:
            return json.loads(cached)
            # Cache hit: 0.1-0.5ms
        
        # Cache miss: Fetch from database
        product = db.get_product(product_id)
        # Database: 10-50ms
        
        # Store in cache for next time
        self.cache_product(product_id, product)
        
        return product
    
    # Advanced Redis Features:
    def increment_page_views(self, page_id):
        """
        Atomic counter
        """
        r.incr(f"pageviews:{page_id}")
        # Performance: 0.1-0.3ms, thread-safe
    
    def add_to_leaderboard(self, user_id, score):
        """
        Sorted set para leaderboards
        """
        r.zadd('game_leaderboard', {user_id: score})
        # Performance: 0.2-0.5ms
    
    def get_top_players(self, count=10):
        """
        Top N players
        """
        return r.zrevrange('game_leaderboard', 0, count-1, withscores=True)
        # Performance: 0.3-1ms

# Performance Comparison:
# Redis (in-memory): 0.1-0.5ms
# DynamoDB (SSD): 1-5ms
# PostgreSQL (disk): 10-50ms
# 
# Redis is 20-500x faster!
# Trade-off: Volatile (data lost on restart, unless persistence enabled)
```

---

## Key-Value Databases (Bancos de Dados Chave-Valor)

Key-Value databases representam o modelo de armazenamento NoSQL mais fundamental e eficiente. Como observa DeCandia et al. no paper seminal "Dynamo: Amazon's Highly Available Key-value Store" (2007): "A chave é usada para determinar o nó de armazenamento, e o valor pode ser um blob opaco sem schema definido". Este design minimalista oferece performance excepcional e escalabilidade massiva.

### Data Model (Modelo de Dados)

#### Conceitos Fundamentais

O modelo de dados de um Key-Value store é conceitualmente simples mas incrivelmente poderoso:

**Estrutura Básica:**
```
Key → Value
```

Cada item no banco é identificado por uma **chave única** que mapeia para um **valor**. A chave é o único meio de acesso aos dados, e o valor é tipicamente opaco para o sistema de armazenamento.

**Características do Modelo:**

```python
class KeyValueDataModel:
    """
    Representação conceitual do modelo Key-Value
    """
    def __init__(self):
        # Estrutura interna: Hash table distribuído
        self.data_store = {}
    
    # API Minimalista
    def put(self, key: str, value: any) -> bool:
        """
        Armazena ou atualiza um valor
        - Key: String única (max 2KB no DynamoDB)
        - Value: Objeto opaco (max 400KB no DynamoDB)
        - Complexidade: O(1)
        """
        self.data_store[key] = value
        return True
    
    def get(self, key: str) -> any:
        """
        Recupera um valor pela chave
        - Complexidade: O(1)
        - Retorna None se key não existe
        """
        return self.data_store.get(key)
    
    def delete(self, key: str) -> bool:
        """
        Remove um item
        - Complexidade: O(1)
        """
        if key in self.data_store:
            del self.data_store[key]
            return True
        return False
    
    def exists(self, key: str) -> bool:
        """
        Verifica existência de uma chave
        - Complexidade: O(1)
        """
        return key in self.data_store

# Propriedades do Modelo:
# 1. Acesso exclusivo por chave (sem queries complexas)
# 2. Valor é opaco (store não interpreta estrutura interna)
# 3. Sem relacionamentos entre items
# 4. Sem índices secundários nativos (algumas implementações adicionam)
# 5. Atomic operations em nível de item
```

#### Tipos de Valores

Key-Value stores suportam diversos tipos de valores:

**1. String Values (Redis)**

```python
import redis

r = redis.Redis()

# String simples
r.set('user:1000:name', 'João Silva')

# Número como string
r.set('user:1000:age', '35')

# JSON serializado
import json
user_data = {'name': 'João', 'email': 'joao@example.com', 'age': 35}
r.set('user:1000', json.dumps(user_data))

# Binary data (imagens, documentos)
with open('profile.jpg', 'rb') as f:
    r.set('user:1000:photo', f.read())

# Benefício: Flexibilidade total no valor
# Trade-off: Aplicação deve serializar/deserializar
```

**2. Structured Values (DynamoDB)**

```python
import boto3
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

# Valor estruturado (map)
table.put_item(
    Item={
        'user_id': 'user1000',  # Key
        # Value: Structured data
        'name': 'João Silva',
        'email': 'joao@example.com',
        'age': 35,
        'balance': Decimal('1500.50'),
        'preferences': {
            'language': 'pt-BR',
            'notifications': True,
            'theme': 'dark'
        },
        'tags': ['premium', 'verified'],
        'created_at': '2024-01-15T10:30:00Z'
    }
)

# DynamoDB suporta tipos:
# - String, Number, Binary
# - Boolean, Null
# - List, Map (nested)
# - String Set, Number Set, Binary Set

# Benefício: Rich data types, sem serialização manual
# Trade-off: Tamanho máximo 400KB por item
```

**3. Data Structures (Redis)**

Redis oferece estruturas de dados nativas além de strings:

```python
import redis

r = redis.Redis()

# LISTS - Sequências ordenadas
r.lpush('user:1000:notifications', 'Nova mensagem')
r.lpush('user:1000:notifications', 'Pagamento aprovado')
recent_notifications = r.lrange('user:1000:notifications', 0, 9)  # Top 10

# SETS - Coleções únicas, não ordenadas
r.sadd('user:1000:interests', 'tecnologia', 'esportes', 'música')
r.sadd('user:2000:interests', 'esportes', 'viagens', 'música')
# Interseção: Interesses comuns
common = r.sinter('user:1000:interests', 'user:2000:interests')
# Result: {'esportes', 'música'}

# SORTED SETS - Sets ordenados por score
r.zadd('game:leaderboard', {
    'user1000': 9500,
    'user2000': 8700,
    'user3000': 9200
})
top_players = r.zrevrange('game:leaderboard', 0, 2, withscores=True)
# Result: [('user1000', 9500), ('user3000', 9200), ('user2000', 8700)]

# HASHES - Mapas de campo-valor
r.hset('user:1000:profile', mapping={
    'name': 'João Silva',
    'email': 'joao@example.com',
    'city': 'São Paulo'
})
email = r.hget('user:1000:profile', 'email')

# BITMAPS - Arrays compactos de bits
# Tracking: Usuário fez login nos últimos 365 dias?
r.setbit('user:1000:login_days', 0, 1)   # Dia 0: Sim
r.setbit('user:1000:login_days', 1, 0)   # Dia 1: Não
r.setbit('user:1000:login_days', 2, 1)   # Dia 2: Sim
active_days = r.bitcount('user:1000:login_days')  # Total de dias ativos

# HYPERLOGLOGS - Contagem aproximada de elementos únicos
# Tracking: Unique visitors
r.pfadd('page:home:visitors:2024-01-15', 'user1000', 'user2000', 'user1000')
unique_count = r.pfcount('page:home:visitors:2024-01-15')  # 2 (deduplicated)

# Benefícios: Operações especializadas, eficiência de memória
# Trade-offs: Mais complexidade, Redis-specific
```

#### Key Design Patterns

Design de chaves é crítico para performance e escalabilidade:

**Padrão 1: Hierarchical Keys**

```python
# Pattern: entity:id:attribute
# Benefícios: Namespace claro, evita colisões

# User data
r.set('user:1000:name', 'João Silva')
r.set('user:1000:email', 'joao@example.com')
r.set('user:1000:last_login', '2024-01-15T10:30:00Z')

# Product data
r.set('product:5678:name', 'Laptop Dell')
r.set('product:5678:price', '2999.99')
r.set('product:5678:stock', '45')

# Order data
r.set('order:9012:user_id', '1000')
r.set('order:9012:total', '2999.99')
r.set('order:9012:status', 'shipped')

# Scan pattern: Listar todas as keys de um namespace
keys = r.keys('user:1000:*')  # Todas as keys do user 1000
# Warning: keys() é O(N), evitar em produção!
# Alternativa: Manter índice separado
```

**Padrão 2: Composite Keys**

```python
# Pattern: entity:attribute:value
# Use case: Índices secundários, lookups reversos

# Index: Email → User ID
r.set('user:email:joao@example.com', '1000')

# Lookup: Encontrar user_id por email
email = 'joao@example.com'
user_id = r.get(f'user:email:{email}')  # '1000'

# Então buscar dados completos
user_data = r.get(f'user:{user_id}')

# Index: Product category
r.sadd('product:category:laptops', '5678', '5679', '5680')
r.sadd('product:category:phones', '1234', '1235', '1236')

# Query: Todos os produtos da categoria laptops
laptop_ids = r.smembers('product:category:laptops')

# Benefício: Simula índices secundários
# Trade-off: Manutenção manual dos índices
```

**Padrão 3: Time-based Keys**

```python
from datetime import datetime

# Pattern: entity:date:id
# Use case: Time-series data, logs, events

# Logs por dia
today = datetime.now().strftime('%Y-%m-%d')
r.lpush(f'logs:errors:{today}', 'Error message 1')
r.lpush(f'logs:errors:{today}', 'Error message 2')

# Metrics por hora
hour = datetime.now().strftime('%Y-%m-%d:%H')
r.incr(f'metrics:page_views:{hour}')
r.incr(f'metrics:api_calls:{hour}')

# TTL automático: Logs expiram após 7 dias
r.expire(f'logs:errors:{today}', 7 * 24 * 60 * 60)  # 7 dias em segundos

# Agregação: Total de page views hoje
hours = [f'metrics:page_views:{today}:{h:02d}' for h in range(24)]
total_views = sum(int(r.get(h) or 0) for h in hours)

# Benefício: Particionamento temporal, cleanup automático
# Trade-off: Queries cross-time complexas
```

**Padrão 4: Versioned Keys**

```python
# Pattern: entity:id:version
# Use case: Imutabilidade, histórico, cache busting

# Versão atual
r.set('config:app:current', '3')
r.set('config:app:v3', json.dumps({
    'api_url': 'https://api.example.com/v3',
    'timeout': 30,
    'retry_count': 3
}))

# Versões antigas mantidas
r.set('config:app:v2', json.dumps({...}))
r.set('config:app:v1', json.dumps({...}))

# Read: Sempre buscar versão atual
current_version = r.get('config:app:current')
config = json.loads(r.get(f'config:app:v{current_version}'))

# Update: Criar nova versão, não sobrescrever
r.set('config:app:v4', json.dumps({...}))
r.set('config:app:current', '4')

# Rollback: Fácil
r.set('config:app:current', '3')

# Cache busting: URLs com versão
image_version = '5'
image_url = f'https://cdn.example.com/product/1000/photo.jpg?v={image_version}'

# Benefícios: Imutabilidade, auditoria, rollback fácil
# Trade-offs: Mais armazenamento, cleanup manual de versões antigas
```

#### Limitações do Modelo

Entender as limitações é essencial para design apropriado:

```python
class KeyValueLimitations:
    """
    Demonstração de limitações e workarounds
    """
    
    # Limitação 1: Sem queries por valor
    def query_by_value_problem(self):
        """
        Problem: Encontrar todos users com age > 30
        """
        # Impossível em Key-Value puro!
        # Não há como query por atributos do valor
        
        # Workaround 1: Scan completo (lento!)
        all_users = []
        for key in r.scan_iter('user:*:age'):  # Scan todas as keys
            age = int(r.get(key))
            if age > 30:
                user_id = key.split(':')[1]
                all_users.append(user_id)
        # O(N) - Inaceitável para grandes datasets
        
        # Workaround 2: Manter índice secundário
        # Durante write:
        def create_user(user_id, age):
            r.set(f'user:{user_id}:age', age)
            # Adicionar ao índice de idade
            r.zadd('users:by_age', {user_id: age})
        
        # Query eficiente:
        users_over_30 = r.zrangebyscore('users:by_age', 30, '+inf')
        # O(log(N) + M) onde M = resultado size
        
        # Trade-off: Manutenção manual de índices
    
    # Limitação 2: Sem relacionamentos
    def relationships_problem(self):
        """
        Problem: User tem múltiplas orders, order tem múltiplos items
        """
        # Impossível fazer JOIN!
        
        # Solução 1: Denormalização
        user_data = {
            'user_id': '1000',
            'name': 'João',
            'recent_orders': [  # Embedded
                {
                    'order_id': '9001',
                    'date': '2024-01-10',
                    'total': 299.99,
                    'items': [  # Nested
                        {'product': 'Mouse', 'price': 29.99},
                        {'product': 'Keyboard', 'price': 270.00}
                    ]
                }
            ]
        }
        r.set('user:1000', json.dumps(user_data))
        
        # Benefício: 1 read para tudo
        # Trade-off: Duplicação, inconsistência potencial
        
        # Solução 2: Application-level JOINs
        def get_user_orders(user_id):
            # Read 1: User
            user = json.loads(r.get(f'user:{user_id}'))
            
            # Read 2: Order IDs
            order_ids = r.smembers(f'user:{user_id}:orders')
            
            # Read 3-N: Cada order
            orders = []
            for order_id in order_ids:
                order = json.loads(r.get(f'order:{order_id}'))
                orders.append(order)
            
            user['orders'] = orders
            return user
        
        # Trade-off: N+2 queries (N+1 problem)
    
    # Limitação 3: Sem transações multi-key (Redis)
    def multi_key_transaction_problem(self):
        """
        Problem: Transferir saldo entre duas contas atomicamente
        """
        # Solução: Redis transactions (limited)
        pipe = r.pipeline()
        
        try:
            # WATCH: Detecta modificações concorrentes
            pipe.watch('account:1000:balance', 'account:2000:balance')
            
            # Verificar saldos
            balance_1 = Decimal(pipe.get('account:1000:balance'))
            balance_2 = Decimal(pipe.get('account:2000:balance'))
            
            amount = Decimal('100.00')
            if balance_1 < amount:
                raise InsufficientFundsError()
            
            # Transaction
            pipe.multi()
            pipe.decrbyfloat('account:1000:balance', float(amount))
            pipe.incrbyfloat('account:2000:balance', float(amount))
            pipe.execute()
            
            return "Transfer successful"
            
        except redis.WatchError:
            # Concurrent modification detected
            # Retry necessário
            return "Conflict, retry"
        
        # Limitações:
        # - Não há rollback automático
        # - Watch/Multi/Exec é optimistic locking
        # - Complexidade alta para desenvolvedores
        
        # Alternativa: Use banco com ACID para casos críticos
    
    # Limitação 4: Sem agregações
    def aggregation_problem(self):
        """
        Problem: Calcular média de preços de produtos
        """
        # Impossível em Key-Value puro
        
        # Workaround: Manter agregações pré-computadas
        def add_product(product_id, price):
            # Armazenar produto
            r.set(f'product:{product_id}:price', price)
            
            # Atualizar agregação
            pipe = r.pipeline()
            pipe.incrbyfloat('products:total_price', price)
            pipe.incr('products:count')
            pipe.execute()
        
        def get_average_price():
            total = Decimal(r.get('products:total_price'))
            count = int(r.get('products:count'))
            return total / count if count > 0 else 0
        
        # Trade-off: Agregações devem ser antecipadas no design

# Real-world Example: Quando NÃO usar Key-Value
use_cases_not_suitable = {
    'Complex Queries': 'Usar SQL ou Document DB',
    'Ad-hoc Analytics': 'Usar Redshift ou BigQuery',
    'Graph Relationships': 'Usar Neptune',
    'Rich Transactions': 'Usar PostgreSQL',
    'Full-text Search': 'Usar Elasticsearch'
}

# When TO use Key-Value:
use_cases_ideal = {
    'Session Storage': 'Simple, fast, TTL support',
    'Caching': 'Ultra-low latency, high throughput',
    'Real-time Counters': 'Atomic increments',
    'Leaderboards': 'Sorted sets',
    'Rate Limiting': 'Counters + TTL',
    'Shopping Carts': 'Temporary data, high availability'
}
```

### Data Access and Retrieval Operations (Operações de Acesso e Recuperação de Dados)

Key-Value databases oferecem operações simples mas extremamente eficientes para acesso a dados.

#### Operações Básicas

**1. PUT/SET - Escrita de Dados**

```python
class KeyValueWriteOperations:
    """
    Operações de escrita em Key-Value stores
    """
    
    # DynamoDB PUT
    def dynamodb_put(self, item):
        """
        PUT: Create or Replace item completamente
        """
        table.put_item(
            Item={
                'user_id': 'user1000',
                'name': 'João Silva',
                'email': 'joao@example.com',
                'age': 35,
                'created_at': '2024-01-15T10:30:00Z'
            }
        )
        # Comportamento: Sobrescreve item existente completamente
        # Performance: 5-15ms
        # Custo: 1 WCU (Write Capacity Unit) para items ≤ 1KB
    
    # DynamoDB UPDATE
    def dynamodb_update(self, user_id):
        """
        UPDATE: Modifica atributos específicos
        """
        table.update_item(
            Key={'user_id': user_id},
            UpdateExpression='SET #name = :name, age = age + :inc',
            ExpressionAttributeNames={'#name': 'name'},  # 'name' é palavra reservada
            ExpressionAttributeValues={
                ':name': 'João Silva Oliveira',
                ':inc': 1
            },
            ReturnValues='ALL_NEW'  # Retorna item após update
        )
        # Benefício: Atomic updates, não precisa read-modify-write
        # Performance: 5-15ms
        # Custo: 1 WCU mesmo modificando só 1 atributo
    
    # Conditional Writes
    def conditional_write(self, user_id):
        """
        Conditional PUT: Write somente se condição satisfeita
        """
        try:
            table.put_item(
                Item={
                    'user_id': user_id,
                    'status': 'active',
                    'version': 2
                },
                ConditionExpression='attribute_not_exists(user_id) OR version < :new_version',
                ExpressionAttributeValues={':new_version': 2}
            )
            return "Write successful"
        except ClientError as e:
            if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
                return "Condition not met - write rejected"
        
        # Use cases:
        # - Prevent overwriting: attribute_not_exists(key)
        # - Optimistic locking: version matching
        # - Business rules: balance >= withdrawal_amount
        
        # Performance: Mesma latência, mas pode falhar
    
    # Redis SET
    def redis_set_operations(self):
        """
        Redis SET com opções avançadas
        """
        # SET básico
        r.set('user:1000:name', 'João Silva')
        
        # SET com TTL (expira automaticamente)
        r.setex('session:abc123', 3600, json.dumps({'user_id': '1000'}))
        # TTL: 1 hora
        
        # SET condicional: Só se key não existe (NX = Not eXists)
        success = r.set('user:1000:email', 'joao@example.com', nx=True)
        # True se criou, False se key já existia
        
        # SET condicional: Só se key existe (XX = eXists)
        success = r.set('user:1000:email', 'new@example.com', xx=True)
        # True se atualizou, False se key não existia
        
        # SET com GET antigo (GETSET pattern)
        old_value = r.getset('counter', '100')
        # Atomicamente: retorna valor antigo + seta novo
        
        # SET múltiplo (atomic)
        r.mset({
            'user:1000:name': 'João',
            'user:1000:email': 'joao@example.com',
            'user:1000:age': '35'
        })
        # Performance: 1 round-trip para múltiplos sets

# Atomic Counters
class AtomicCounters:
    """
    Contadores atômicos - operação crucial em Key-Value stores
    """
    
    # DynamoDB Atomic Counter
    def dynamodb_counter(self):
        """
        Incremento atômico sem race conditions
        """
        response = table.update_item(
            Key={'page_id': 'home'},
            UpdateExpression='ADD view_count :inc',
            ExpressionAttributeValues={':inc': 1},
            ReturnValues='UPDATED_NEW'
        )
        new_count = response['Attributes']['view_count']
        
        # Garantias:
        # - Atomic: Não há race condition
        # - No read before write: Mais eficiente
        # - Safe com concorrência: Múltiplos clients incrementando
        
        # Use cases:
        # - Page views, API calls
        # - Like counts, vote counts
        # - Inventory management
        
        # Performance: 10-15ms
        # Throughput: Limitado pela partition (3000 WCU/s max)
    
    # Redis Atomic Counter
    def redis_counter(self):
        """
        Incremento atômico ultra-rápido
        """
        # INCR: Incrementa por 1
        new_count = r.incr('page:home:views')
        # Performance: 0.1-0.3ms
        
        # INCRBY: Incrementa por N
        new_count = r.incrby('api:calls', 10)
        
        # INCRBYFLOAT: Incrementa decimal
        new_balance = r.incrbyfloat('account:1000:balance', 50.25)
        
        # DECR: Decrementa por 1
        remaining = r.decr('inventory:product123')
        
        # Atomic pattern: Reserve inventory
        remaining = r.decr('inventory:product123')
        if remaining < 0:
            # Oversold! Rollback
            r.incr('inventory:product123')
            return "Out of stock"
        else:
            return f"Reserved! {remaining} remaining"
        
        # Performance: 100,000+ ops/second por node
        # Thread-safe: Sem locks necessários
```

**2. GET - Leitura de Dados**

```python
class KeyValueReadOperations:
    """
    Operações de leitura eficientes
    """
    
    # DynamoDB GET
    def dynamodb_get(self, user_id):
        """
        GET: Recupera item por chave
        """
        # Eventual consistency (default, mais rápido)
        response = table.get_item(
            Key={'user_id': user_id}
        )
        item = response.get('Item')
        # Performance: 2-5ms
        # Custo: 0.5 RCU (Read Capacity Unit) para items ≤ 4KB
        
        # Strong consistency (garantia de ler última versão)
        response = table.get_item(
            Key={'user_id': user_id},
            ConsistentRead=True
        )
        item = response.get('Item')
        # Performance: 5-10ms (2x mais lento)
        # Custo: 1 RCU (2x mais caro)
        
        # Projection: Ler apenas atributos específicos
        response = table.get_item(
            Key={'user_id': user_id},
            ProjectionExpression='#name, email, age',
            ExpressionAttributeNames={'#name': 'name'}
        )
        # Benefício: Menos data transfer, mesma RCU
        
        return item
    
    # Batch GET
    def batch_get_dynamodb(self, user_ids):
        """
        Batch GET: Múltiplos items em uma request
        """
        response = dynamodb_client.batch_get_item(
            RequestItems={
                'Users': {
                    'Keys': [{'user_id': uid} for uid in user_ids],
                    'ProjectionExpression': 'user_id, #name, email',
                    'ExpressionAttributeNames': {'#name': 'name'}
                }
            }
        )
        
        items = response['Responses']['Users']
        
        # Limitações:
        # - Max 100 items por request
        # - Max 16 MB de data total
        # - Items podem vir de múltiplas tables
        
        # Performance: 10-30ms para 100 items
        # vs 100 individual GETs: 200-500ms
        # Speedup: 10-20x
        
        # Unprocessed items: Retry necessário
        unprocessed = response.get('UnprocessedKeys', {})
        if unprocessed:
            # Recursivamente processar unprocessed
            retry_response = dynamodb_client.batch_get_item(
                RequestItems=unprocessed
            )
            items.extend(retry_response['Responses']['Users'])
        
        return items
    
    # Redis GET
    def redis_get(self):
        """
        Redis GET: Ultra-rápido
        """
        # GET simples
        value = r.get('user:1000:name')  # bytes
        name = value.decode('utf-8')  # string
        # Performance: 0.1-0.5ms
        
        # MGET: Múltiplas keys
        values = r.mget([
            'user:1000:name',
            'user:1000:email',
            'user:1000:age'
        ])
        # Performance: 0.2-1ms para 10-100 keys
        # 1 round-trip vs N round-trips
        
        # GET com fallback
        def get_with_fallback(key, fallback_fn):
            value = r.get(key)
            if value is None:
                # Cache miss: Compute e cache
                value = fallback_fn()
                r.setex(key, 3600, value)  # Cache 1 hora
            return value
        
        # EXISTS: Verificar existência sem ler valor
        exists = r.exists('user:1000:name')  # 1 se existe, 0 se não
        # Performance: 0.1ms
        # Use case: Verificação leve antes de operação pesada
        
        # TTL: Verificar tempo restante
        ttl_seconds = r.ttl('session:abc123')
        # -1: Key existe, sem expiry
        # -2: Key não existe
        # N: Segundos até expirar

# Scan Operations (use with caution!)
class ScanOperations:
    """
    Scan: Iterar sobre múltiplos items
    Atenção: Operação cara, evitar em hot paths
    """
    
    # DynamoDB Scan
    def dynamodb_scan(self):
        """
        Scan: Lê toda a tabela sequencialmente
        """
        # Scan básico (evitar!)
        response = table.scan()
        items = response['Items']
        
        # Pagination: Tabelas grandes requerem múltiplas requests
        all_items = []
        last_evaluated_key = None
        
        while True:
            if last_evaluated_key:
                response = table.scan(ExclusiveStartKey=last_evaluated_key)
            else:
                response = table.scan()
            
            all_items.extend(response['Items'])
            
            last_evaluated_key = response.get('LastEvaluatedKey')
            if not last_evaluated_key:
                break  # Scan completo
        
        # Performance: Lento! Lê 1MB por request
        # - Tabela 10GB: ~10,000 requests, minutos
        # - Consome RCUs massivamente
        
        # Scan com filtro (ainda lento)
        response = table.scan(
            FilterExpression='age > :min_age',
            ExpressionAttributeValues={':min_age': 30}
        )
        # Atenção: Filtro aplicado DEPOIS de ler dados
        # Consome RCUs de TODOS os items lidos, não só os filtrados!
        
        # Parallel Scan: Dividir scan entre workers
        def parallel_scan_segment(segment, total_segments):
            response = table.scan(
                Segment=segment,
                TotalSegments=total_segments
            )
            return response['Items']
        
        # Executar 4 workers em paralelo
        from concurrent.futures import ThreadPoolExecutor
        with ThreadPoolExecutor(max_workers=4) as executor:
            futures = [
                executor.submit(parallel_scan_segment, i, 4)
                for i in range(4)
            ]
            all_items = []
            for future in futures:
                all_items.extend(future.result())
        
        # Speedup: 4x mais rápido
        # Trade-off: 4x mais RCU consumption rate
    
    # Redis SCAN
    def redis_scan(self):
        """
        Redis SCAN: Iteração segura sobre keys
        """
        # SCAN: Cursor-based iteration
        cursor = 0
        all_keys = []
        
        while True:
            cursor, keys = r.scan(cursor, match='user:*', count=100)
            all_keys.extend(keys)
            
            if cursor == 0:
                break  # Iteração completa
        
        # Propriedades:
        # - Não bloqueia: Safe em produção
        # - Pode retornar duplicatas: Application deve deduplicate
        # - Não garante snapshot: Keys criados/deletados durante scan
        
        # KEYS: Evitar em produção!
        keys = r.keys('user:*')  # Bloqueia Redis até completar!
        # Use somente em desenvolvimento/debugging
        
        # Alternativa: Manter índice
        # Durante write, adicionar key ao set
        r.set('user:1000:name', 'João')
        r.sadd('all_user_keys', 'user:1000:name')  # Índice
        
        # Listar todas as keys instantaneamente
        all_keys = r.smembers('all_user_keys')
        # Performance: O(N) onde N = número de keys, mas sem bloqueio

# Advanced: Query Patterns
class QueryPatterns:
    """
    Padrões de query apesar das limitações
    """
    
    def begins_with_query(self):
        """
        Query: Todas as sessions de um usuário
        """
        # DynamoDB: Sort key com begins_with
        response = table.query(
            KeyConditionExpression='user_id = :uid AND begins_with(session_id, :prefix)',
            ExpressionAttributeValues={
                ':uid': 'user1000',
                ':prefix': '2024-01'  # Sessions de Janeiro 2024
            }
        )
        # Performance: 10-50ms, escalável
        # Requer: Sort key com formato hierárquico
        
        # Redis: SCAN com pattern matching
        cursor = 0
        matching_keys = []
        while True:
            cursor, keys = r.scan(
                cursor,
                match='session:user1000:2024-01*',
                count=100
            )
            matching_keys.extend(keys)
            if cursor == 0:
                break
    
    def range_query(self):
        """
        Query: Range de valores
        """
        # DynamoDB: Sort key range
        response = table.query(
            KeyConditionExpression='user_id = :uid AND #timestamp BETWEEN :start AND :end',
            ExpressionAttributeNames={'#timestamp': 'timestamp'},
            ExpressionAttributeValues={
                ':uid': 'user1000',
                ':start': '2024-01-01T00:00:00Z',
                ':end': '2024-01-31T23:59:59Z'
            }
        )
        # Eficiente: Usa índice ordenado
        
        # Redis: Sorted sets
        # Store data com timestamp como score
        r.zadd('user:1000:events', {
            'event1': 1704067200,  # Unix timestamp
            'event2': 1704153600,
            'event3': 1704240000
        })
        
        # Query: Events em range de tempo
        events = r.zrangebyscore(
            'user:1000:events',
            1704067200,  # Start timestamp
            1704240000   # End timestamp
        )
        # Performance: O(log(N) + M), muito eficiente
    
    def secondary_index_pattern(self):
        """
        Simular secondary index
        """
        # Problema: Buscar user por email (not primary key)
        
        # Solução 1: Maintain reverse index
        def create_user(user_id, email):
            # Write user data
            table.put_item(Item={
                'user_id': user_id,
                'email': email,
                'name': 'João'
            })
            
            # Write reverse index
            table.put_item(Item={
                'email': email,  # Key na index table
                'user_id': user_id
            })
        
        def find_user_by_email(email):
            # Lookup index
            response = index_table.get_item(Key={'email': email})
            user_id = response['Item']['user_id']
            
            # Fetch user data
            response = table.get_item(Key={'user_id': user_id})
            return response['Item']
        
        # Trade-off: 2 queries, manutenção de índice
        
        # Solução 2: DynamoDB Global Secondary Index (GSI)
        # Definido na criação da table
        # Automaticamente mantido pelo DynamoDB
        response = table.query(
            IndexName='EmailIndex',
            KeyConditionExpression='email = :email',
            ExpressionAttributeValues={':email': 'joao@example.com'}
        )
        # Benefício: Single query, sem manutenção manual
        # Trade-off: Eventual consistency, custo extra
```

**3. DELETE - Remoção de Dados**

```python
class KeyValueDeleteOperations:
    """
    Operações de deleção
    """
    
    # DynamoDB DELETE
    def dynamodb_delete(self, user_id):
        """
        DELETE: Remove item
        """
        # Delete simples
        table.delete_item(Key={'user_id': user_id})
        # Performance: 5-15ms
        # Custo: 1 WCU
        
        # Delete condicional
        try:
            table.delete_item(
                Key={'user_id': user_id},
                ConditionExpression='#status = :inactive',
                ExpressionAttributeNames={'#status': 'status'},
                ExpressionAttributeValues={':inactive': 'inactive'}
            )
            return "Deleted successfully"
        except ClientError as e:
            if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
                return "Condition not met - item not deleted"
        
        # Delete com retorno do item deletado
        response = table.delete_item(
            Key={'user_id': user_id},
            ReturnValues='ALL_OLD'
        )
        deleted_item = response.get('Attributes')
        # Use case: Audit log, undo operations
    
    # Batch DELETE
    def batch_delete_dynamodb(self, user_ids):
        """
        Batch delete: Múltiplos items
        """
        with table.batch_writer() as batch:
            for user_id in user_ids:
                batch.delete_item(Key={'user_id': user_id})
        
        # Limitações:
        # - Max 25 items por batch
        # - Não suporta conditional deletes
        # - Partial failures possíveis (retry necessário)
        
        # Performance: 20-50ms para 25 items
        # vs 25 individual deletes: 125-375ms
    
    # Redis DELETE
    def redis_delete(self):
        """
        Redis DELETE: Ultra-rápido
        """
        # DELETE single key
        deleted = r.delete('user:1000:name')
        # Returns: Número de keys deletadas (1 ou 0)
        # Performance: 0.1ms
        
        # DELETE múltiplas keys
        deleted_count = r.delete(
            'user:1000:name',
            'user:1000:email',
            'user:1000:age'
        )
        # Returns: Total de keys deletadas
        
        # UNLINK: Async delete (não bloqueia)
        r.unlink('large_key_with_millions_of_items')
        # Benefício: Delete de estruturas grandes não bloqueia
        # Background thread remove data
        
        # Delete por pattern (perigoso!)
        keys_to_delete = []
        cursor = 0
        while True:
            cursor, keys = r.scan(cursor, match='temp:*', count=100)
            keys_to_delete.extend(keys)
            if cursor == 0:
                break
        
        if keys_to_delete:
            r.delete(*keys_to_delete)
        # Atenção: Use com cuidado, pode deletar muitos dados!
    
    # TTL-based expiration (auto-delete)
    def ttl_expiration(self):
        """
        Expiração automática via TTL
        """
        # DynamoDB TTL (gratuito!)
        # Configurar na table: TTL attribute = 'expires_at'
        
        # Write com TTL
        import time
        ttl_timestamp = int(time.time()) + 86400  # 24 horas
        
        table.put_item(
            Item={
                'session_id': 'abc123',
                'user_id': 'user1000',
                'expires_at': ttl_timestamp  # Unix timestamp
            }
        )
        # DynamoDB deleta automaticamente após expires_at
        # Delay: Até 48 horas após expiration (eventual deletion)
        
        # Redis TTL (preciso)
        # SET com expiry
        r.setex('session:abc123', 3600, 'data')  # Expira em 1 hora
        
        # Ou EXPIRE em key existente
        r.set('session:xyz789', 'data')
        r.expire('session:xyz789', 3600)  # 1 hora
        
        # Check TTL restante
        ttl = r.ttl('session:abc123')
        # -1: Sem expiry
        # -2: Key não existe (já expirou)
        # N: Segundos restantes
        
        # Remove TTL
        r.persist('session:abc123')  # Torna key permanente
        
        # Benefícios:
        # - Cleanup automático (no manual delete)
        # - Ideal para dados temporários
        # - Cache invalidation automática
```

#### Performance Optimization Patterns

```python
class PerformanceOptimizations:
    """
    Padrões de otimização para Key-Value operations
    """
    
    # 1. Connection Pooling
    def connection_pooling(self):
        """
        Reutilizar conexões para reduzir overhead
        """
        # Redis connection pool
        pool = redis.ConnectionPool(
            host='redis.amazonaws.com',
            port=6379,
            max_connections=50,  # Pool size
            decode_responses=True
        )
        r = redis.Redis(connection_pool=pool)
        
        # Benefício: Evita overhead de TCP handshake
        # New connection: 5-20ms overhead
        # Pooled connection: <1ms overhead
        
        # DynamoDB: Cliente já usa connection pooling internamente
        dynamodb_client = boto3.client('dynamodb')
        # Reutilize cliente, não crie novo a cada request
    
    # 2. Pipeline/Batching
    def pipelining(self):
        """
        Agrupar múltiplas operações em uma round-trip
        """
        # Redis pipeline
        pipe = r.pipeline()
        pipe.set('key1', 'value1')
        pipe.set('key2', 'value2')
        pipe.incr('counter')
        pipe.get('key1')
        results = pipe.execute()
        # 1 round-trip vs 4 round-trips
        # Speedup: 3-10x dependendo de network latency
        
        # DynamoDB batch operations
        with table.batch_writer() as batch:
            for i in range(100):
                batch.put_item(Item={'id': f'item{i}', 'data': '...'})
        # Automatically batches em requests de 25 items
        # Speedup: 10-25x vs individual writes
    
    # 3. Caching Strategies
    def caching_strategies(self):
        """
        Padrões de cache para otimizar reads
        """
        # Cache-Aside (Lazy Loading)
        def get_user_cache_aside(user_id):
            # Try cache first
            cached = r.get(f'user:{user_id}')
            if cached:
                return json.loads(cached)  # Cache hit
            
            # Cache miss: Load from DB
            user = table.get_item(Key={'user_id': user_id})['Item']
            
            # Store in cache
            r.setex(f'user:{user_id}', 3600, json.dumps(user))
            
            return user
        
        # Write-Through
        def update_user_write_through(user_id, updates):
            # Write to DB first
            table.update_item(
                Key={'user_id': user_id},
                UpdateExpression='SET #name = :name',
                ExpressionAttributeNames={'#name': 'name'},
                ExpressionAttributeValues={':name': updates['name']}
            )
            
            # Update cache
            user = table.get_item(Key={'user_id': user_id})['Item']
            r.setex(f'user:{user_id}', 3600, json.dumps(user))
        
        # Write-Behind (Async)
        def update_user_write_behind(user_id, updates):
            # Update cache immediately
            user = json.loads(r.get(f'user:{user_id}'))
            user.update(updates)
            r.setex(f'user:{user_id}', 3600, json.dumps(user))
            
            # Queue DB write (async)
            queue.put({
                'operation': 'update_user',
                'user_id': user_id,
                'updates': updates
            })
            # Background worker processa queue
            
            # Benefício: Latência mínima (1-2ms)
            # Trade-off: Eventual consistency, risk of data loss
    
    # 4. Read Replicas (Redis)
    def read_replicas(self):
        """
        Distribuir reads entre replicas
        """
        # Redis Cluster com replicas
        from rediscluster import RedisCluster
        
        startup_nodes = [
            {"host": "redis-master.aws.com", "port": 6379},
            {"host": "redis-replica1.aws.com", "port": 6379},
            {"host": "redis-replica2.aws.com", "port": 6379}
        ]
        
        rc = RedisCluster(
            startup_nodes=startup_nodes,
            decode_responses=True,
            readonly_mode=True  # Permite reads de replicas
        )
        
        # Reads: Distribuídos entre master e replicas
        value = rc.get('key')  # Pode ler de qualquer node
        
        # Writes: Sempre vão para master
        rc.set('key', 'value')  # Vai para master
        
        # Benefício: 3x read throughput
        # Trade-off: Eventual consistency (replication lag ~1-10ms)
    
    # 5. Compression
    def compression(self):
        """
        Comprimir valores grandes para economizar storage e bandwidth
        """
        import gzip
        import json
        
        def set_compressed(key, data):
            # Serialize e compress
            json_data = json.dumps(data).encode('utf-8')
            compressed = gzip.compress(json_data)
            
            r.set(key, compressed)
            # Savings: 50-90% dependendo dos dados
        
        def get_compressed(key):
            # Retrieve e decompress
            compressed = r.get(key)
            if compressed:
                json_data = gzip.decompress(compressed)
                return json.loads(json_data)
            return None
        
        # Trade-off:
        # - Storage: 50-90% menor
        # - CPU: +1-5ms para compress/decompress
        # - Use para valores >10KB

# Performance Metrics Cheatsheet
performance_cheatsheet = {
    'DynamoDB': {
        'GET (eventual)': '2-5ms',
        'GET (strong)': '5-10ms',
        'PUT/UPDATE': '5-15ms',
        'DELETE': '5-15ms',
        'Batch (25 items)': '20-50ms',
        'Query': '10-50ms',
        'Scan (1MB)': '100-500ms'
    },
    'Redis': {
        'GET': '0.1-0.5ms',
        'SET': '0.1-0.5ms',
        'INCR': '0.1-0.3ms',
        'DELETE': '0.1ms',
        'MGET (10 keys)': '0.2-1ms',
        'ZADD/ZRANGE': '0.2-1ms',
        'Pipeline (10 ops)': '0.5-2ms'
    },
    'Network Overhead': {
        'Same AZ': '0.5-1ms',
        'Cross AZ': '1-3ms',
        'Cross Region': '50-150ms',
        'TCP Handshake': '5-20ms',
        'TLS Handshake': '10-30ms'
    }
}
```

### Scaling Key-Value Stores (Escalabilidade em Armazenamentos Chave-Valor)

A escalabilidade é uma das principais vantagens dos Key-Value stores. Eles foram projetados desde o início para escalar horizontalmente de forma eficiente.

#### Horizontal Partitioning (Sharding)

**Conceito Fundamental:**

Dados são distribuídos entre múltiplos nodes baseado em uma função de hash da chave. Este processo, chamado de particionamento ou sharding, é transparente para a aplicação.

```python
class HorizontalPartitioning:
    """
    Como Key-Value stores distribuem dados entre nodes
    """
    
    def __init__(self, nodes):
        self.nodes = nodes  # Lista de nodes no cluster
        self.num_nodes = len(nodes)
    
    # Particionamento Simples: Modulo Hashing
    def simple_hash_partition(self, key):
        """
        Particionamento básico: hash(key) % num_nodes
        """
        hash_value = hash(key)
        node_index = hash_value % self.num_nodes
        target_node = self.nodes[node_index]
        
        return target_node
    
    # Problema com simple hashing:
    def problem_with_modulo(self):
        """
        Adicionar/remover node requer redistribuir TODOS os dados
        """
        # Cenário: 3 nodes inicialmente
        # key='user1000' → hash=12345 → 12345 % 3 = 0 → node0
        
        # Adicionar 4º node
        # key='user1000' → hash=12345 → 12345 % 4 = 1 → node1 ❌
        # TODOS as keys mudam de node!
        
        # Problema: Rebalanceamento massivo (horas/dias para TB de dados)
        pass
    
    # Solução: Consistent Hashing
    def consistent_hashing(self, key):
        """
        Consistent hashing: Apenas 1/N dos dados movem ao adicionar node
        """
        from hashlib import md5
        
        # Hash ring: Círculo de 0 a 2^128
        # Nodes e keys são mapeados no ring
        
        # 1. Hash da key
        key_hash = int(md5(key.encode()).hexdigest(), 16)
        
        # 2. Encontrar primeiro node >= key_hash no ring
        # (clockwise no ring)
        target_node = self._find_node_on_ring(key_hash)
        
        return target_node
    
    def _find_node_on_ring(self, key_hash):
        """
        Encontra node responsável pela key no hash ring
        """
        # Sorted list de node positions no ring
        # Binário search para eficiência: O(log N)
        
        # Pseudo-code:
        # if key_hash > all node positions:
        #     return first_node  # Wrap around
        # else:
        #     return next_node_clockwise
        pass
    
    # Benefícios do Consistent Hashing:
    benefits = {
        'Add Node': 'Apenas 1/N dos dados redistribuídos',
        'Remove Node': 'Apenas dados daquele node redistribuídos',
        'Minimal Disruption': 'Maioria das keys não movem',
        'Load Distribution': 'Virtual nodes para balanceamento uniforme'
    }

# Real Implementation: Amazon DynamoDB
class DynamoDBPartitioning:
    """
    Como DynamoDB particiona dados
    """
    
    def partition_key_hashing(self):
        """
        DynamoDB usa MD5 hash da partition key
        """
        # Table: Users
        # Partition Key: user_id
        
        # Write: user_id='user1000'
        # 1. MD5 hash: md5('user1000') → 128-bit hash
        # 2. Determina partition baseado em hash
        # 3. Escreve em partition específica
        
        # Propriedades:
        # - Uniform distribution (se keys bem distribuídas)
        # - Deterministic (mesma key → mesma partition sempre)
        # - Transparente (application não sabe qual partition)
        
        # Hot Partition Problem:
        hot_partition_example = """
        Problem: 80% dos requests vão para 1 partition
        
        Causa: Partition key mal escolhida
        - Exemplo: status='active' (maioria dos users)
        - Exemplo: date='2024-01-15' (todos writes hoje)
        
        Consequência:
        - 1 partition: 3000 WCU/s máximo
        - Throttling em outras partitions idle
        
        Solução: Adicionar suffix randômico
        - partition_key = 'status_active_' + random(1, 10)
        - Distribui load entre 10 partitions
        - Trade-off: Queries mais complexas
        """
        
        return hot_partition_example
    
    def adaptive_capacity(self):
        """
        DynamoDB Adaptive Capacity: Mitigação automática de hot partitions
        """
        # Cenário: Partition A está hot (3000 WCU/s)
        #          Partition B está fria (100 WCU/s)
        
        # DynamoDB automaticamente:
        # 1. Detecta hot partition
        # 2. "Empresta" capacidade de partitions frias
        # 3. Hot partition pode usar até 300 segundos de burst
        
        # Benefício: Tolera spikes temporários
        # Limitação: Não resolve hot partitions permanentes
        
        # Best practice: Design partition key para distribuição uniforme
        good_partition_keys = [
            'user_id',  # Muitos users, acesso distribuído
            'device_id',  # Muitos devices, acesso distribuído
            'order_id',  # IDs únicos, acesso distribuído
        ]
        
        bad_partition_keys = [
            'status',  # Poucos valores, acesso concentrado
            'country',  # Poucos valores (se users concentrados)
            'date',  # Todos writes vão para partition de hoje
        ]

# Virtual Nodes (Cassandra)
class VirtualNodes:
    """
    Cassandra usa virtual nodes para balanceamento fino
    """
    
    def vnodes_concept(self):
        """
        Cada physical node possui múltiplos virtual nodes (vnodes)
        """
        # Tradicional: 1 physical node = 1 posição no hash ring
        # Problem: Desbalanceamento se nodes têm capacidades diferentes
        
        # VNodes: 1 physical node = 256 posições no hash ring (default)
        # Benefício: Balanceamento muito mais fino
        
        # Exemplo:
        # Cluster: 3 physical nodes (A, B, C)
        # Cada node: 256 vnodes
        # Total: 768 vnodes no ring
        
        # Data distribution:
        # - Aproximadamente 256 vnodes por node
        # - Pequenas variações (±5%) são normais
        # - Balanceamento automático
        
        # Add node D:
        # - D recebe ~256 vnodes
        # - Distribuídos uniformemente no ring
        # - Cada existing node doa ~85 vnodes para D
        # - Apenas 1/4 dos dados movem (vs 1/3 no exemplo acima)
        
        # Benefício: Rebalanceamento muito mais rápido
        # Node grande: Pode ter mais vnodes (ex: 512 vs 256)
    
    def token_allocation(self):
        """
        Como tokens (positions no ring) são alocados
        """
        import random
        
        # Random token allocation (Cassandra 3.0+)
        def allocate_tokens(num_vnodes=256):
            max_token = 2**63  # 64-bit token space
            tokens = []
            for _ in range(num_vnodes):
                token = random.randint(-max_token, max_token-1)
                tokens.append(token)
            return sorted(tokens)
        
        # Physical node A: tokens [123, 5678, 91011, ...]
        # Physical node B: tokens [456, 7890, 10111, ...]
        # Physical node C: tokens [789, 8901, 11121, ...]
        
        # Key placement:
        # key='user1000' → hash → token=6500
        # Find next token clockwise: 7890 (node B)
        # Write to node B (and replicas)

#### Replication for Scalability

```python
class ReplicationForScalability:
    """
    Replicação não é apenas para availability, mas também para scalability
    """
    
    # Read Scalability via Replicas
    def read_replicas(self):
        """
        Distribuir reads entre múltiplas replicas
        """
        # Cassandra: RF=3 (3 replicas por partition)
        # - Replica A (primary)
        # - Replica B (secondary)
        # - Replica C (secondary)
        
        # Read com CL=ONE:
        # - Coordinator escolhe replica mais próxima
        # - Distribui reads entre A, B, C
        # - Read throughput: 3x vs single replica
        
        # Trade-off: Eventual consistency
        # - Replica pode estar ligeiramente desatualizada
        # - Acceptable para maioria dos use cases
        
        # Read throughput scaling:
        # RF=1: 1x read capacity
        # RF=3: 3x read capacity (com CL=ONE)
        # RF=5: 5x read capacity (com CL=ONE)
    
    # DynamoDB Global Tables
    def global_tables_scalability(self):
        """
        Replicação multi-region para scalability global
        """
        # Setup: Global table em 3 regiões
        # - us-east-1
        # - eu-west-1
        # - ap-southeast-1
        
        # Writes: Cada região aceita writes localmente
        # - US users: Write para us-east-1 (10ms)
        # - EU users: Write para eu-west-1 (10ms)
        # - APAC users: Write para ap-southeast-1 (10ms)
        # vs single region: 100-300ms cross-region latency
        
        # Reads: Cada região responde localmente
        # - Global read capacity = Sum of all regions
        # - 10K RCU/s × 3 regions = 30K RCU/s global
        
        # Benefícios:
        # - Low latency worldwide
        # - Linear scalability: Add region = Add capacity
        # - Disaster recovery automático
        
        # Trade-offs:
        # - 3x cost (3 replicas)
        # - Eventual consistency cross-region
        # - Conflict resolution necessário

#### Auto-Scaling

```python
class AutoScaling:
    """
    Escalabilidade automática baseada em demanda
    """
    
    # DynamoDB Auto Scaling
    def dynamodb_auto_scaling(self):
        """
        DynamoDB ajusta capacidade automaticamente
        """
        # Configuration:
        auto_scaling_config = {
            'target_utilization': 70,  # Target 70% utilization
            'min_capacity': 5,  # Minimum RCU/WCU
            'max_capacity': 1000,  # Maximum RCU/WCU
            'scale_up_policy': {
                'cooldown': 60,  # 60 segundos entre scale-ups
                'increment': '20%'  # Aumenta 20% por vez
            },
            'scale_down_policy': {
                'cooldown': 300,  # 5 minutos entre scale-downs
                'decrement': '10%'  # Reduz 10% por vez
            }
        }
        
        # Comportamento:
        # - Load aumenta: Auto-scaling aumenta capacity
        # - Load diminui: Auto-scaling reduz capacity (mais lento)
        # - Proteção contra thrashing: Cooldown periods
        
        # Performance:
        # - Scale-up: 1-3 minutos para aplicar
        # - Scale-down: 5-15 minutos para aplicar
        
        # Limitações:
        # - Não previne throttling durante spikes súbitos
        # - Melhor para workloads graduais
        
        # Alternativa: On-Demand pricing
        # - Sem planejamento de capacity
        # - Paga por request (mais caro por request)
        # - Ideal para workloads imprevisíveis
    
    # Redis Cluster Scaling
    def redis_cluster_scaling(self):
        """
        Redis Cluster: Escalabilidade horizontal
        """
        # Initial setup: 3 master nodes
        # - Master 0: Hash slots 0-5461
        # - Master 1: Hash slots 5462-10922
        # - Master 2: Hash slots 10923-16383
        
        # Add 4th master node:
        # 1. Add new node to cluster
        # 2. Rebalance hash slots
        #    - Master 0: 0-4095 (cede slots 4096-5461)
        #    - Master 1: 4096-8191 (cede slots 8192-10922)
        #    - Master 2: 8192-12287 (cede slots 12288-16383)
        #    - Master 3: 12288-16383 (recebe slots redistributed)
        # 3. Migrate data para novo node
        # 4. Cluster rebalanced
        
        # Downtime: Zero (rolling migration)
        # Time: 10-60 minutos dependendo de data size
        
        # Scalability limit:
        # - Max 1000 nodes por cluster
        # - 16,384 hash slots total
        # - Cada node: ~16 slots (com 1000 nodes)
        
        # Write throughput scaling:
        # 3 masters: 300K ops/s
        # 6 masters: 600K ops/s
        # 10 masters: 1M ops/s
        # Linear scalability!

#### Performance Under Scale

```python
class PerformanceAtScale:
    """
    Como performance muda com escala
    """
    
    def latency_characteristics(self):
        """
        Latência em diferentes escalas
        """
        performance_by_scale = {
            'Single Node Redis': {
                '10K ops/s': '0.1-0.3ms P99',
                '50K ops/s': '0.2-0.5ms P99',
                '100K ops/s': '0.5-2ms P99',  # CPU bound
                '200K ops/s': 'Throttling/Errors'
            },
            
            'DynamoDB Small Table (<10GB)': {
                '100 RCU/s': '3-8ms P99',
                '1000 RCU/s': '5-15ms P99',
                '10000 RCU/s': '8-25ms P99'
            },
            
            'DynamoDB Large Table (>1TB)': {
                '100K RCU/s': '10-30ms P99',
                '500K RCU/s': '15-50ms P99',
                '1M RCU/s': '25-100ms P99',
                # Multiple partitions: Mais variabilidade
            },
            
            'Cassandra Cluster (10 nodes)': {
                '10K ops/s': '5-15ms P99',
                '100K ops/s': '10-30ms P99',
                '500K ops/s': '20-60ms P99',
                '1M ops/s': '30-100ms P99'
            }
        }
        
        # Observações:
        # - Latência aumenta com throughput (queuing)
        # - Latência aumenta com data size (mais partitions)
        # - P99 latência >> P50 latência (tail latency)
        # - Variabilidade aumenta com escala
        
        return performance_by_scale
    
    def scaling_best_practices(self):
        """
        Best practices para scaling Key-Value stores
        """
        best_practices = {
            'Partition Key Design': [
                'Distribuição uniforme: Evitar hot keys',
                'High cardinality: Muitos valores únicos',
                'Predictable access: User ID melhor que status',
                'Consider composite keys: user_id + timestamp'
            ],
            
            'Data Modeling': [
                'Denormalize para evitar joins',
                'Pre-compute aggregations',
                'Use secondary indexes com cuidado (custo)',
                'TTL para auto-cleanup de dados temporários'
            ],
            
            'Read Optimization': [
                'Cache frequently accessed data (Redis)',
                'Use eventual consistency quando possível',
                'Batch reads com batch_get_item',
                'Projection para ler apenas campos necessários'
            ],
            
            'Write Optimization': [
                'Batch writes com batch_write_item',
                'Async writes quando possível',
                'Compress large items (>10KB)',
                'Avoid hot partitions: Shard busy keys'
            ],
            
            'Monitoring': [
                'Track throttled requests',
                'Monitor partition key distribution',
                'Alert on high latency (P99)',
                'Capacity utilization alarms'
            ],
            
            'Capacity Planning': [
                'Size for peak load + 20% buffer',
                'Use auto-scaling para gradual changes',
                'Pre-warm partitions antes de eventos grandes',
                'Consider on-demand para spiky workloads'
            ]
        }
        
        return best_practices

# Real-world Scaling Example
class ScalingExample:
    """
    Exemplo: Escalar sessão store de 10K para 1M users
    """
    
    def initial_setup(self):
        """
        Setup inicial: 10K concurrent users
        """
        setup = {
            'service': 'DynamoDB',
            'table': 'UserSessions',
            'partition_key': 'session_id',
            'item_size': '2KB',
            
            'workload': {
                'reads': '100/s',  # Cada user: 1 read a cada 100s
                'writes': '10/s',  # Session updates
                'sessions_per_user': 1,
                'concurrent_users': 10000
            },
            
            'capacity': {
                'RCU': 50,  # 50 reads/s × 2KB/4KB = 25 RCU
                'WCU': 20,  # 10 writes/s × 2KB/1KB = 20 WCU
                'cost': '$12/month'  # Provisioned capacity
            },
            
            'performance': {
                'p50_latency': '3-5ms',
                'p99_latency': '10-20ms',
                'throttling': '0%'
            }
        }
        return setup
    
    def scaled_setup(self):
        """
        Scaled up: 1M concurrent users (100x)
        """
        setup = {
            'service': 'DynamoDB',
            'table': 'UserSessions',
            'partition_key': 'session_id',  # Mesma strategy, escala bem
            
            'workload': {
                'reads': '10000/s',  # 100x
                'writes': '1000/s',  # 100x
                'concurrent_users': 1000000
            },
            
            'capacity': {
                'RCU': 5000,  # 100x (10K reads/s × 2KB/4KB ×  0.5)
                'WCU': 2000,  # 100x (1K writes/s × 2KB/1KB × 2)
                'cost': '$1,200/month',  # Provisioned capacity
                # Alternativa: On-demand = $1,800/month
            },
            
            'architecture_changes': [
                'Added Redis cache layer',
                '80% cache hit rate',
                'DynamoDB only for cache misses + writes',
                'Actual DynamoDB load: 2K RCU, 2K WCU',
                'Cost optimizado: $500/month DynamoDB + $100 Redis'
            ],
            
            'performance': {
                'p50_latency': '5-10ms',  # Slightly higher
                'p99_latency': '20-50ms',  # Tail latency degradation
                'throttling': '<0.1%'  # Minimal com adaptive capacity
            }
        }
        return setup
```

### Availability in Key-Value Stores (Disponibilidade em Armazenamentos Chave-Valor)

Disponibilidade é um pilar fundamental dos Key-Value stores, especialmente em sistemas críticos que não podem tolerar downtime.

#### Replication Strategies

```python
class ReplicationStrategies:
    """
    Estratégias de replicação para alta disponibilidade
    """
    
    # Synchronous Replication (Strong Consistency)
    def synchronous_replication(self):
        """
        Write só confirma após replicar para múltiplos nodes
        """
        # DynamoDB: Cross-AZ synchronous replication
        def write_with_sync_replication(key, value):
            # 1. Write enviado para partition leader
            leader_node.write(key, value)
            
            # 2. Leader replica para 2 outros AZs (synchronously)
            responses = []
            responses.append(az1_replica.write(key, value))
            responses.append(az2_replica.write(key, value))
            
            # 3. Aguarda QUORUM (2 de 3) antes de confirmar
            successful = sum(1 for r in responses if r.success)
            
            if successful >= 2:
                return "Write committed"  # Durável em ≥2 AZs
            else:
                return "Write failed"  # Rollback
            
            # Performance: 10-20ms (inclui replication latency)
            # Availability: Pode escrever com 1 AZ down
            # Durability: 11 noves (99.999999999%)
        
        # Propriedades:
        # - Strong consistency: Reads sempre veem últimas writes
        # - Durável: Dados replicados antes de ACK
        # - Lento: Latência de replicação no critical path
        # - Less available: Requer quorum para writes
    
    # Asynchronous Replication (Eventual Consistency)
    def asynchronous_replication(self):
        """
        Write confirma imediatamente, replica em background
        """
        # Cassandra (with CL=ONE)
        def write_with_async_replication(key, value):
            # 1. Write para node local
            local_node.write(key, value)
            
            # 2. Retorna sucesso IMEDIATAMENTE
            # (sem esperar replicas)
            
            # 3. Background: Async replication para replicas
            async def replicate_background():
                replica1.write(key, value)
                replica2.write(key, value)
            
            asyncio.create_task(replicate_background())
            
            return "Write accepted"  # Rápido!
            
            # Performance: 1-5ms (sem esperar replicas)
            # Availability: Sempre disponível (mesmo com failures)
            # Consistency: Eventual (replication lag ~10-100ms)
            # Risk: Data loss se node crashes antes de replicar
        
        # Propriedades:
        # - Fast writes: Latência mínima
        # - High availability: Aceita writes com failures
        # - Eventual consistency: Reads podem ver dados antigos
        # - Durability risk: Potencial data loss

#### Failure Detection and Handling

```python
class FailureDetection:
    """
    Como Key-Value stores detectam e tratam failures
    """
    
    # Heartbeat Mechanism
    def heartbeat_detection(self):
        """
        Nodes enviam heartbeats periódicos
        """
        class Node:
            def __init__(self, node_id):
                self.node_id = node_id
                self.last_heartbeat = time.time()
            
            def send_heartbeat(self):
                """Node envia heartbeat a cada segundo"""
                self.last_heartbeat = time.time()
                broadcast_to_cluster({
                    'node_id': self.node_id,
                    'status': 'alive',
                    'timestamp': self.last_heartbeat
                })
            
            def is_alive(self, timeout=10):
                """Considera node morto se sem heartbeat por 10s"""
                return (time.time() - self.last_heartbeat) < timeout
        
        # Cassandra Gossip Protocol:
        # - Cada node fala com 1-3 outros nodes a cada segundo
        # - Informação propaga exponencialmente
        # - Toda a cluster sabe de um failure em ~log(N) heartbeats
        # - Com 100 nodes: ~7 segundos para full propagation
    
    # Failure Handling: Hinted Handoff
    def hinted_handoff(self):
        """
        Cassandra: Armazenar writes para nodes down temporariamente
        """
        def write_with_hinted_handoff(key, value, replicas):
            hints = []
            successful_writes = 0
            
            for replica in replicas:
                if replica.is_alive():
                    try:
                        replica.write(key, value)
                        successful_writes += 1
                    except Exception:
                        # Replica down, store hint
                        hints.append({
                            'target_replica': replica,
                            'key': key,
                            'value': value,
                            'timestamp': time.time()
                        })
                else:
                    # Replica known to be down, store hint
                    hints.append({
                        'target_replica': replica,
                        'key': key,
                        'value': value,
                        'timestamp': time.time()
                    })
            
            # Store hints para delivery posterior
            for hint in hints:
                hint_storage.store(hint)
            
            # Background process: Deliver hints quando replica voltar
            def deliver_hints_periodically():
                while True:
                    pending_hints = hint_storage.get_all()
                    for hint in pending_hints:
                        if hint['target_replica'].is_alive():
                            try:
                                hint['target_replica'].write(
                                    hint['key'],
                                    hint['value']
                                )
                                hint_storage.delete(hint)  # Delivered!
                            except Exception:
                                pass  # Retry later
                    time.sleep(60)  # Check every minute
        
        # Benefícios:
        # - Zero writes lost durante temporary failures
        # - Eventual consistency mantida
        # - Availability não degradada
        
        # Limitações:
        # - Hints armazenados por max 3 horas (default Cassandra)
        # - Se node não voltar em 3h, hints descartados
        # - Requer anti-entropy repair para recuperar
    
    # Read Repair
    def read_repair(self):
        """
        Detectar e corrigir inconsistências durante reads
        """
        def read_with_repair(key, replicas):
            # 1. Read de múltiplas replicas
            responses = []
            for replica in replicas[:3]:  # Read from 3 replicas
                try:
                    data = replica.read(key)
                    responses.append((replica, data))
                except Exception:
                    pass  # Ignore failed replica
            
            # 2. Compare responses
            if len(responses) < 2:
                return responses[0][1] if responses else None
            
            # 3. Detect inconsistency
            values = [r[1] for r in responses]
            if not all_equal(values):
                # Inconsistency detected!
                # Choose most recent based on timestamp
                latest = max(responses, key=lambda r: r[1]['timestamp'])
                
                # 4. Async repair: Update stale replicas
                async def repair_stale_replicas():
                    for replica, data in responses:
                        if data != latest[1]:
                            replica.write(key, latest[1])
                            print(f"Repaired stale data on {replica}")
                
                asyncio.create_task(repair_stale_replicas())
                
                return latest[1]
            
            return responses[0][1]
        
        # Benefícios:
        # - Self-healing: Inconsistências corrigidas automaticamente
        # - No admin intervention: Totalmente automático
        # - Eventual consistency: Converge com o tempo
        
        # Cassandra: Read repair probability configurável (default 10%)
        # - Trade-off: 100% = consistência mais rápida mas mais overhead
    
    # Anti-Entropy Repair
    def anti_entropy_repair(self):
        """
        Background process: Comparar e sincronizar todas as replicas
        """
        # Cassandra: nodetool repair
        def full_repair():
            """
            Scheduled repair: Comparar Merkle trees
            """
            # 1. Cada replica constrói Merkle tree dos dados
            #    - Tree hash: Eficiente comparar grandes datasets
            #    - Detecta diferenças sem transferir todos os dados
            
            # 2. Replicas trocam Merkle trees
            # 3. Identificam partitions com diferenças
            # 4. Sincronizam apenas partitions diferentes
            
            # Scheduling:
            # - Run semanalmente ou mensalmente
            # - Off-peak hours (overhead significativo)
            # - Required: Para evitar data loss de hints expiradas
            
            # Performance impact:
            # - CPU: 10-30% durante repair
            # - Network: Alta bandwidth para sync
            # - Latency: +10-50ms para requests durante repair
        
        # DynamoDB: Automatic continuous repair
        # - Background process sempre ativo
        # - Compara replicas continuamente
        # - Sem intervenção manual necessária

#### Multi-Region Availability

```python
class MultiRegionAvailability:
    """
    Disponibilidade cross-region
    """
    
    # DynamoDB Global Tables
    def dynamodb_global_tables(self):
        """
        Multi-master replication cross-region
        """
        config = {
            'regions': ['us-east-1', 'eu-west-1', 'ap-southeast-1'],
            'replication': 'Bi-directional multi-master',
            'consistency': 'Eventual (typically <1 second)',
            
            'availability': {
                'regional_failure': 'Traffic routes to other regions',
                'rto': 'Seconds to minutes',  # Failover time
                'rpo': '<1 second',  # Data loss potential
                'sla': '99.999% (5 noves)'
            },
            
            'conflict_resolution': {
                'strategy': 'Last Writer Wins (LWW)',
                'based_on': 'Timestamp comparison',
                'alternative': 'Custom Lambda resolver'
            }
        }
        
        # Scenario: us-east-1 region failure
        # 1. Health checks detect region down
        # 2. Route 53 routes traffic to eu-west-1 e ap-southeast-1
        # 3. Users experience: 2-5 min downtime (DNS propagation)
        # 4. Data: Zero loss (replicated < 1s before failure)
        # 5. After recovery: us-east-1 auto-syncs com outras regiões
        
        # Write conflicts:
        # T0: US user writes {name: 'John Smith'}
        # T0: EU user writes {name: 'John Doe'} (same item, concurrent)
        # 
        # Resolution:
        # - Compare timestamps
        # - Most recent write wins
        # - Both regions converge to winner
        # - Loser's write is lost (acceptable para maioria dos casos)
        
        return config
    
    # Cassandra Multi-DC
    def cassandra_multi_dc(self):
        """
        Cassandra: Multi-datacenter deployment
        """
        # Setup: 3 datacenters
        # - DC1: us-east (3 nodes, RF=3)
        # - DC2: eu-west (3 nodes, RF=3)
        # - DC3: ap-south (3 nodes, RF=3)
        
        # Write path com LOCAL_QUORUM:
        def write_local_quorum(key, value):
            # Client writes to local DC (ex: us-east)
            # 1. Coordinator node em us-east
            # 2. Write para LOCAL replicas (3 nodes em us-east)
            # 3. Aguarda LOCAL_QUORUM (2 de 3 em us-east)
            # 4. Returns success (10-20ms)
            
            # Background: Async replication para outros DCs
            # 5. us-east → eu-west (100ms)
            # 6. us-east → ap-south (200ms)
            # 
            # Cross-DC replication: Eventual
            
            return "Write committed locally"
        
        # Read path com LOCAL_QUORUM:
        def read_local_quorum(key):
            # Read from local DC apenas
            # Fast: 10-20ms
            # Consistency: LOCAL strong, GLOBAL eventual
            
            return local_dc.read(key, CL='LOCAL_QUORUM')
        
        # DC failure scenario:
        # - us-east DC completamente down
        # - Clients automaticamente failover para eu-west
        # - Zero data loss (multi-DC replication)
        # - Performance: Slightly higher latency (cross-region)
        # - Capacity: 66% (2 de 3 DCs remaining)
        
        # Benefits:
        # - Disaster recovery: DC failure tolerado
        # - Geo-distribution: Low latency global
        # - Compliance: Data residency reqs
        
        # Trade-offs:
        # - 3x cost (3 full replicas)
        # - Complexity: Config e monitoring
        # - Consistency: Cross-DC eventual

#### Backup and Recovery

```python
class BackupRecovery:
    """
    Estratégias de backup para disaster recovery
    """
    
    # DynamoDB Point-in-Time Recovery (PITR)
    def dynamodb_pitr(self):
        """
        Continuous backups com 35 dias de retention
        """
        # Enable PITR:
        dynamodb.update_continuous_backups(
            TableName='Users',
            PointInTimeRecoverySpecification={
                'PointInTimeRecoveryEnabled': True
            }
        )
        
        # Cost: 20% do storage cost (exemplo: $0.20/GB-month)
        
        # Recovery scenarios:
        scenarios = {
            'Accidental delete': {
                'problem': 'Desenvolvedor deletou 10K items por engano',
                'solution': 'Restore table para 5 minutos antes',
                'rto': '10-30 minutos',
                'rpo': '1 segundo'
            },
            
            'Data corruption': {
                'problem': 'Bug alterou dados incorretamente',
                'solution': 'Restore table para antes do deploy buggy',
                'rto': '10-30 minutos',
                'rpo': '1 segundo'
            },
            
            'Compliance': {
                'problem': 'Audit requer dados de 30 dias atrás',
                'solution': 'Restore para timestamp específico',
                'rto': 'Não crítico',
                'rpo': 'Perfeito (point-in-time)'
            }
        }
        
        # Restore process:
        dynamodb.restore_table_to_point_in_time(
            SourceTableName='Users',
            TargetTableName='Users-Restored-2024-01-15',
            RestoreDateTime=datetime(2024, 1, 15, 10, 30, 0)
        )
        
        # Notes:
        # - Restore cria NOVA table (não sobrescreve original)
        # - Aplicação deve apontar para nova table
        # - Original table pode ser mantida para comparação
        
        return scenarios
    
    # On-Demand Backups
    def on_demand_backups(self):
        """
        Snapshots manuais para long-term retention
        """
        # DynamoDB on-demand backup:
        response = dynamodb.create_backup(
            TableName='Users',
            BackupName='users-backup-pre-migration-2024-01-15'
        )
        
        # Properties:
        # - Full snapshot: Toda a table em momento específico
        # - Retention: Indefinido (até deletar manualmente)
        # - Cost: $0.10/GB-month (metade do custo de PITR)
        # - Recovery: Restore para new table
        
        # Use cases:
        # - Before major migrations
        # - Compliance (long-term retention)
        # - Disaster recovery tests
        
        # Restore:
        dynamodb.restore_table_from_backup(
            TargetTableName='Users-Restored',
            BackupArn=response['BackupDetails']['BackupArn']
        )
    
    # Redis Persistence
    def redis_persistence(self):
        """
        Redis: RDB snapshots + AOF log
        """
        # RDB (Redis Database Snapshot)
        rdb_config = {
            'save_intervals': [
                '900 1',    # Save se ≥1 change em 900s
                '300 10',   # Save se ≥10 changes em 300s
                '60 10000'  # Save se ≥10K changes em 60s
            ],
            'file': 'dump.rdb',
            'compression': 'yes',
            
            'pros': [
                'Compact single file',
                'Fast restart',
                'Good for backups'
            ],
            'cons': [
                'Potential data loss (between snapshots)',
                'Fork() pode ser lento para large datasets',
                'Not suitable for zero data loss'
            ]
        }
        
        # AOF (Append-Only File)
        aof_config = {
            'appendonly': 'yes',
            'appendfsync': 'everysec',  # Fsync every second
            # Alternatives: 'always' (slow), 'no' (fast, risky)
            
            'file': 'appendonly.aof',
            'rewrite_threshold': '100%',  # Rewrite quando 2x tamanho
            
            'pros': [
                'Better durability (max 1s data loss)',
                'Append-only: Corruption resistant',
                'Human-readable log'
            ],
            'cons': [
                'Larger file size',
                'Slower restart (replay log)',
                'Slight performance overhead'
            ]
        }
        
        # Best practice: RDB + AOF
        # - RDB para fast recovery
        # - AOF para durability
        # - Trade-off: 2x storage, melhor de ambos
        
        # ElastiCache for Redis (AWS):
        elasticache_config = {
            'daily_backups': 'Automatic RDB snapshots',
            'retention': '1-35 days',
            'manual_snapshots': 'Unlimited retention',
            'restore': 'Create new cluster from snapshot',
            
            'multi_az': {
                'enabled': True,
                'automatic_failover': '< 1 minute',
                'read_replica': 'Promoted to primary'
            }
        }

#### Availability SLAs

```python
class AvailabilitySLAs:
    """
    Service Level Agreements e garantias de disponibilidade
    """
    
    aws_nosql_slas = {
        'DynamoDB': {
            'single_region': {
                'sla': '99.99%',  # 4 noves
                'downtime_month': '4.38 minutos',
                'architecture': '3 AZs, synchronous replication'
            },
            'global_tables': {
                'sla': '99.999%',  # 5 noves
                'downtime_month': '26.3 segundos',
                'architecture': 'Multi-region, active-active'
            }
        },
        
        'ElastiCache': {
            'single_node': {
                'sla': 'None',  # Sem SLA!
                'reason': 'Single point of failure'
            },
            'cluster_mode': {
                'sla': '99.99%',
                'downtime_month': '4.38 minutos',
                'architecture': 'Multi-AZ, automatic failover'
            }
        },
        
        'DocumentDB': {
            'sla': '99.99%',
            'downtime_month': '4.38 minutos',
            'architecture': '3 AZs, 6 replicas'
        }
    }
    
    # Calculating Composite Availability
    def composite_availability(self):
        """
        Calcular availability de sistema composto
        """
        # Sistema: API Gateway → Lambda → DynamoDB
        
        component_availability = {
            'api_gateway': 0.9999,  # 99.99%
            'lambda': 0.9999,       # 99.99%
            'dynamodb': 0.9999      # 99.99%
        }
        
        # Serial components: Multiply
        system_availability = (
            component_availability['api_gateway'] *
            component_availability['lambda'] *
            component_availability['dynamodb']
        )
        # = 0.9997 = 99.97%
        # Downtime: ~13 minutos/mês
        
        # Com fallback/redundancy:
        # Primary: DynamoDB (99.99%)
        # Fallback: Cache (99.9%)
        # Combined = 1 - (1 - 0.9999) * (1 - 0.999)
        #          = 1 - 0.0001 * 0.001
        #          = 1 - 0.0000001
        #          = 0.9999999 = 99.99999% (7 noves!)
        
        return system_availability

### Advantages, Trade-offs, and Considerations (Vantagens, Trade-offs e Considerações)

Key-Value databases oferecem vantagens significativas mas também vêm com trade-offs que devem ser cuidadosamente considerados no design de sistemas.

#### Vantagens dos Key-Value Stores

```python
class KeyValueAdvantages:
    """
    Principais vantagens dos Key-Value databases
    """
    
    # 1. Performance Excepcional
    def performance_advantage(self):
        """
        Latência ultra-baixa e high throughput
        """
        performance_comparison = {
            'Operation': 'GET by Primary Key',
            
            'Redis (in-memory)': {
                'latency_p50': '0.1-0.3ms',
                'latency_p99': '0.5-1ms',
                'throughput': '100,000+ ops/s per node'
            },
            
            'DynamoDB (SSD)': {
                'latency_p50': '2-5ms',
                'latency_p99': '10-20ms',
                'throughput': '1,000,000+ ops/s per table'
            },
            
            'PostgreSQL (relational)': {
                'latency_p50': '5-15ms',
                'latency_p99': '50-200ms',
                'throughput': '10,000-50,000 ops/s per instance',
                'note': 'Com índice otimizado, sem JOINs'
            }
        }
        
        # Por que tão rápido?
        reasons = {
            'Simple data model': 'Sem JOINs, sem query planning',
            'O(1) lookup': 'Hash table direto',
            'No schema validation': 'Valor opaco, sem parsing',
            'Optimized storage': 'LSM trees (writes), B-trees (reads)',
            'In-memory option': 'Redis mantém tudo em RAM'
        }
        
        # Use case: Real-time bidding
        real_time_bidding = """
        Ad Exchange: 100ms total para:
        1. Receber bid request
        2. Lookup user profile (DynamoDB: 2-5ms) ✓
        3. Check targeting rules
        4. Calculate bid price  
        5. Return bid response
        
        Com SQL: 20-50ms só para lookup → Não atende SLA
        Com Key-Value: 2-5ms → Sobra tempo para lógica
        """
        
        return performance_comparison, reasons, real_time_bidding
    
    # 2. Escalabilidade Linear
    def scalability_advantage(self):
        """
        Add nodes = Add capacity (quase linear)
        """
        scalability_example = {
            'Initial setup': {
                'nodes': 3,
                'throughput': '30K ops/s',
                'capacity': '300GB',
                'cost': '$300/month'
            },
            
            'After doubling nodes': {
                'nodes': 6,
                'throughput': '58K ops/s',  # ~2x
                'capacity': '600GB',        # 2x
                'cost': '$600/month',       # 2x
                'efficiency': '97%'         # Quase linear!
            },
            
            'Comparison with relational': {
                'vertical_scaling': 'Limit físico (máquina maior)',
                'sharding_manual': 'Complexo, requer rewrite application',
                'efficiency': '50-70% (overhead de coordenação)'
            }
        }
        
        # Por que escalabilidade tão boa?
        reasons = [
            'Particionamento natural por key',
            'Sem coordenação cross-partition',
            'Stateless request routing',
            'Consistent hashing (minimal rebalancing)'
        ]
        
        return scalability_example, reasons
    
    # 3. Alta Disponibilidade
    def availability_advantage(self):
        """
        Built-in replication e failover
        """
        availability_features = {
            'DynamoDB': {
                'replication': '3 AZs automático',
                'failover': 'Transparente (segundos)',
                'sla': '99.99% (single-region), 99.999% (global)',
                'manual_intervention': 'Zero'
            },
            
            'Cassandra': {
                'replication': 'RF configurável (tipicamente 3)',
                'failover': 'Automático via gossip protocol',
                'sla': 'Depende de configuração',
                'tunable_consistency': 'ONE, QUORUM, ALL'
            },
            
            'Comparison': {
                'Traditional DB': {
                    'replication': 'Manual setup, master-slave',
                    'failover': '30-300 segundos, pode precisar intervenção',
                    'sla': '99.9-99.95% típico',
                    'manual_intervention': 'Frequentemente necessário'
                }
            }
        }
        
        # Zero-downtime operations:
        zero_downtime_ops = [
            'Add/remove nodes: Rolling, sem downtime',
            'Schema changes: Não necessário (schema-less)',
            'Backups: Online, sem impacto',
            'Upgrades: Rolling upgrades',
            'Disaster recovery: Multi-region automático'
        ]
        
        return availability_features, zero_downtime_ops
    
    # 4. Simplicidade Operacional
    def operational_simplicity(self):
        """
        Menor overhead operacional
        """
        comparison = {
            'Setup Complexity': {
                'Key-Value (DynamoDB)': {
                    'setup_time': '5 minutos (CreateTable API)',
                    'expertise_required': 'Básico AWS',
                    'infra_management': 'Zero (fully managed)'
                },
                'Key-Value (Redis)': {
                    'setup_time': '15-30 minutos',
                    'expertise_required': 'Intermediate',
                    'infra_management': 'Medium (ElastiCache) to High (self-managed)'
                },
                'Relational (PostgreSQL RDS)': {
                    'setup_time': '30-60 minutos',
                    'expertise_required': 'Intermediate to Advanced',
                    'infra_management': 'Medium (parameter tuning, vacuuming)'
                }
            },
            
            'Day-to-day Operations': {
                'Key-Value': [
                    'No vacuuming/reindexing',
                    'No query optimization',
                    'No schema migrations',
                    'Auto-scaling available',
                    'Monitoring: Simples métricas (throughput, latency)'
                ],
                'Relational': [
                    'Regular vacuuming (PostgreSQL)',
                    'Query plan analysis',
                    'Schema migration planning',
                    'Manual scaling coordination',
                    'Monitoring: Complexo (locks, deadlocks, slow queries)'
                ]
            }
        }
        
        return comparison
    
    # 5. Cost-Effectiveness (Para casos de uso apropriados)
    def cost_advantage(self):
        """
        Custo-benefício para workloads apropriados
        """
        # Session store example: 1M active sessions
        session_store_costs = {
            'DynamoDB': {
                'storage': '1M sessions × 2KB = 2GB × $0.25/GB = $0.50',
                'throughput': '1000 RCU + 200 WCU = $250',
                'total_monthly': '$250.50',
                'notes': 'On-demand seria ~$300'
            },
            
            'Redis (ElastiCache)': {
                'instances': '1× cache.r6g.large = $150',
                'data_transfer': '$20',
                'total_monthly': '$170',
                'notes': 'Mais barato mas volatile (sem persistence)'
            },
            
            'PostgreSQL (RDS)': {
                'instance': '1× db.r6g.large = $300',
                'storage': '10GB × $0.115 = $1.15',
                'iops': '$100 (provisioned)',
                'total_monthly': '$401',
                'notes': 'Overkill para session store, performance inferior'
            }
        }
        
        # Key insight:
        key_insight = """
        Key-Value é cost-effective quando:
        1. Access pattern é simples (GET/PUT by key)
        2. Alto volume de requests
        3. Não precisa de queries complexas
        4. Schema flexível/evolutivo
        
        Relational é melhor quando:
        1. Queries complexas com JOINs
        2. Transactions multi-table
        3. Strong consistency critical
        4. Ad-hoc analytics
        """
        
        return session_store_costs, key_insight

#### Trade-offs e Limitações

```python
class KeyValueTradeoffs:
    """
    Limitações e trade-offs a considerar
    """
    
    # 1. Queries Limitadas
    def limited_queries(self):
        """
        Apenas queries por primary key (sem rich queries)
        """
        limitations = {
            'No support for': [
                'JOINs entre tables',
                'Aggregations (SUM, AVG, GROUP BY)',
                'Full-text search',
                'Complex filters (múltiplos atributos)',
                'Sorting por atributos não-key'
            ],
            
            'Workarounds': {
                'Secondary Indexes': {
                    'solution': 'DynamoDB GSI/LSI, maintain manual indexes',
                    'trade_off': 'Eventual consistency, custo extra, complexity'
                },
                'Denormalization': {
                    'solution': 'Duplicate data, embed related entities',
                    'trade_off': 'Storage overhead, potential inconsistency'
                },
                'Application-level JOINs': {
                    'solution': 'Fetch múltiplos items, join in code',
                    'trade_off': 'N+1 queries, higher latency'
                },
                'Specialized indexes': {
                    'solution': 'Elasticsearch para search, data warehouse para analytics',
                    'trade_off': 'Multiple databases, sync overhead'
                }
            }
        }
        
        # Example: E-commerce query impossível em Key-Value:
        impossible_query = """
        SQL (fácil):
        SELECT p.name, p.price, c.name as category, AVG(r.rating)
        FROM products p
        JOIN categories c ON p.category_id = c.id
        JOIN reviews r ON p.id = r.product_id
        WHERE p.price < 100
          AND c.name = 'Electronics'
          AND r.rating >= 4
        GROUP BY p.id
        HAVING COUNT(r.id) >= 10
        ORDER BY AVG(r.rating) DESC
        LIMIT 10
        
        Key-Value: Impossível nativamente!
        Requer:
        1. Denormalização massiva
        2. Múltiplos índices secundários
        3. Application logic para aggregation
        4. Ou usar banco secundário (Elasticsearch)
        """
        
        return limitations, impossible_query
    
    # 2. Eventual Consistency Challenges
    def eventual_consistency_challenges(self):
        """
        Consistency model pode causar problemas
        """
        problems = {
            'Read-your-writes': {
                'problem': 'User faz write, immediately read vê valor antigo',
                'example': """
                # User atualiza foto de perfil
                dynamodb.put_item(Item={'user_id': '123', 'photo': 'new.jpg'})
                
                # Immediately lê (pode ser de replica stale)
                user = dynamodb.get_item(Key={'user_id': '123'})
                photo = user['photo']  # Pode ser 'old.jpg'! ❌
                """,
                'solutions': [
                    'Strong consistency reads (2x latência)',
                    'Session affinity (sticky sessions)',
                    'Optimistic UI (update client-side immediately)'
                ]
            },
            
            'Causality violations': {
                'problem': 'Events aparecem fora de ordem',
                'example': """
                # Thread 1: Create order
                orders.put(order_id='001', status='created')
                
                # Thread 2: Update order (happens-after)
                orders.put(order_id='001', status='paid')
                
                # Thread 3: Read order (pode ver 'created' após 'paid'!)
                order = orders.get(order_id='001')
                # Due to replication lag, pode ver status='created'
                # Mesmo que 'paid' já foi escrito!
                """,
                'solutions': [
                    'Version vectors',
                    'Logical clocks (Lamport timestamps)',
                    'Strong consistency para critical operations'
                ]
            },
            
            'Lost updates': {
                'problem': 'Concurrent writes podem se sobrescrever',
                'example': """
                # User A e User B editam perfil simultaneamente
                # A: Read profile → Edit name → Write
                # B: Read profile → Edit email → Write
                # Result: B's write sobrescreve name edit de A ❌
                """,
                'solutions': [
                    'Conditional writes (optimistic locking)',
                    'Atomic operations (UPDATE SET específico)',
                    'Version numbers ou timestamps'
                ]
            }
        }
        
        # When eventual consistency is acceptable:
        acceptable_use_cases = [
            'Social media feeds (lag de segundos OK)',
            'View counts, likes (approximate values)',
            'Product recommendations (staleness não crítica)',
            'Logging, metrics (aggregate data)',
            'Content delivery (CDN caching)'
        ]
        
        # When strong consistency is required:
        required_use_cases = [
            'Financial transactions',
            'Inventory management',
            'User authentication',
            'Order processing',
            'Healthcare records'
        ]
        
        return problems, acceptable_use_cases, required_use_cases
    
    # 3. Storage Limitations
    def storage_limitations(self):
        """
        Limitações de tamanho e estrutura
        """
        limitations = {
            'DynamoDB': {
                'item_size_max': '400 KB',
                'partition_key_max': '2048 bytes',
                'sort_key_max': '1024 bytes',
                'attribute_name_max': '255 bytes',
                
                'workarounds': {
                    'Large items': 'Store em S3, reference em DynamoDB',
                    'Large attributes': 'Compress, or split into múltiplos items'
                }
            },
            
            'Redis': {
                'key_size_max': '512 MB',
                'value_size_max': '512 MB',
                'memory_limit': 'RAM size (sem overflow to disk)',
                'collection_size_max': '2^32-1 elements',
                
                'practical_limits': {
                    'recommended_value_size': '< 10 MB',
                    'recommended_collection_size': '< 1M elements',
                    'reason': 'Performance degradation, memory fragmentation'
                }
            },
            
            'Cassandra': {
                'partition_size_recommended': '< 100 MB',
                'partition_size_max': '2 GB (warning threshold)',
                'row_size_max': '2 GB',
                
                'problems_with_large_partitions': [
                    'Read latency degradation',
                    'Compaction overhead',
                    'Tombstone accumulation',
                    'Memory pressure'
                ]
            }
        }
        
        # Example problema: Storing chat history
        chat_history_problem = """
        Bad design:
        partition_key = conversation_id
        clustering_key = timestamp
        
        Problem: Popular conversation com 1M mensagens
        - Partition size: 1M × 2KB = 2GB (TOO BIG!)
        - Read entire conversation: Timeout
        - Compaction: Hours, high CPU
        
        Better design:
        partition_key = (conversation_id, bucket)
        bucket = year_month (ex: '2024-01')
        clustering_key = timestamp
        
        Benefit: Cada partition ~30K mensagens = 60MB ✓
        Trade-off: Query cross-buckets requer multiple reads
        """
        
        return limitations, chat_history_problem
    
    # 4. Operational Complexity (Para self-managed)
    def operational_complexity(self):
        """
        Complexidade operacional para soluções self-managed
        """
        # Cassandra operational burden:
        cassandra_ops = {
            'Daily tasks': [
                'Monitor compaction queues',
                'Check tombstone ratios',
                'Review GC pause times',
                'Monitor disk utilization'
            ],
            
            'Weekly tasks': [
                'Analyze slow queries',
                'Review partition sizes',
                'Check repair status',
                'Capacity planning'
            ],
            
            'Monthly tasks': [
                'Full cluster repair (anti-entropy)',
                'Rolling restarts para patches',
                'Performance tuning',
                'Backup validation'
            ],
            
            'Expertise required': [
                'JVM tuning (heap, GC)',
                'Network configuration (gossip, timeouts)',
                'Data modeling best practices',
                'Troubleshooting distributed systems',
                'Understanding consistency levels'
            ],
            
            'Team size': '1-2 dedicated DBAs para cluster production'
        }
        
        # Managed services advantage:
        managed_comparison = {
            'DynamoDB (fully managed)': {
                'operational_burden': 'Minimal',
                'expertise_required': 'Basic AWS knowledge',
                'team_size': '0 dedicated DBAs',
                'cost_premium': '20-40% mais caro que self-managed',
                'trade_off': 'Worth it para maioria dos casos'
            },
            
            'ElastiCache (managed Redis)': {
                'operational_burden': 'Low',
                'expertise_required': 'Intermediate Redis knowledge',
                'team_size': '0.5 DBA (part-time)',
                'cost_premium': '30-50%',
                'benefits': 'Auto-failover, backups, patching automático'
            }
        }
        
        return cassandra_ops, managed_comparison
    
    # 5. Cost at Scale
    def cost_at_scale(self):
        """
        Custos podem escalar rapidamente
        """
        cost_examples = {
            'DynamoDB - High throughput case': {
                'workload': '100K reads/s + 20K writes/s',
                'requirements': '50K RCU + 20K WCU',
                'provisioned_cost': '$28,800/month',
                'on_demand_cost': '$50,000+/month',
                
                'optimization_strategies': [
                    'Caching (Redis): Reduz para 10K RCU = $5,000 savings',
                    'Compression: Reduz item sizes = 30% savings',
                    'Batch operations: Reduce WCU consumption',
                    'DynamoDB Accelerator (DAX): $3,000/m mas evita $20K em RCU'
                ]
            },
            
            'Storage costs': {
                'DynamoDB': {
                    'rate': '$0.25/GB-month',
                    '1TB example': '$250/month',
                    'note': 'Competitive para data frequentemente acessado'
                },
                'S3': {
                    'rate': '$0.023/GB-month',
                    '1TB example': '$23/month',
                    'note': 'Muito mais barato, mas não queryable'
                },
                'EBS (PostgreSQL)': {
                    'rate': '$0.10/GB-month',
                    '1TB example': '$100/month',
                    'note': 'Mais barato mas I/O costs extras'
                }
            },
            
            'Hidden costs': [
                'Data transfer: Significativo para multi-region',
                'Backups: 20% adicional para PITR',
                'Global tables: 3x replication cost',
                'GSI: Cada índice = custo extra de storage + throughput'
            ]
        }
        
        # When Key-Value becomes expensive:
        expensive_scenarios = [
            'High write throughput com large items',
            'Multiple GSIs (each costs like main table)',
            'Global tables com heavy writes',
            'On-demand pricing com steady high load',
            'Storage-heavy com infrequent access'
        ]
        
        # Cost optimization tips:
        optimization_tips = [
            'Use caching layer (Redis) para hot data',
            'Compress large attributes',
            'Archive cold data para S3',
            'Right-size provisioned capacity',
            'Delete unused GSIs',
            'Use DynamoDB Streams judiciosamente',
            'Consider reserved capacity para steady workloads'
        ]
        
        return cost_examples, expensive_scenarios, optimization_tips

#### Quando Usar Key-Value Stores

```python
class WhenToUseKeyValue:
    """
    Decision matrix para escolher Key-Value database
    """
    
    ideal_use_cases = {
        'Session Management': {
            'why_key_value': [
                'Simple GET/SET por session ID',
                'High throughput (todos requests)',
                'TTL automático (session expiry)',
                'Low latency requirement (<10ms)'
            ],
            'recommended': 'Redis ou DynamoDB',
            'anti_pattern': None
        },
        
        'Caching Layer': {
            'why_key_value': [
                'Ultra-low latency (<1ms)',
                'High read throughput',
                'TTL para cache invalidation',
                'Simple key-based access'
            ],
            'recommended': 'Redis (ElastiCache)',
            'anti_pattern': 'Não usar para source of truth'
        },
        
        'User Profiles': {
            'why_key_value': [
                'Single-item access por user ID',
                'Flexible schema (different users, different attributes)',
                'High availability requirement',
                'Global distribution (multi-region)'
            ],
            'recommended': 'DynamoDB Global Tables',
            'anti_pattern': 'Se precisar queries complexas cross-user'
        },
        
        'Shopping Carts': {
            'why_key_value': [
                'Temporary data (pode perder sem disaster)',
                'High availability > consistency',
                'Simple add/remove operations',
                'Per-user isolation'
            ],
            'recommended': 'DynamoDB ou Redis',
            'anti_pattern': None
        },
        
        'Real-time Analytics': {
            'why_key_value': [
                'Atomic counters (page views, clicks)',
                'High write throughput',
                'Low latency increments',
                'Approximate values acceptable'
            ],
            'recommended': 'Redis (Counters, HyperLogLog)',
            'anti_pattern': 'Para analytics históricos, usar warehouse'
        },
        
        'IoT Data Ingestion': {
            'why_key_value': [
                'High write throughput (milhões/s)',
                'Time-series data',
                'Device-based partitioning',
                'Simple append operations'
            ],
            'recommended': 'DynamoDB ou Cassandra',
            'anti_pattern': 'Para queries analíticas, sync para warehouse'
        }
    }
    
    not_recommended_use_cases = {
        'Reporting/Analytics': {
            'why_not': 'No aggregations, no complex queries',
            'use_instead': 'Redshift, BigQuery, or sync data para warehouse'
        },
        
        'Content Management com Rich Search': {
            'why_not': 'No full-text search, no complex filters',
            'use_instead': 'Elasticsearch, or DynamoDB + Elasticsearch'
        },
        
        'Multi-entity Transactions': {
            'why_not': 'Limited transactions, no cross-table JOINs',
            'use_instead': 'PostgreSQL, or DynamoDB transactions (limited)'
        },
        
        'Graph Relationships': {
            'why_not': 'No efficient graph traversals',
            'use_instead': 'Neptune (graph database)'
        },
        
        'Ad-hoc Queries': {
            'why_not': 'Queries devem ser predefined no design',
            'use_instead': 'SQL database, or denormalize para cada query pattern'
        }
    }
    
    # Decision tree
    def decision_tree(self, requirements):
        """
        Decision tree para escolher database type
        """
        # Q1: Access pattern
        if requirements['access_pattern'] == 'complex_queries':
            return 'SQL Database'
        
        # Q2: Consistency requirements
        if requirements['consistency'] == 'strong_always':
            if requirements['scale'] == 'moderate':
                return 'SQL Database'
            else:
                return 'DynamoDB (strong consistency mode)'
        
        # Q3: Latency requirements
        if requirements['latency'] == 'submillisecond':
            return 'Redis (in-memory)'
        elif requirements['latency'] == 'single_digit_ms':
            return 'DynamoDB'
        
        # Q4: Data model
        if requirements['data_model'] == 'simple_key_value':
            return 'Redis or DynamoDB'
        elif requirements['data_model'] == 'documents_with_nesting':
            return 'DocumentDB or DynamoDB'
        elif requirements['data_model'] == 'time_series':
            return 'Cassandra or Timestream'
        elif requirements['data_model'] == 'graph':
            return 'Neptune'
        
        return 'Evaluate multiple options'
```

### Dynamo: Key-Value Database

Amazon Dynamo, descrito no paper seminal de DeCandia et al. (2007), é o sistema que inspirou toda uma geração de Key-Value databases incluindo DynamoDB, Cassandra, Riak, e Voldemort. Entender Dynamo é fundamental para compreender o design de Key-Value stores modernos.

#### História e Contexto

```python
class DynamoHistory:
    """
    História e contexto do Amazon Dynamo
    """
    
    problem_statement = {
        'year': 2004,
        'context': 'Amazon.com crescimento exponencial',
        'challenges': [
            'Relational databases não escalavam para demanda',
            'Downtime custava milhões de dólares',
            'Black Friday failures eram inaceitáveis',
            'Availability > Consistency para e-commerce'
        ],
        
        'requirements': {
            'availability': '99.99% (4 noves) mínimo',
            'latency': '<300ms P99.9',
            'scalability': 'Linear, adicionar nodes facilmente',
            'simplicity': 'Simple data model, sem queries complexas',
            'durability': 'Zero data loss'
        },
        
        'key_insight': """
        Para shopping cart e session data:
        - Availability é mais crítica que consistency
        - Better to show stale cart than no cart
        - Better to allow duplicate items than block checkout
        - Strong consistency não necessária para maioria das operations
        """
    }
    
    design_principles = {
        'Sacrifice strong consistency': {
            'reason': 'CAP theorem - choose Availability + Partition tolerance',
            'strategy': 'Eventual consistency com conflict resolution'
        },
        
        'Always writable': {
            'reason': 'Never reject writes due to failures',
            'strategy': 'Accept writes mesmo com nodes down (hinted handoff)'
        },
        
        'Incremental scalability': {
            'reason': 'Add/remove one node at a time sem disruption',
            'strategy': 'Consistent hashing'
        },
        
        'Decentralization': {
            'reason': 'No single point of failure',
            'strategy': 'Peer-to-peer architecture, all nodes equal'
        },
        
        'Heterogeneity': {
            'reason': 'Different nodes may have different capacities',
            'strategy': 'Virtual nodes (vnodes) para load balancing'
        }
    }

#### Arquitetura Técnica do Dynamo

```python
class DynamoArchitecture:
    """
    Componentes principais da arquitetura Dynamo
    """
    
    # 1. Consistent Hashing com Virtual Nodes
    def consistent_hashing_with_vnodes(self):
        """
        Particionamento e rebalanceamento eficiente
        """
        architecture = {
            'Hash Ring': {
                'concept': 'Círculo lógico de 0 a 2^128-1',
                'node_placement': 'Cada node tem múltiplas positions (vnodes)',
                'key_placement': 'MD5(key) determina position no ring',
                'ownership': 'Node responsável: primeira position ≥ key hash'
            },
            
            'Virtual Nodes': {
                'count_per_node': '100-256 vnodes',
                'benefit': 'Load balancing fino-grained',
                'add_node': 'Recebe vnodes distribuídos uniformemente',
                'remove_node': 'Seus vnodes redistribuídos uniformemente',
                'heterogeneity': 'Nodes maiores = mais vnodes'
            },
            
            'Example': """
            Cluster: 4 nodes (A, B, C, D)
            Each: 256 vnodes
            Total: 1024 positions no ring
            
            Key 'user:1000' → MD5 → hash=12345678
            Find: Primeira vnode position ≥ 12345678
            Suppose: vnode-542 (pertence ao node B)
            Write to: Node B (+ replicas)
            """
        }
        
        return architecture
    
    # 2. Replication
    def replication_strategy(self):
        """
        Como Dynamo replica dados
        """
        strategy = {
            'Replication Factor (N)': {
                'typical_value': 3,
                'meaning': 'Cada key replicada em N nodes',
                'selection': 'N successor nodes clockwise no ring'
            },
            
            'Coordinator': {
                'role': 'Node que recebe request torna-se coordinator',
                'responsibilities': [
                    'Determina N nodes responsáveis pela key',
                    'Envia writes para N nodes',
                    'Aguarda responses baseado em W',
                    'Retorna success ao client'
                ]
            },
            
            'Quorum-based Replication': {
                'N': 'Replication factor (total replicas)',
                'R': 'Read quorum (# nodes para read)',
                'W': 'Write quorum (# nodes para write)',
                'rule': 'R + W > N garante consistency',
                
                'typical_config': {
                    'N': 3,
                    'R': 2,
                    'W': 2,
                    'guarantee': 'Read sempre vê última write (R+W=4 > N=3)'
                },
                
                'high_availability_config': {
                    'N': 3,
                    'R': 1,
                    'W': 1,
                    'trade_off': 'Máxima availability, eventual consistency'
                }
            }
        }
        
        return strategy
    
    # 3. Versioning e Conflict Resolution
    def versioning_and_conflicts(self):
        """
        Como Dynamo trata updates conflitantes
        """
        versioning = {
            'Vector Clocks': {
                'purpose': 'Track causal relationships entre versions',
                'structure': 'Lista de (node, counter) pairs',
                
                'example': """
                Initial: [{A: 1}]           # Node A write version 1
                Branch 1: [{A: 1}, {B: 1}]  # Node B concurrent write
                Branch 2: [{A: 1}, {C: 1}]  # Node C concurrent write
                
                Conflict: Duas versions incomparáveis!
                - Não há "latest" version clear
                - Ambas são válidas (concurrent updates)
                - Require reconciliation
                """
            },
            
            'Conflict Detection': {
                'comparable': {
                    'v1': [{A: 1}],
                    'v2': [{A: 2}],
                    'relationship': 'v2 descends from v1 (A: 2 > 1)',
                    'action': 'Use v2, discard v1'
                },
                'concurrent': {
                    'v1': [{A: 1}, {B: 1}],
                    'v2': [{A: 1}, {C: 1}],
                    'relationship': 'Neither descends from other',
                    'action': 'Both kept, require reconciliation'
                }
            },
            
            'Conflict Resolution': {
                'syntactic_reconciliation': {
                    'strategy': 'Last-write-wins (timestamp)',
                    'problem': 'Can lose data',
                    'use_case': 'Quando perda de data é aceitável'
                },
                
                'semantic_reconciliation': {
                    'strategy': 'Application-specific merge logic',
                    'example': """
                    Shopping Cart Merge:
                    Cart v1: [item A, item B]
                    Cart v2: [item A, item C]
                    Merged: [item A, item B, item C]  # Union
                    
                    Better: Allow duplicates, let user remove
                    """,
                    'use_case': 'Quando merge semântico faz sentido'
                }
            }
        }
        
        return versioning
    
    # 4. Gossip Protocol
    def failure_detection_gossip(self):
        """
        Membership e failure detection via gossip
        """
        gossip = {
            'Protocol': {
                'frequency': 'Cada node gossips a cada 1 segundo',
                'target': 'Random 1-3 nodes',
                'payload': 'Node ID, timestamp, token ranges owned'
            },
            
            'Failure Detection': {
                'mechanism': 'No heartbeat em T segundos → suspect',
                'T_typical': '10-30 segundos',
                'propagation': 'Exponential (log N hops para full cluster)',
                
                'example': """
                Cluster: 100 nodes
                Node A fails
                
                t=0s: A stops sending heartbeats
                t=10s: 3 nodes detect A down
                t=11s: 3×3 = 9 nodes know A down
                t=12s: 9×3 = 27 nodes know
                t=13s: 27×3 = 81 nodes know
                t=14s: 100 nodes know A down
                
                Total: ~14 segundos para full propagation
                """
            },
            
            'Permanent vs Temporary Failure': {
                'temporary': 'Node down <T hours → hinted handoff',
                'permanent': 'Node down >T hours → replica transfer',
                'T_cassandra': '3 hours default'
            }
        }
        
        return gossip
    
    # 5. Hinted Handoff
    def hinted_handoff_mechanism(self):
        """
        Garantir writes durante temporary failures
        """
        mechanism = {
            'Normal Write': """
            Key hash → Nodes [A, B, C]
            All healthy: Write to A, B, C
            """,
            
            'Node Down Scenario': """
            Key hash → Nodes [A, B, C]
            B is down
            
            Coordinator writes to:
            - A: Normal write
            - D: Hinted handoff (temporary holder)
            - C: Normal write
            
            D stores hint: "This data belongs to B"
            """,
            
            'Hint Delivery': """
            B comes back online
            D detects B is up (via gossip)
            D transfers hints to B
            D deletes local hints
            
            Result: B now has consistent data
            """,
            
            'Benefits': [
                'Zero write failures durante temporary outages',
                'Eventual consistency maintained',
                'No manual intervention required'
            ],
            
            'Limitations': [
                'Hints stored max T hours (3h default)',
                'If B down > T, hints expire',
                'Requires anti-entropy repair to recover'
            ]
        }
        
        return mechanism

#### Influência e Legado

```python
class DynamoLegacy:
    """
    Como Dynamo influenciou databases modernos
    """
    
    influenced_systems = {
        'AWS DynamoDB (2012)': {
            'adoption': 'Managed service inspirado em Dynamo paper',
            'differences': [
                'Fully managed (vs self-hosted)',
                'Automatic partitioning',
                'No vector clocks (simplified consistency)',
                'Last-write-wins conflict resolution'
            ],
            'similarities': [
                'Key-value model',
                'Consistent hashing',
                'Tunable consistency (eventual vs strong)',
                'Multi-AZ replication'
            ]
        },
        
        'Apache Cassandra (2008)': {
            'origin': 'Facebook, baseado em Dynamo + BigTable',
            'dynamo_concepts': [
                'Consistent hashing com vnodes',
                'Gossip protocol',
                'Hinted handoff',
                'Tunable consistency (ONE, QUORUM, ALL)',
                'Peer-to-peer architecture'
            ],
            'additions': [
                'Column-family data model (de BigTable)',
                'CQL query language',
                'Secondary indexes',
                'Materialized views'
            ]
        },
        
        'Riak (2009)': {
            'adoption': 'Implementação quase-literal do Dynamo paper',
            'additions': [
                'Built-in conflict resolution (CRDTs)',
                'Full-text search integration',
                'MapReduce support'
            ]
        },
        
        'Voldemort (2009)': {
            'origin': 'LinkedIn',
            'use_case': 'Read-heavy, low-latency key-value',
            'optimizations': [
                'Client-side routing',
                'Read-your-writes consistency',
                'Pluggable storage engines'
            ]
        }
    }
    
    key_innovations = {
        'Consistent Hashing': {
            'problem_solved': 'Minimal data movement durante scaling',
            'adoption': 'Universal em distributed systems',
            'used_by': ['CDNs', 'Load balancers', 'DHTs', 'Caches']
        },
        
        'Vector Clocks': {
            'problem_solved': 'Causality tracking sem coordenação',
            'adoption': 'Distributedsystems, CRDTs',
            'used_by': ['Riak', 'Akka', 'Orleans']
        },
        
        'Quorum-based Replication': {
            'problem_solved': 'Tunable consistency/availability',
            'adoption': 'Most NoSQL databases',
            'used_by': ['Cassandra', 'Riak', 'Cosmos DB']
        },
        
        'Gossip Protocol': {
            'problem_solved': 'Decentralized failure detection',
            'adoption': 'Cluster management',
            'used_by': ['Cassandra', 'Consul', 'Kubernetes']
        },
        
        'Hinted Handoff': {
            'problem_solved': 'Writes durante temporary failures',
            'adoption': 'High-availability systems',
            'used_by': ['Cassandra', 'Riak']
        }
    }
    
    lessons_learned = {
        'CAP Theorem Practical': {
            'lesson': 'Trade-offs são inevitáveis',
            'insight': 'Choose availability + partition tolerance para e-commerce'
        },
        
        'Simplicity Enables Scale': {
            'lesson': 'Simple data model escala melhor',
            'insight': 'GET/PUT by key → massive throughput'
        },
        
        'Decentralization is Key': {
            'lesson': 'Peer-to-peer > master-slave para availability',
            'insight': 'No single point of failure'
        },
        
        'Application-aware Conflict Resolution': {
            'lesson': 'Sistema não pode resolver todos os conflitos',
            'insight': 'Algumas decisões requerem semântica de aplicação'
        }
    }

# Questões para Estudo sobre Dynamo
class DynamoStudyQuestions:
    """
    Questões para aprofundar compreensão do Dynamo
    """
    
    questions = [
        {
            'Q1': 'Por que Dynamo escolheu eventual consistency sobre strong consistency?',
            'answer_key_points': [
                'CAP theorem: Impossível ter C + A + P',
                'Amazon priorizou Availability (vendas) sobre Consistency',
                'Shopping cart: Melhor cart stale que indisponível',
                'Conflicts resolváveis pela aplicação (merge carts)'
            ]
        },
        
        {
            'Q2': 'Como vector clocks detectam conflitos?',
            'answer_key_points': [
                'Vector clock: Lista de (node, counter) tuples',
                'v1 < v2: Todos counters v1 ≤ v2, pelo menos 1 menor',
                'v1 || v2: Nem v1 < v2 nem v2 < v1 (concurrent!)',
                'Conflitos: Versões concurrent requerem merge'
            ]
        },
        
        {
            'Q3': 'Calcule garantia de consistency com N=3, R=2, W=2',
            'answer_key_points': [
                'R + W = 4 > N = 3',
                'Read overlap: Pelo menos 1 node em R∩W',
                'Garantia: Read sempre vê última write',
                'Trade-off: Pode bloquear se 2 nodes down'
            ]
        },
        
        {
            'Q4': 'Por que consistent hashing é superior a modulo hashing?',
            'answer_key_points': [
                'Modulo: hash(key) % N → Adicionar node move TODOS dados',
                'Consistent: Apenas 1/N dos dados movem',
                'Dynamo: Virtual nodes para balanceamento fino',
                'Resultado: Scaling incremental sem downtime'
            ]
        }
    ]

### 2. Document Stores

**Conceito:**
Armazenam documentos semi-estruturados (JSON, BSON, XML), com capacidade de query em campos internos.

**Características:**
- **Flexibilidade**: Documentos podem ter estruturas diferentes
- **Nested Data**: Suporta objetos e arrays aninhados
- **Indexação**: Índices em qualquer campo, incluindo nested
- **Queries**: Rich query language (quase SQL-like)

**MongoDB / Amazon DocumentDB:**

```python
from pymongo import MongoClient

client = MongoClient('mongodb://docdb-cluster.amazonaws.com:27017/')
db = client.ecommerce

class ProductCatalog:
    """
    Catálogo de produtos com estrutura flexível
    """
    def insert_product(self, product_data):
        """
        Insert produto com nested data
        """
        product = {
            'name': 'Smart TV 55"',
            'price': 899.99,
            'category': 'Electronics',
            'brand': 'Samsung',
            
            # Nested object
            'specifications': {
                'screen_size': '55 inches',
                'resolution': '4K UHD',
                'smart_features': ['Netflix', 'Prime Video', 'YouTube'],
                'connectivity': ['HDMI', 'USB', 'Wi-Fi', 'Bluetooth']
            },
            
            # Array of nested objects
            'reviews': [
                {
                    'user': 'John Doe',
                    'rating': 5,
                    'comment': 'Excellent TV!',
                    'date': '2024-01-10'
                },
                {
                    'user': 'Jane Smith',
                    'rating': 4,
                    'comment': 'Good quality, fast delivery',
                    'date': '2024-01-12'
                }
            ],
            
            # Geospatial data
            'warehouse_location': {
                'type': 'Point',
                'coordinates': [-73.935242, 40.730610]  # Long, Lat
            },
            
            'stock': 45,
            'created_at': datetime.now()
        }
        
        result = db.products.insert_one(product)
        return result.inserted_id
    
    def query_products(self):
        """
        Rich queries em nested fields
        """
        # Query 1: Produtos com Netflix
        netflix_products = db.products.find({
            'specifications.smart_features': 'Netflix'
        })
        
        # Query 2: Produtos com rating alto
        high_rated = db.products.find({
            'reviews.rating': {'$gte': 4}
        })
        
        # Query 3: Produtos baratos com boa avaliação
        good_deals = db.products.find({
            'price': {'$lt': 500},
            'reviews': {
                '$elemMatch': {
                    'rating': {'$gte': 4}
                }
            }
        })
        
        # Query 4: Geospatial - produtos perto de uma localização
        nearby_products = db.products.find({
            'warehouse_location': {
                '$near': {
                    '$geometry': {
                        'type': 'Point',
                        'coordinates': [-74.0060, 40.7128]  # NYC
                    },
                    '$maxDistance': 50000  # 50 km
                }
            }
        })
        
        # Query 5: Aggregation pipeline
        category_stats = db.products.aggregate([
            {
                '$group': {
                    '_id': '$category',
                    'avg_price': {'$avg': '$price'},
                    'total_stock': {'$sum': '$stock'},
                    'count': {'$sum': 1}
                }
            },
            {
                '$sort': {'avg_price': -1}
            }
        ])
        
        return list(category_stats)
    
    def create_indexes(self):
        """
        Criar índices para performance
        """
        # Index simples
        db.products.create_index('category')
        
        # Compound index
        db.products.create_index([
            ('category', 1),
            ('price', -1)
        ])
        
        # Text index para full-text search
        db.products.create_index([
            ('name', 'text'),
            ('specifications.smart_features', 'text')
        ])
        
        # Geospatial index
        db.products.create_index([
            ('warehouse_location', '2dsphere')
        ])
    
    def full_text_search(self, query):
        """
        Full-text search
        """
        results = db.products.find({
            '$text': {'$search': query}
        }, {
            'score': {'$meta': 'textScore'}
        }).sort([
            ('score', {'$meta': 'textScore'})
        ])
        
        return list(results)

# Use Cases Ideais:
# - Content Management Systems
# - E-commerce product catalogs
# - User profiles
# - Mobile app backends
# - Real-time analytics

# Performance:
# - Simple queries: 5-20ms
# - Aggregation pipelines: 50-500ms (depending on data size)
# - Full-text search: 10-100ms
```

### 3. Column-Family Stores

**Conceito:**
Organizam dados em column families (grupos de colunas), otimizados para queries que acessam múltiplas linhas mas poucas colunas.

**Características:**
- **Wide-Column**: Cada row pode ter milhares/milhões de colunas
- **Sparse Data**: Colunas não definidas não ocupam espaço
- **Scalability**: Particionamento por row key, replicação automática
- **Write-Optimized**: LSM trees para writes rápidos

**Apache Cassandra / Amazon Keyspaces:**

```python
from cassandra.cluster import Cluster

cluster = Cluster(['cassandra.us-east-1.amazonaws.com'])
session = cluster.connect('iot_data')

class IoTTimeSeriesStore:
    """
    Armazenamento de dados de sensores IoT
    """
    def create_schema(self):
        """
        Schema otimizado para time-series
        """
        session.execute("""
            CREATE TABLE IF NOT EXISTS sensor_data (
                sensor_id text,
                year int,
                month int,
                timestamp timestamp,
                temperature decimal,
                humidity decimal,
                pressure decimal,
                battery_level decimal,
                PRIMARY KEY ((sensor_id, year, month), timestamp)
            ) WITH CLUSTERING ORDER BY (timestamp DESC)
        """)
        # Partition key: (sensor_id, year, month)
        # Clustering key: timestamp (ordenação)
        # Benefício: Queries por sensor + time range são muito eficientes
    
    def insert_sensor_reading(self, sensor_id, reading):
        """
        Insert reading de sensor
        """
        timestamp = datetime.now()
        
        session.execute("""
            INSERT INTO sensor_data (
                sensor_id, year, month, timestamp,
                temperature, humidity, pressure, battery_level
            ) VALUES (%s, %s, %s, %s, %s, %s, %s, %s)
        """, (
            sensor_id,
            timestamp.year,
            timestamp.month,
            timestamp,
            reading['temperature'],
            reading['humidity'],
            reading['pressure'],
            reading['battery_level']
        ))
        # Performance: 1-5ms (write-optimized)
        # Throughput: 100,000s writes/second
    
    def query_sensor_range(self, sensor_id, start_date, end_date):
        """
        Query readings de um sensor em time range
        """
        results = session.execute("""
            SELECT * FROM sensor_data
            WHERE sensor_id = %s
              AND year = %s
              AND month = %s
              AND timestamp >= %s
              AND timestamp <= %s
            ORDER BY timestamp DESC
        """, (
            sensor_id,
            start_date.year,
            start_date.month,
            start_date,
            end_date
        ))
        
        return list(results)
        # Performance: 10-50ms para milhões de rows
        # Eficiente porque lê de uma partition apenas
    
    def get_latest_readings(self, sensor_id, limit=100):
        """
        Últimas N leituras de um sensor
        """
        results = session.execute("""
            SELECT * FROM sensor_data
            WHERE sensor_id = %s
              AND year = %s
              AND month = %s
            ORDER BY timestamp DESC
            LIMIT %s
        """, (
            sensor_id,
            datetime.now().year,
            datetime.now().month,
            limit
        ))
        
        return list(results)
        # Performance: 5-15ms
        # Clustering order otimiza queries DESC

# Wide Column Example - User Activity Tracking:
class UserActivityStore:
    """
    Track user activities com colunas dinâmicas
    """
    def create_activity_table(self):
        session.execute("""
            CREATE TABLE IF NOT EXISTS user_activities (
                user_id text,
                activity_date date,
                activity_time timestamp,
                activity_type text,
                details map<text, text>,  # Dynamic columns!
                PRIMARY KEY ((user_id, activity_date), activity_time)
            )
        """)
    
    def log_activity(self, user_id, activity_type, details):
        """
        Log user activity com metadata dinâmica
        """
        now = datetime.now()
        
        session.execute("""
            INSERT INTO user_activities (
                user_id, activity_date, activity_time,
                activity_type, details
            ) VALUES (%s, %s, %s, %s, %s)
        """, (
            user_id,
            now.date(),
            now,
            activity_type,
            details  # Pode ter qualquer key-value pairs
        ))
    
    # Exemplo de uso:
    # log_activity('user123', 'page_view', {
    #     'page': '/products/laptop',
    #     'referrer': 'google',
    #     'device': 'mobile',
    #     'session_id': 'abc123'
    # })
    #
    # log_activity('user123', 'purchase', {
    #     'order_id': 'order456',
    #     'amount': '299.99',
    #     'payment_method': 'credit_card',
    #     'shipping_address': '123 Main St'
    # })
    # 
    # Diferentes tipos de activities têm diferentes details
    # Column-family store permite essa flexibilidade

# Use Cases Ideais:
# - Time-series data (IoT, logs, metrics)
# - High write throughput applications
# - Event logging
# - Messaging systems
# - Recommendation engines

# Performance Characteristics:
# - Writes: 1-5ms, 100K+ ops/second per node
# - Reads (by partition): 10-50ms
# - Reads (cross-partition): Lento, evitar
# - Scalability: Linear (adicionar nodes = proporcional increase capacity)
```

### 4. Graph Databases

**Conceito:**
Otimizados para armazenar e query relacionamentos entre entidades (grafos).

**Características:**
- **Nodes**: Entidades (users, products, places)
- **Edges**: Relacionamentos (follows, bought, located_in)
- **Properties**: Atributos em nodes e edges
- **Traversals**: Queries navegam relacionamentos eficientemente

**Amazon Neptune (Gremlin API):**

```python
from gremlin_python.driver import client
from gremlin_python.process.graph_traversal import __

neptune_client = client.Client(
    'wss://my-cluster.neptune.amazonaws.com:8182/gremlin',
    'g'
)

class SocialNetworkGraph:
    """
    Rede social com relacionamentos complexos
    """
    def add_user(self, user_id, name, age, city):
        """
        Adicionar usuário como node
        """
        query = """
            g.addV('user')
             .property('user_id', user_id)
             .property('name', name)
             .property('age', age)
             .property('city', city)
        """
        neptune_client.submit(query, {
            'user_id': user_id,
            'name': name,
            'age': age,
            'city': city
        })
    
    def add_friendship(self, user1_id, user2_id, since_date):
        """
        Criar relacionamento de amizade (edge)
        """
        query = """
            g.V().has('user', 'user_id', user1_id)
             .addE('friends_with')
             .to(g.V().has('user', 'user_id', user2_id))
             .property('since', since_date)
        """
        neptune_client.submit(query, {
            'user1_id': user1_id,
            'user2_id': user2_id,
            'since_date': since_date
        })
    
    def find_mutual_friends(self, user1_id, user2_id):
        """
        Encontrar amigos em comum
        """
        query = """
            g.V().has('user', 'user_id', user1_id)
             .out('friends_with')
             .where(
                __.in('friends_with')
                  .has('user_id', user2_id)
             )
             .values('name')
        """
        result = neptune_client.submit(query, {
            'user1_id': user1_id,
            'user2_id': user2_id
        })
        
        return result.all().result()
        # Performance: 10-100ms
        # Eficiente mesmo com milhões de usuários
        # SQL equivalente seria muito complexo e lento
    
    def recommend_friends(self, user_id, limit=10):
        """
        Recomendação de amigos (friends of friends)
        """
        query = """
            g.V().has('user', 'user_id', user_id)
             .out('friends_with')  # Meus amigos
             .aggregate('friends')
             .out('friends_with')  # Amigos dos meus amigos
             .where(
                __.not(within('friends'))  # Não são meus amigos ainda
             )
             .where(
                __.not(has('user_id', user_id))  # Não sou eu
             )
             .groupCount()  # Contar conexões em comum
             .order(local).by(values, desc)
             .limit(local, limit)
             .select(keys)
             .values('name', 'city')
        """
        result = neptune_client.submit(query, {
            'user_id': user_id,
            'limit': limit
        })
        
        return result.all().result()
        # Performance: 50-500ms
        # Sophisticada traversal em grafo
        # Resultado: Top 10 pessoas com mais amigos em comum
    
    def shortest_path(self, user1_id, user2_id):
        """
        Caminho mais curto entre dois usuários (graus de separação)
        """
        query = """
            g.V().has('user', 'user_id', user1_id)
             .repeat(
                out('friends_with').simplePath()
             )
             .until(
                has('user_id', user2_id)
             )
             .path()
             .by('name')
             .limit(1)
        """
        result = neptune_client.submit(query, {
            'user1_id': user1_id,
            'user2_id': user2_id
        })
        
        path = result.all().result()[0]
        return path
        # Performance: 100-1000ms depending on distance
        # Encontra caminho: User A → Friend1 → Friend2 → User B
        # "Seis graus de separação"

# Complex Graph Use Case - E-commerce Recommendations:
class ProductRecommendationGraph:
    """
    Grafo de produtos, usuários, categorias, compras
    """
    def create_product_graph(self):
        # Nodes: User, Product, Category
        # Edges: PURCHASED, VIEWED, BELONGS_TO, SIMILAR_TO
        pass
    
    def personalized_recommendations(self, user_id, limit=5):
        """
        Recomendações personalizadas baseadas em grafo
        """
        query = """
            g.V().has('user', 'user_id', user_id)
             .out('purchased')  # Produtos que comprei
             .aggregate('bought')
             .out('similar_to')  # Produtos similares
             .where(
                __.not(within('bought'))  # Que ainda não comprei
             )
             .dedup()
             .order().by(
                __.in('purchased').count(), desc  # Mais populares
             )
             .limit(limit)
             .values('name', 'price', 'rating')
        """
        result = neptune_client.submit(query, {
            'user_id': user_id,
            'limit': limit
        })
        
        return result.all().result()
        # Algoritmo:
        # 1. Encontrar produtos que usuário comprou
        # 2. Encontrar produtos similares
        # 3. Filtrar os já comprados
        # 4. Ordenar por popularidade
        # 5. Return top N
        
        # Performance: 100-500ms
        # vs SQL: Seria 5-10 JOINs complexos, 5-30 segundos

# Use Cases Ideais:
# - Social networks
# - Recommendation engines
# - Fraud detection
# - Network topology
# - Knowledge graphs

# Performance vs SQL:
# Graph traversal (Neptune): 50-500ms
# Equivalent JOINs (SQL): 5-60 seconds
# Speedup: 10-1000x para queries de relacionamentos
```

### Comparação de Modelos de Dados

| Modelo | Best Use Case | Query Flexibility | Write Performance | Read Performance |
|--------|---------------|-------------------|-------------------|------------------|
| Key-Value | Session, cache | Baixa (só por key) | Muito Alta (O(1)) | Muito Alta (O(1)) |
| Document | CMS, catalogs | Alta (rich queries) | Alta | Alta |
| Column-Family | Time-series, IoT | Média (por partition) | Muito Alta | Alta (single partition) |
| Graph | Social, recommendations | Muito Alta (relationships) | Média | Muito Alta (traversals) |

---

## Scalability (Escalabilidade)

### Horizontal Scaling (Scale-Out)

Bancos NoSQL são projetados para escalabilidade horizontal desde o início.

**Partitioning/Sharding Automático:**

```python
class AutoShardingSystem:
    """
    Como NoSQL databases automaticamente particionam dados
    """
    def __init__(self, num_nodes=10):
        self.nodes = [Node(i) for i in range(num_nodes)]
        self.hash_ring = ConsistentHashRing(self.nodes)
    
    def write(self, key, value):
        """
        Write é automaticamente roteado para node correto
        """
        # Hash da key determina node
        node = self.hash_ring.get_node(key)
        
        # Write no node primary
        node.write(key, value)
        
        # Replicate para nodes secundários (RF=3)
        replicas = self.hash_ring.get_replicas(key, count=3)
        for replica in replicas[1:]:  # Skip primary
            replica.async_write(key, value)
        
        # Transparente para aplicação!
        # Não precisa saber qual node tem os dados
    
    def read(self, key):
        """
        Read também é roteado automaticamente
        """
        node = self.hash_ring.get_node(key)
        return node.read(key)
    
    def add_node(self, new_node):
        """
        Adicionar node ao cluster (scaling out)
        """
        # 1. Adicionar ao hash ring
        self.hash_ring.add_node(new_node)
        
        # 2. Rebalanceamento automático
        # Apenas ~1/N dos dados movem (N = total nodes)
        keys_to_move = self.calculate_keys_to_move(new_node)
        
        for key in keys_to_move:
            # Move data from old node to new node
            value = self.hash_ring.get_old_node(key).read(key)
            new_node.write(key, value)
        
        # 3. Update routing table
        # 4. Background verification
        
        # Total time: Minutes to hours (depending on data size)
        # vs SQL sharding: Days to weeks of planning + downtime

# Real-world example - DynamoDB Auto-Scaling:
class DynamoDBAutoScaling:
    """
    DynamoDB escala automaticamente baseado em carga
    """
    table_config = {
        'TableName': 'Products',
        'BillingMode': 'PAY_PER_REQUEST',  # On-demand scaling
        
        # Ou provisioned com auto-scaling:
        'ProvisionedThroughput': {
            'ReadCapacityUnits': 1000,
            'WriteCapacityUnits': 1000
        },
        'AutoScalingSettings': {
            'MinCapacity': 1000,
            'MaxCapacity': 100000,  # Até 100x scaling!
            'TargetUtilization': 70  # Escala quando > 70% utilization
        }
    }
    
    # DynamoDB internamente:
    # - Monitora load em cada partition
    # - Split partitions quando necessário
    # - Adiciona storage nodes automaticamente
    # - Rebalanceia dados transparentemente
    
    # Application não precisa saber de nada disso!
    # Apenas envia requests, DynamoDB escala conforme necessário
    
    # Capacidade máxima:
    # - 20M requests/second
    # - 100+ TB de dados
    # - Centenas de partitions
    # - Zero downtime durante scaling

# Cassandra Multi-Datacenter Replication:
class CassandraMultiDC:
    """
    Cassandra escala globalmente com múltiplos datacenters
    """
    cluster_config = {
        'cluster_name': 'GlobalEcommerce',
        'datacenters': {
            'US-EAST': {
                'nodes': 50,
                'replication_factor': 3
            },
            'EU-WEST': {
                'nodes': 30,
                'replication_factor': 3
            },
            'ASIA-PACIFIC': {
                'nodes': 20,
                'replication_factor': 3
            }
        },
        'total_nodes': 100,
        'total_capacity': '500 TB'
    }
    
    def write_with_local_quorum(self, key, value):
        """
        Write com LOCAL_QUORUM: Fast, no cross-DC latency
        """
        session.execute(
            "INSERT INTO products (id, name, price) VALUES (%s, %s, %s)",
            (key, value['name'], value['price']),
            consistency_level=ConsistencyLevel.LOCAL_QUORUM
        )
        # Write em quorum do DC local apenas
        # Background: Async replication para outros DCs
        # Latency: 5-15ms (dentro do DC)
        # vs ALL: 150-300ms (cross-DC round-trip)
    
    def read_from_nearest_dc(self, key):
        """
        Read do DC mais próximo
        """
        # Client automaticamente conecta ao DC mais próximo
        # Users na Europa leem de EU-WEST
        # Users na Ásia leem de ASIA-PACIFIC
        # Latency: 10-50ms (low latency)
        # vs Single DC: 150-300ms para usuários distantes
        
        result = session.execute(
            "SELECT * FROM products WHERE id = %s",
            (key,),
            consistency_level=ConsistencyLevel.LOCAL_ONE
        )
        return result.one()
    
    # Benefits:
    # - Global low latency
    # - Disaster recovery (DC fail = outros DCs continuam)
    # - Compliance (data residency em cada região)
    # - Linear scalability (adicionar DC = adicionar capacity)

# Scalability Metrics - Real Numbers:
scalability_comparison = {
    'Single PostgreSQL Server': {
        'max_throughput': '10,000 QPS',
        'max_data_size': '10 TB (practical)',
        'scaling': 'Vertical only'
    },
    'PostgreSQL Sharded (Manual)': {
        'max_throughput': '100,000 QPS',
        'max_data_size': '100+ TB',
        'scaling': 'Complex, manual'
    },
    'DynamoDB': {
        'max_throughput': '20,000,000 QPS',
        'max_data_size': '100+ TB per table',
        'scaling': 'Automatic, seamless'
    },
    'Cassandra Cluster': {
        'max_throughput': '1,000,000+ QPS per cluster',
        'max_data_size': 'Petabytes',
        'scaling': 'Linear, automatic'
    }
}
```

### Load Balancing e Request Routing

```python
class NoSQLLoadBalancing:
    """
    Como requests são distribuídos em cluster NoSQL
    """
    def client_side_routing(self):
        """
        Client-side routing (Cassandra, DynamoDB)
        """
        # Client library tem knowledge do cluster topology
        # Calcula partition key hash
        # Roteia request diretamente para node correto
        # Benefício: Sem hop intermediário, latência mínima
        
        from cassandra.cluster import Cluster
        
        cluster = Cluster(['node1', 'node2', 'node3'])
        session = cluster.connect('keyspace1')
        
        # Client library automaticamente:
        # 1. Descobre todos os nodes
        # 2. Mantém connection pool para cada node
        # 3. Calcula qual node tem data
        # 4. Envia request diretamente
        
        session.execute("INSERT INTO users (id, name) VALUES (%s, %s)", (123, 'John'))
        # Client sabe que user 123 está no node2
        # Envia diretamente para node2
        # Latency: 1 network hop
    
    def proxy_based_routing(self):
        """
        Proxy-based routing (MongoDB, Redis Cluster)
        """
        # Proxy (mongos, redis-cluster-proxy) roteia requests
        # Client conecta ao proxy, não aos shards diretamente
        
        # Benefits:
        # - Client mais simples (não precisa conhecer topology)
        # - Topology changes transparentes
        
        # Trade-off:
        # - 1 hop adicional (proxy)
        # - Proxy pode ser bottleneck
        
        from pymongo import MongoClient
        
        # Conecta ao mongos (query router)
        client = MongoClient('mongos://router1:27017')
        db = client.mydb
        
        # mongos automaticamente roteia para shard correto
        db.users.insert_one({'_id': 123, 'name': 'John'})
        # Latency: 2 network hops (client → mongos → shard)
    
    def coordinator_pattern(self):
        """
        Coordinator pattern (DynamoDB)
        """
        # Qualquer node pode ser coordinator
        # Coordinator:
        # 1. Recebe request
        # 2. Determina nodes responsáveis
        # 3. Envia sub-requests em paralelo
        # 4. Agrega results
        # 5. Retorna ao client
        
        # Example: Read com QUORUM consistency
        # Coordinator envia reads para 3 replicas
        # Espera 2 responses (quorum de 3)
        # Retorna resultado mais recente
        
        # Benefit: Load distribution (any node can coordinate)
        # Trade-off: Coordination overhead
```

---

## High Availability and Fault Tolerance (Alta Disponibilidade e Tolerância a Falhas)

### Replication Strategies

**Multi-Master Replication (Cassandra):**

```python
class MultiMasterReplication:
    """
    Todos os nodes aceitam writes (masterless)
    """
    def write_with_quorum(self, key, value):
        """
        Write em quorum de replicas
        """
        # Configuração: RF=3 (3 replicas)
        replicas = self.hash_ring.get_replicas(key, count=3)
        
        # Envia write para todas as 3 replicas em paralelo
        responses = []
        for replica in replicas:
            future = replica.async_write(key, value, timestamp=now())
            responses.append(future)
        
        # Espera QUORUM (2 de 3) responderem
        success_count = wait_for_count(responses, count=2, timeout=1000ms)
        
        if success_count >= 2:
            return "OK"  # Write successful
        else:
            raise WriteTimeoutException()
        
        # Benefícios:
        # - No single point of failure
        # - Writes sempre disponíveis (se quorum alcançável)
        # - Linear scalability
        
        # Terceira replica (que não respondeu a tempo):
        # - Background hinted handoff entrega write depois
        # - Eventual consistency garantida
    
    def read_with_quorum(self, key):
        """
        Read com quorum para consistency
        """
        replicas = self.hash_ring.get_replicas(key, count=3)
        
        # Envia read para todas as 3 replicas
        responses = []
        for replica in replicas:
            future = replica.async_read(key)
            responses.append(future)
        
        # Espera QUORUM (2 de 3)
        results = wait_for_count(responses, count=2, timeout=500ms)
        
        # Read repair: Se valores diferentes, usa timestamp
        latest_value = max(results, key=lambda r: r.timestamp)
        
        # Async: Atualiza replica com valor antigo (read repair)
        for result in results:
            if result.value != latest_value.value:
                result.replica.async_write(key, latest_value)
        
        return latest_value
    
    # Tunable Consistency:
    consistency_levels = {
        'ONE': 'Fastest, eventual consistency',
        'QUORUM': 'Balance between consistency and performance',
        'ALL': 'Strongest consistency, slowest',
        'LOCAL_QUORUM': 'Quorum within local datacenter'
    }
    
    # Escolha baseada em requirements:
    # - User profile: ONE/QUORUM (eventual OK)
    # - Financial transaction: QUORUM/ALL (consistency crítica)
    # - Analytics: ONE (performance > consistency)

# DynamoDB Global Tables:
class DynamoDBGlobalTables:
    """
    Multi-region, multi-master replication
    """
    global_table_config = {
        'TableName': 'Users',
        'Regions': ['us-east-1', 'eu-west-1', 'ap-southeast-1'],
        'ReplicationType': 'Multi-Master',
        'ConflictResolution': 'Last Writer Wins'
    }
    
    def write_to_local_region(self, user_id, user_data):
        """
        Write na região local
        """
        # US user escreve em us-east-1
        dynamodb_us.put_item(
            TableName='Users',
            Item={
                'user_id': user_id,
                **user_data
            }
        )
        # Write latency: 5-15ms (local region)
        
        # Background: DynamoDB replica para outras regiões
        # - EU replica: ~100ms after
        # - APAC replica: ~200ms after
        # Eventual consistency global
    
    def read_from_nearest_region(self, user_id):
        """
        Read da região mais próxima
        """
        # EU user lê de eu-west-1
        response = dynamodb_eu.get_item(
            TableName='Users',
            Key={'user_id': user_id}
        )
        # Read latency: 5-15ms (local region)
        # vs cross-region: 100-300ms
        
        return response['Item']
    
    def handle_conflicts(self):
        """
        Conflitos são resolvidos automaticamente
        """
        # Scenario: User atualiza profile simultaneamente em US e EU
        #
        # T0: US write: {name: 'John Smith', email: 'john@us.com'}
        # T0: EU write: {name: 'John Doe', email: 'john@eu.com'}
        #
        # Conflict Resolution (Last Writer Wins):
        # - Usa timestamp para determinar vencedor
        # - Write mais recente prevalece
        # - Ambas as regiões convergem para mesmo valor
        #
        # Alternativa: Custom Conflict Resolution
        # - Application-defined merge logic
        # - Exemplo: Merge arrays, sum counters, etc.
        pass
    
    # Benefits:
    # - Global low latency (users acessam região próxima)
    # - Disaster recovery (região fail = outras continuam)
    # - 99.999% availability SLA
    # - Zero downtime maintenance

# MongoDB Replica Sets:
class MongoDBReplicaSet:
    """
    Primary-Secondary replication com automatic failover
    """
    replica_set_config = {
        'name': 'rs0',
        'members': [
            {'host': 'mongo1:27017', 'priority': 2},  # Preferred primary
            {'host': 'mongo2:27017', 'priority': 1},  # Secondary
            {'host': 'mongo3:27017', 'priority': 1}   # Secondary
        ]
    }
    
    def write_with_majority(self, doc):
        """
        Write com write concern majority
        """
        db.users.insert_one(
            doc,
            write_concern=WriteConcern(w='majority')
        )
        # Write vai para primary
        # Aguarda replicação para maioria (2 de 3)
        # Garante durabilidade mesmo se primary falhar
        # Latency: 10-30ms (replication overhead)
    
    def automatic_failover(self):
        """
        Automatic failover quando primary falha
        """
        # 1. Primary crashes
        # 2. Secondaries detectam failure (heartbeat timeout: 10s)
        # 3. Election: Secondary com maior priority vira primary
        # 4. Clients automaticamente reconectam ao novo primary
        # 5. Total downtime: 10-30 segundos
        
        # During failover:
        # - Reads: Podem continuar de secondaries
        # - Writes: Bloqueados até election completar
        
        # Client code: Retry logic
        from pymongo.errors import AutoReconnect
        
        def write_with_retry(doc, max_retries=3):
            for attempt in range(max_retries):
                try:
                    db.users.insert_one(doc)
                    return "OK"
                except AutoReconnect:
                    if attempt == max_retries - 1:
                        raise
                    time.sleep(1)  # Wait for failover
```

### Data Durability

```python
class DataDurability:
    """
    Garantias de durabilidade em NoSQL
    """
    def cassandra_durability(self):
        """
        Cassandra: Commit log + memtable + SSTable
        """
        # Write path:
        # 1. Write para commit log (append-only file on disk)
        #    - fsync opcional (configurable)
        #    - Durável mesmo com crash
        # 2. Write para memtable (in-memory)
        #    - Sorted structure
        # 3. Async: Flush memtable para SSTable (disk)
        #    - Immutable files
        #    - Compaction merge SSTables
        
        # Durability levels:
        commitlog_sync = {
            'batch': 'Group commits, 10-50ms delay, high throughput',
            'periodic': 'Sync every Nms, tunable safety/performance',
            'immediate': 'fsync every write, max durability, slowest'
        }
        
        # Typical config (balance):
        config = {
            'commitlog_sync': 'periodic',
            'commitlog_sync_period_in_ms': 10000,  # 10s
            'replication_factor': 3,
            'write_consistency': 'QUORUM'
        }
        # Result: RPO < 10 segundos, mesmo com node failure
    
    def dynamodb_durability(self):
        """
        DynamoDB: Automatic 3-AZ replication
        """
        # Write é durável quando replicado para 2+ AZs
        # - Synchronous replication
        # - Write latency: 10-20ms (includes replication)
        # - Durability: 11 noves (99.999999999%)
        
        # Failure scenarios:
        # - 1 AZ fails: Continue operando (other 2 AZs)
        # - 2 AZs fail: Read-only mode
        # - 3 AZs fail: Extremely unlikely (regional disaster)
        
        # Recovery:
        # - Failed AZ: Automatic repair quando volta
        # - Data loss: Virtually impossible
    
    def mongodb_durability(self):
        """
        MongoDB: Journal + Write Concern
        """
        # Write path:
        # 1. Write para journal (on-disk log)
        #    - Group commits every 100ms
        # 2. Write para data files (async)
        #    - Checkpoint every 60s
        
        # Write concern levels:
        write_concerns = {
            'w=1': 'Primary only, fast but not durable',
            'w="majority"': 'Majority of replicas, durable',
            'w=3': 'All 3 replicas, most durable'
        }
        
        # Journal:
        journal_config = {
            'journal': True,
            'journalCommitInterval': 100  # ms
        }
        
        # Crash recovery:
        # - Replay journal desde último checkpoint
        # - Recovery time: Segundos a minutos
        # - No data loss (se write concern=majority)

# Backup and Recovery Strategies:
class BackupStrategies:
    """
    Estratégias de backup para disaster recovery
    """
    def point_in_time_recovery(self):
        """
        DynamoDB Point-in-Time Recovery (PITR)
        """
        # Enable PITR:
        dynamodb.update_continuous_backups(
            TableName='Users',
            PointInTimeRecoverySpecification={'PointInTimeRecoveryEnabled': True}
        )
        
        # Features:
        # - Continuous backups (automatic)
        # - 35 day retention
        # - Restore para qualquer segundo nos últimos 35 dias
        # - RPO: 1 segundo
        # - RTO: Minutes to hours (depending on table size)
        
        # Recovery:
        dynamodb.restore_table_to_point_in_time(
            SourceTableName='Users',
            TargetTableName='Users-Restored',
            RestoreDateTime=datetime(2024, 1, 15, 10, 30, 0)
        )
        # Restore table para estado em 2024-01-15 10:30:00
    
    def snapshot_backups(self):
        """
        Snapshot-based backups (Cassandra, MongoDB)
        """
        # Cassandra snapshot:
        # - nodetool snapshot
        # - Cria hardlinks para SSTables
        # - Instant (sem copy de dados)
        # - Backup cada node independentemente
        
        # MongoDB backup:
        # - mongodump (logical backup)
        # - File system snapshot (physical backup)
        # - Replica set: Backup de secondary (zero impact em primary)
        
        # Best practice:
        # - Daily incremental backups
        # - Weekly full backups
        # - Offsite storage (S3, Glacier)
        # - Test recovery regularly!
        pass
    
    def cross_region_replication(self):
        """
        Replicação cross-region para DR
        """
        # DynamoDB Global Tables: Built-in
        # Cassandra: Multi-DC clusters
        # MongoDB: Cross-region replica sets
        
        # Benefits:
        # - RPO: Seconds (replication lag)
        # - RTO: Seconds to minutes (failover)
        # - Zero data loss (if properly configured)
        
        # Trade-offs:
        # - Cost: 2-3x (multiple regions)
        # - Complexity: Conflict resolution
        # - Consistency: Eventual
```


---

## BASE (Basically Available, Soft state, Eventually consistent)

### Fundamentos do BASE

BASE é um modelo alternativo ao ACID, projetado para sistemas distribuídos que priorizam disponibilidade e performance sobre consistência imediata.

**BASE vs ACID:**

| Propriedade | ACID | BASE |
|-------------|------|------|
| **Atomicity** | Transações all-or-nothing | Operações podem ser parciais |
| **Consistency** | Strong consistency sempre | Eventual consistency |
| **Isolation** | Transações isoladas | Isolamento relaxado |
| **Durability** | Imediato em disco | Eventual em disco |
| **Availability** | Pode degradar durante partições | Prioriza availability |
| **Performance** | Overhead de garantias | Otimizado para throughput |

### Basically Available (Basicamente Disponível)

**Conceito:**
Sistema responde a requisições mesmo durante falhas parciais, possivelmente com dados degradados.

```python
class BasicallyAvailableSystem:
    """
    Sistema que prioriza availability sobre consistency
    """
    def read_with_fallback(self, key):
        """
        Read com múltiplos níveis de fallback
        """
        try:
            # Try primary read (fastest)
            return self.primary_store.get(key)
        except TimeoutException:
            # Primary timeout: Try replica
            try:
                return self.replica_store.get(key)
            except TimeoutException:
                # Replica também timeout: Try cache
                try:
                    cached = self.cache.get(key)
                    if cached:
                        return {
                            'value': cached,
                            'stale': True,  # Pode estar desatualizado
                            'warning': 'Data from cache, may be stale'
                        }
                except:
                    pass
                
                # Último recurso: Retorna erro mas mantém serviço up
                return {
                    'error': 'Service degraded',
                    'value': None,
                    'retry_after': 5
                }
        
        # Sistema NUNCA fica completamente indisponível
        # Sempre retorna ALGO, mesmo que degradado

# Real Example - Amazon Shopping Cart:
class ShoppingCartBASE:
    """
    Shopping cart com BASE properties
    """
    def add_to_cart(self, user_id, item_id, quantity):
        """
        Adicionar item ao carrinho (basically available)
        """
        try:
            # Try write para DynamoDB
            dynamodb.put_item(
                TableName='ShoppingCarts',
                Item={
                    'user_id': user_id,
                    'items': [{'item_id': item_id, 'quantity': quantity}]
                }
            )
            return {'status': 'success'}
        
        except ServiceUnavailableException:
            # DynamoDB temporariamente indisponível
            # Fallback: Salvar em cache local (Redis)
            redis.lpush(
                f'pending_cart_updates:{user_id}',
                json.dumps({'item_id': item_id, 'quantity': quantity})
            )
            
            # Background job: Sync pendentes quando DynamoDB voltar
            
            return {
                'status': 'accepted',
                'message': 'Item adicionado (processando)',
                'basically_available': True
            }
        
        # User experience: Item é adicionado ao carrinho SEMPRE
        # Pode haver pequeno delay para sync, mas serviço nunca falha
        # Trade-off: Eventual consistency > downtime

# Cassandra Hinted Handoff:
class HintedHandoff:
    """
    Mecanismo para manter availability durante node failures
    """
    def write_with_hints(self, key, value):
        """
        Write com hinted handoff
        """
        replicas = get_replicas(key, count=3)
        
        success_count = 0
        for replica in replicas:
            try:
                replica.write(key, value)
                success_count += 1
            except NodeDownException:
                # Node está down, mas não falha o write
                # Store "hint" (pending write) em outro node
                coordinator_node = self.get_coordinator()
                coordinator_node.store_hint(
                    target_node=replica,
                    key=key,
                    value=value
                )
                # Quando replica voltar, hint é entregue
        
        if success_count >= 2:  # Quorum
            return "OK"
        else:
            # Mesmo sem quorum, write pode ser aceito
            # Hints garantem eventual consistency
            return "ACCEPTED"
    
    # Benefit: Sistema continua aceitando writes mesmo com nodes down
    # Trade-off: Temporary inconsistency até hints serem delivered
```

### Soft State (Estado Flexível)

**Conceito:**
Estado do sistema pode mudar ao longo do tempo, mesmo sem input, devido a eventual consistency.

```python
class SoftStateExample:
    """
    Estado que evolui assincronamente
    """
    def view_counter_soft_state(self, page_id):
        """
        View counter com eventual consistency
        """
        # Múltiplos servidores incrementam contador localmente
        # Sync periódico ao banco central
        
        # Server 1:
        local_cache['views'] = local_cache.get('views', 0) + 1
        
        # Server 2:
        local_cache['views'] = local_cache.get('views', 0) + 1
        
        # Server 3:
        local_cache['views'] = local_cache.get('views', 0) + 1
        
        # Background job (every 10s): Flush para DynamoDB
        def flush_views():
            total_views = sum(server.local_cache['views'] for server in servers)
            dynamodb.update_item(
                TableName='PageViews',
                Key={'page_id': page_id},
                UpdateExpression='ADD view_count :increment',
                ExpressionAttributeValues={':increment': total_views}
            )
            # Reset local caches
            for server in servers:
                server.local_cache['views'] = 0
        
        # Result: View count é "soft"
        # - Aumenta gradualmente conforme flushes
        # - Não é instantaneamente preciso
        # - Converge para valor correto eventualmente
        
        # Trade-off:
        # - Write throughput: 1000x melhor (local increments)
        # - Consistency: Eventual (10s lag)
        # - Acceptable para metrics não-críticas

# Cassandra Counters (Soft State):
class CassandraCounters:
    """
    Distributed counters com soft state
    """
    def increment_counter(self, counter_name):
        """
        Increment distributed counter
        """
        session.execute(
            "UPDATE counters SET count = count + 1 WHERE name = %s",
            (counter_name,)
        )
        # Counter update é "commutative"
        # Ordem não importa: +1 +1 = 2 (independente de ordem)
        # Conflitos são resolvidos automaticamente (soma)
        
        # Durante partição de rede:
        # - Node A vê counter = 100, incrementa para 101
        # - Node B vê counter = 100, incrementa para 101
        # - Após partition heal: Conflict resolution
        # - Result: 102 (soma dos incrementos)
        # - Soft state: Valor muda para resolver conflicts

# Shopping Cart Merge (Soft State):
class ShoppingCartMerge:
    """
    Merging de shopping carts conflitantes
    """
    def merge_carts_after_partition(self, user_id):
        """
        User adiciona items em diferentes devices durante network partition
        """
        # Device 1 (mobile): Adiciona item A
        cart_mobile = ['item_A']
        
        # Device 2 (desktop): Adiciona item B
        cart_desktop = ['item_B']
        
        # Após network partition heal:
        # Conflito: Duas versões do cart
        
        # Merge strategy: Union (adicionar tudo)
        merged_cart = set(cart_mobile) | set(cart_desktop)
        # Result: ['item_A', 'item_B']
        
        # Soft state: Cart mudou sem user input
        # Evoluiu para estado consistente através de merge
        
        # Alternative strategies:
        # - Timestamp-based (último write vence)
        # - Custom logic (application-specific)
        # - CRDTs (Conflict-free Replicated Data Types)
        
        return list(merged_cart)

# CRDTs (Conflict-free Replicated Data Types):
class GCounter:
    """
    Grow-only counter (CRDT)
    Estado sempre converge sem coordination
    """
    def __init__(self, node_id, num_nodes):
        self.node_id = node_id
        self.counts = {i: 0 for i in range(num_nodes)}
    
    def increment(self):
        """Increment counter no node local"""
        self.counts[self.node_id] += 1
    
    def value(self):
        """Valor atual do counter"""
        return sum(self.counts.values())
    
    def merge(self, other):
        """Merge com outro contador"""
        for node_id in self.counts:
            self.counts[node_id] = max(
                self.counts[node_id],
                other.counts.get(node_id, 0)
            )
    
    # Propriedade CRDT:
    # - Commutative: merge(A, B) = merge(B, A)
    # - Associative: merge(merge(A, B), C) = merge(A, merge(B, C))
    # - Idempotent: merge(A, A) = A
    # 
    # Result: Sempre converge para mesmo estado
    # Sem coordenação, sem conflitos!

# Example usage:
# Node 1: increment 5 vezes → counts = {0: 5, 1: 0, 2: 0}
# Node 2: increment 3 vezes → counts = {0: 0, 1: 3, 2: 0}
# Node 3: increment 7 vezes → counts = {0: 0, 1: 0, 2: 7}
# 
# After merge: counts = {0: 5, 1: 3, 2: 7}
# Total: 15 (correto!)
```

### Eventually Consistent (Eventualmente Consistente)

**Conceito:**
Após cessarem updates, todos os acessos eventualmente retornarão o último valor atualizado.

```python
class EventualConsistency:
    """
    Demonstração de eventual consistency
    """
    def write_and_read_scenario(self):
        """
        Timeline de eventual consistency
        """
        # T0: Write em região US
        dynamodb_us.put_item(
            TableName='Users',
            Item={'user_id': '123', 'name': 'John Updated'}
        )
        # Write commit: T0 + 5ms
        
        # T0 + 10ms: Replicação para EU inicia
        # T0 + 110ms: Replicação para EU completa
        # T0 + 150ms: Replicação para APAC inicia  
        # T0 + 350ms: Replicação para APAC completa
        
        # Read scenarios:
        
        # T0 + 20ms: Read de US
        user = dynamodb_us.get_item(TableName='Users', Key={'user_id': '123'})
        assert user['name'] == 'John Updated'  # ✓ Consistente
        
        # T0 + 50ms: Read de EU
        user = dynamodb_eu.get_item(TableName='Users', Key={'user_id': '123'})
        assert user['name'] == 'John Updated'  # ❌ Pode ainda ser valor antigo!
        
        # T0 + 200ms: Read de EU (após replication)
        user = dynamodb_eu.get_item(TableName='Users', Key={'user_id': '123'})
        assert user['name'] == 'John Updated'  # ✓ Consistente agora
        
        # T0 + 400ms: Read de APAC (após replication)
        user = dynamodb_apac.get_item(TableName='Users', Key={'user_id': '123'})
        assert user['name'] == 'John Updated'  # ✓ Todas regiões consistentes
        
        # "Eventually": Replication lag típico 100-500ms
        # Após esse período, todas as réplicas convergem

# Staleness Bounds:
class StalenessBounds:
    """
    Limites de inconsistência temporal
    """
    staleness_metrics = {
        'DynamoDB (same region)': {
            'typical': '10-50ms',
            'p99': '100-200ms',
            'max': '1 second (rare)'
        },
        'DynamoDB Global Tables': {
            'typical': '100-500ms',
            'p99': '1-2 seconds',
            'max': '5 seconds (rare)'
        },
        'Cassandra (LOCAL_QUORUM)': {
            'typical': '10-100ms',
            'p99': '500ms',
            'max': '1 second'
        },
        'Cassandra (cross-DC)': {
            'typical': '100-1000ms',
            'p99': '5 seconds',
            'max': '30 seconds (during issues)'
        }
    }
    
    def acceptable_staleness(self, use_case):
        """
        Staleness aceitável por use case
        """
        acceptable_by_use_case = {
            'social_media_feed': '5-30 seconds',  # OK
            'comments': '1-5 seconds',  # OK
            'likes_counter': '10-60 seconds',  # OK
            'user_profile': '1-10 seconds',  # OK
            'shopping_cart': '100-1000ms',  # Marginal
            'inventory': '10-100ms',  # Critical
            'payment': '0ms (strong consistency)',  # Critical
            'authentication': '0ms (strong consistency)'  # Critical
        }
        
        return acceptable_by_use_case.get(use_case)

# Read-after-Write Consistency:
class ReadAfterWriteConsistency:
    """
    Garantir que user vê seus próprios writes
    """
    def implement_read_your_writes(self, user_id):
        """
        Padrão para garantir read-your-writes
        """
        # Opção 1: Strong consistency read
        def option1_strong_read():
            # DynamoDB
            response = dynamodb.get_item(
                TableName='Users',
                Key={'user_id': user_id},
                ConsistentRead=True  # Strong consistency
            )
            # Trade-off: 2x latência (10ms vs 5ms)
            return response['Item']
        
        # Opção 2: Read from same node (session affinity)
        def option2_session_affinity():
            # Sticky session: User sempre roteia para mesmo node
            # Node tem suas próprias writes localmente
            session_node = get_node_for_session(user_id)
            return session_node.read(user_id)
            # Benefit: Fast, sem inconsistency para own writes
            # Limitation: Não escala tão bem (sticky sessions)
        
        # Opção 3: Version tracking
        def option3_version_tracking():
            # Client tracks version do último write
            last_write_version = self.write_user(user_id, data)
            
            # Read com version requirement
            user = self.read_user_min_version(user_id, last_write_version)
            # Se replica está atrás, retry ou wait
            return user
        
        # Opção 4: Timestamp-based
        def option4_timestamp():
            # Write retorna timestamp
            write_timestamp = self.write_user(user_id, data)
            
            # Read aceita apenas data mais recente que timestamp
            max_age = datetime.now() - write_timestamp
            if max_age > timedelta(milliseconds=100):
                # Too stale, use strong consistency
                return self.strong_read(user_id)
            else:
                return self.eventual_read(user_id)

# Monotonic Reads:
class MonotonicReads:
    """
    Garantir que reads não "voltam no tempo"
    """
    def ensure_monotonic_reads(self, user_id):
        """
        User não vê dados mais antigos após ver mais novos
        """
        # Problema:
        # T0: Read de replica A → version 5
        # T1: Read de replica B → version 3 (mais antiga!)
        # User vê "voltar no tempo" ❌
        
        # Solução 1: Session affinity (sempre mesma replica)
        session['preferred_replica'] = 'replica_A'
        # Sempre lê de replica_A
        
        # Solução 2: Version tracking
        session['last_seen_version'] = None
        
        def read_monotonic(key):
            data = self.read(key)
            current_version = data['_version']
            
            last_seen = session['last_seen_version']
            
            if last_seen and current_version < last_seen:
                # Stale read detectado!
                # Retry de outra replica ou strong read
                data = self.strong_read(key)
            
            session['last_seen_version'] = data['_version']
            return data

# Convergence Time:
class ConvergenceTime:
    """
    Tempo até eventual consistency convergir
    """
    def calculate_convergence(self):
        """
        Fatores que afetam convergence time
        """
        factors = {
            'replication_lag': '100-500ms (network + processing)',
            'anti_entropy': '1-24 hours (background repair)',
            'hinted_handoff_retry': '3 hours default (Cassandra)',
            'conflict_resolution': 'Immediate (automatic) or Manual'
        }
        
        # Typical convergence:
        # - Normal case: 100-1000ms (replication lag)
        # - Node temporary failure: 3+ hours (hinted handoff)
        # - Network partition: Minutes to hours (depends on duration)
        # - Manual conflict: Days (requires operator intervention)
        
        # Best practices:
        # - Monitor replication lag
        # - Alert on prolonged divergence
        # - Run anti-entropy repairs regularly
        # - Test partition scenarios
        
        return factors
```

### Quando Usar BASE vs ACID

```python
class BaseVsAcidDecision:
    """
    Decision framework para escolher BASE ou ACID
    """
    def choose_model(self, requirements):
        """
        Escolher modelo baseado em requirements
        """
        # Use ACID quando:
        acid_use_cases = [
            'Financial transactions (zero tolerance para inconsistency)',
            'Inventory management (overselling não aceitável)',
            'Order processing (strong consistency crítica)',
            'Authentication/Authorization (security critical)',
            'Healthcare records (regulatory compliance)',
            'Legal documents (audit trail essencial)'
        ]
        
        # Use BASE quando:
        base_use_cases = [
            'Social media feeds (eventual consistency OK)',
            'Analytics/Metrics (approximate values acceptable)',
            'Content delivery (stale data não é problema)',
            'Shopping cart (temporary inconsistency tolerável)',
            'Session storage (availability > consistency)',
            'Caching layers (stale = normal)',
            'IoT data ingestion (volume > consistency)',
            'Logging/Monitoring (availability crítica)'
        ]
        
        # Hybrid approach:
        hybrid_approach = {
            'critical_data': 'ACID (SQL ou NoSQL com transactions)',
            'non_critical_data': 'BASE (NoSQL eventual consistency)',
            'example': {
                'order_table': 'RDS PostgreSQL (ACID)',
                'order_history_cache': 'DynamoDB (BASE)',
                'product_catalog': 'DocumentDB (BASE)',
                'inventory': 'PostgreSQL (ACID)',
                'user_sessions': 'Redis (BASE)',
                'payment_processing': 'PostgreSQL (ACID)'
            }
        }
        
        return {
            'acid_for': acid_use_cases,
            'base_for': base_use_cases,
            'hybrid': hybrid_approach
        }

# Real-world Example - E-commerce:
class EcommerceArchitecture:
    """
    Arquitetura híbrida ACID + BASE
    """
    architecture = {
        'Order Processing': {
            'database': 'Aurora PostgreSQL',
            'model': 'ACID',
            'reason': 'Financial accuracy critical',
            'consistency': 'Strong'
        },
        'Product Catalog': {
            'database': 'DocumentDB',
            'model': 'BASE',
            'reason': 'Flexible schema, high read volume',
            'consistency': 'Eventual (1-5s lag OK)'
        },
        'Shopping Cart': {
            'database': 'DynamoDB',
            'model': 'BASE',
            'reason': 'High availability, low latency',
            'consistency': 'Eventual (100ms lag OK)'
        },
        'Inventory': {
            'database': 'Aurora PostgreSQL',
            'model': 'ACID',
            'reason': 'Prevent overselling',
            'consistency': 'Strong with locks'
        },
        'User Sessions': {
            'database': 'ElastiCache Redis',
            'model': 'BASE',
            'reason': 'Ultra-low latency, temporary data',
            'consistency': 'Eventual or None (cached)'
        },
        'Analytics': {
            'database': 'Redshift',
            'model': 'BASE',
            'reason': 'Batch processing, aggregate data',
            'consistency': 'Eventual (hourly ETL)'
        }
    }
```

---

## Questões para Estudo e Prática

### Questão 1: Design de Sistema - Rede Social

**Pergunta:**
Design uma rede social (similar ao Twitter) que suporte 500 milhões de usuários ativos, 50 milhões de tweets por dia, e timeline feed personalizado para cada usuário. Escolha e justifique o tipo de banco NoSQL e estratégia de consistência.

**Aspectos a Considerar:**
- Modelo de dados (users, tweets, followers, timeline)
- Volume de writes (tweets) vs reads (feeds)
- Latência requirements (feed deve carregar em <100ms)
- Consistência (eventual consistency aceitável para feed?)
- Escalabilidade (como adicionar capacity?)

**Resposta Esperada:**

```
Arquitetura Proposta:

1. User Profiles: DynamoDB
   - Modelo: Key-Value/Document
   - Partition key: user_id
   - Justificativa:
     * Low latency reads (<10ms)
     * Flexible schema (profile pode ter campos variados)
     * Global Tables para usuários globais

2. Tweets: Cassandra
   - Modelo: Wide-Column (time-series)
   - Partition key: (user_id, date)
   - Clustering key: tweet_timestamp DESC
   - Justificativa:
     * High write throughput (50M tweets/day = 578 writes/second)
     * Time-series otimizado (tweets por user por data)
     * Linear scalability

3. Followers Graph: Amazon Neptune
   - Modelo: Graph
   - Nodes: Users
   - Edges: follows
   - Justificativa:
     * Relationship queries (mutual follows, suggestions)
     * Fast traversals
     * Degree of separation queries

4. Timeline Feed: Redis (cache) + DynamoDB
   - Cache: Pre-computed top 100 tweets
   - DynamoDB: Full timeline history
   - Justificativa:
     * Ultra-low latency (<5ms from Redis)
     * Cache hit rate ~95%
     * Fallback to DynamoDB if cache miss

Consistência:
- Tweets: Eventual consistency (1-5s lag aceitável)
- Followers: Eventual consistency (novo follower aparece em segundos)
- Profiles: Strong consistency para own profile, eventual para outros

Performance estimada:
- Tweet creation: 10-30ms
- Timeline read (cached): 2-5ms
- Timeline read (cache miss): 50-100ms
- Follow action: 10-20ms
```

### Questão 2: CAP Theorem Aplicado

**Pergunta:**
Explique como você configuraria um banco Cassandra para três cenários diferentes:
1. Sistema bancário (prioriza consistency)
2. Rede social (prioriza availability)
3. IoT sensors (prioriza partition tolerance + throughput)

**Resposta Esperada:**

```
Cenário 1 - Sistema Bancário (CP):
session.execute(query,
    consistency_level=ConsistencyLevel.QUORUM)  # Writes
session.execute(query,
    consistency_level=ConsistencyLevel.QUORUM)  # Reads

Replication Factor: 3
Write CL: QUORUM (2/3)
Read CL: QUORUM (2/3)
Trade-off: Menor availability durante failures, mas consistency garantida

Cenário 2 - Rede Social (AP):
session.execute(query,
    consistency_level=ConsistencyLevel.ONE)  # Writes
session.execute(query,
    consistency_level=ConsistencyLevel.ONE)  # Reads

Replication Factor: 3
Write CL: ONE (1/3)
Read CL: ONE (1/3)
Trade-off: Máxima availability, eventual consistency

Cenário 3 - IoT Sensors (AP + High Throughput):
session.execute(query,
    consistency_level=ConsistencyLevel.ANY)  # Writes

Replication Factor: 3
Write CL: ANY (hinted handoff OK)
Read CL: LOCAL_ONE
Trade-off: Máximo throughput, eventual consistency, data pode ser temporarily lost
```

### Questão 3: Schema Design

**Pergunta:**
Design o schema de um e-commerce product catalog usando MongoDB. Considere:
- 10M produtos
- Múltiplas categorias
- Reviews/ratings
- Variações (size, color)
- Full-text search
- Faceted search (filtros)

**Resposta Esperada:**

```javascript
// Document structure
{
    "_id": ObjectId("..."),
    "sku": "LAPTOP-001",
    "name": "Gaming Laptop XPS 15",
    "slug": "gaming-laptop-xps-15",
    
    // Denormalized category info
    "category": {
        "id": "electronics",
        "name": "Electronics",
        "path": ["Electronics", "Computers", "Laptops"]
    },
    
    "brand": "Dell",
    "price": 1599.99,
    "currency": "USD",
    
    // Variações como array
    "variations": [
        {
            "sku": "LAPTOP-001-16GB",
            "attributes": {"ram": "16GB", "storage": "512GB SSD"},
            "price": 1599.99,
            "stock": 45
        },
        {
            "sku": "LAPTOP-001-32GB",
            "attributes": {"ram": "32GB", "storage": "1TB SSD"},
            "price": 1999.99,
            "stock": 23
        }
    ],
    
    // Reviews summary (aggregated)
    "reviews_summary": {
        "average_rating": 4.5,
        "count": 234,
        "distribution": {
            "5": 150,
            "4": 60,
            "3": 15,
            "2": 5,
            "1": 4
        }
    },
    
    // Recent reviews (limited to 10)
    "recent_reviews": [
        {
            "user": "john_doe",
            "rating": 5,
            "comment": "Excellent laptop!",
            "date": ISODate("2024-01-15"),
            "helpful_votes": 23
        }
        // ... up to 10
    ],
    
    // Faceted search attributes
    "attributes": {
        "processor": "Intel i9",
        "ram": "16GB-32GB",
        "screen_size": "15.6 inches",
        "graphics": "RTX 4080",
        "weight_kg": 2.3
    },
    
    // Full-text search
    "search_terms": ["gaming", "laptop", "dell", "xps", "15", ...],
    
    "images": ["url1.jpg", "url2.jpg", "url3.jpg"],
    "created_at": ISODate("2024-01-01"),
    "updated_at": ISODate("2024-01-15")
}

// Índices necessários:
db.products.createIndex({"category.id": 1, "price": -1})
db.products.createIndex({"brand": 1, "price": -1})
db.products.createIndex({"search_terms": "text"})
db.products.createIndex({"slug": 1}, {unique: true})
db.products.createIndex({
    "attributes.processor": 1,
    "attributes.ram": 1,
    "attributes.graphics": 1
})

// Trade-offs:
// - Denormalização: Reviews summary e recent reviews duplicados
// - Benefício: 1 query para página de produto completa
// - Full reviews: Collection separada, loaded on demand
// - Variações: Embedded (poucas variações) vs Referenced (muitas)
```

### Questão 4: Partitioning Strategy

**Pergunta:**
Como você particionaria dados de um sistema de mensagens (chat) usando DynamoDB? O sistema tem:
- 100M usuários
- 1B mensagens/dia
- Queries: Listar conversas de um usuário, listar mensagens de uma conversa
- Requirement: <50ms latency

**Resposta Esperada:**

```
Tabela 1: Conversations
Partition Key: user_id
Sort Key: last_message_timestamp (DESC)

Item structure:
{
    "user_id": "user123",
    "conversation_id": "conv456",
    "last_message_timestamp": 1705334400,
    "other_user_id": "user789",
    "other_user_name": "Jane Doe",
    "last_message_preview": "Hey, how are you?",
    "unread_count": 3
}

Justificativa:
- Partition por user_id: Hot partition evitável (users leem próprias conversas)
- Sort por timestamp: Conversas recentes primeiro
- Denormalização: Nome do outro usuário cached

Tabela 2: Messages
Partition Key: conversation_id
Sort Key: timestamp (DESC)

Item structure:
{
    "conversation_id": "conv456",
    "message_id": "msg789",
    "timestamp": 1705334400,
    "sender_id": "user123",
    "sender_name": "John Doe",
    "message": "Hello!",
    "read": false
}

Justificativa:
- Partition por conversation_id: Mensagens de uma conversa juntas
- Sort por timestamp: Ordem cronológica
- Denormalização: sender_name para display

Challenges e Soluções:
1. Hot partition (conversa em grupo muito ativa):
   - Solução: Shard por (conversation_id, date_bucket)
   - Exemplo: (conv456, 2024-01-15) como partition key

2. Large conversations (>400KB por partition):
   - Solução: Time-based partitions
   - PK: (conversation_id, year_month)
   - Mensagens antigas em partitions separadas

3. Read pattern (últimas 50 mensagens):
   - Query: Simples, sort key DESC, limit 50
   - Performance: 10-30ms

4. Write throughput (1B messages/day):
   - 11,574 writes/second
   - DynamoDB: Easily handled
   - Cost optimization: On-demand pricing ou provisioned
```

### Questão 5: Eventual Consistency Problem

**Pergunta:**
Seu sistema usa DynamoDB com eventual consistency. Um usuário reporta que alterou sua foto de perfil mas ainda vê a foto antiga em alguns lugares. Explique:
1. Por que isso acontece
2. Como você diagnosticaria
3. Possíveis soluções

**Resposta Esperada:**

```
1. Por que acontece:
- DynamoDB replica dados entre 3 AZs
- Writes confirmados após 2/3 AZs (quorum)
- Eventual consistent reads podem ler de qualquer replica
- Se ler de replica que ainda não recebeu update → valor antigo
- Replication lag típico: 10-100ms, às vezes mais

2. Diagnóstico:
- Verificar se usando ConsistentRead=False (default)
- Medir replication lag via CloudWatch:
  Metric: ReplicationLatency
- Check timing: Update → Read delay
- Se <100ms: Normal eventual consistency
- Se >1s: Possível problema de replicação

3. Soluções:

Opção A - Strong Consistency Reads:
dynamodb.get_item(
    TableName='Users',
    Key={'user_id': user_id},
    ConsistentRead=True  # Força leitura do leader
)
Trade-off: 2x latência, menor throughput

Opção B - Session Affinity:
- User sticky para mesma replica
- Read-your-writes garantido
- Implementation: Route based em user_id hash

Opção C - Client-side Caching + Versioning:
- Cache foto no client após upload
- Usar versioned URLs: photo_v2.jpg
- Invalidar cache do lado do client
- Eventual consistency não afeta UX

Opção D - Optimistic UI:
- UI atualiza imediatamente após upload (optimistic)
- Não espera confirmação do backend
- Se falhar, rollback UI
- User nunca vê inconsistência

Recomendação para este caso:
- Usar Opção C + D (versioned URLs + optimistic UI)
- Strong consistency não necessária (não é critical data)
- Eventual consistency de 100-500ms é aceitável
- Cost-effective e melhor UX
```

---

## Conclusão

### Resumo dos Conceitos Chave

Neste capítulo, exploramos profundamente os conceitos de armazenamento não relacional (NoSQL), que representam uma mudança fundamental no paradigma de gerenciamento de dados:

**Nonrelational Database Concepts:**
- **Teorema CAP**: Trade-off inevitável entre Consistency, Availability e Partition Tolerance
- **PACELC**: Extensão considerando latência em operações normais
- **Vantagens**: Escalabilidade horizontal, flexibilidade de schema, performance especializada
- **Desvantagens**: Eventual consistency, falta de JOINs, transações limitadas

**Schema Flexibility:**
- **Schema-on-Read**: Validação na leitura vs write-time
- **Padrões**: Polymorphic, Attribute, Extended Reference
- **Validação Opcional**: Quando necessário para dados críticos

**Data Models:**
- **Key-Value**: Simplicidade, O(1) performance (DynamoDB, Redis)
- **Document**: Flexibilidade, rich queries (MongoDB, DocumentDB)
- **Column-Family**: Time-series, write-optimized (Cassandra)
- **Graph**: Relacionamentos, traversals (Neptune)

**Scalability:**
- **Horizontal Scaling**: Auto-sharding, linear scalability
- **Partitioning**: Consistent hashing, rebalanceamento automático
- **Multi-Datacenter**: Replicação global, low latency mundial

**High Availability:**
- **Replication**: Multi-master (Cassandra), Primary-Secondary (MongoDB)
- **Failover**: Automatic, subsegundos a minutos
- **Durability**: 11 noves, cross-AZ replication

**BASE Properties:**
- **Basically Available**: Sistema sempre responde, possivelmente degradado
- **Soft State**: Estado evolui assincronamente
- **Eventually Consistent**: Convergência após cessarem updates

### Decision Framework: SQL vs NoSQL

```python
choose_database = {
    'Use SQL (Relational) quando': [
        'Transações ACID são críticas',
        'Queries complexas com múltiplos JOINs',
        'Schema bem definido e estável',
        'Strong consistency requirement',
        'Reporting e analytics ad-hoc',
        'Equipe familiarizada com SQL',
        'Escalabilidade vertical suficiente'
    ],
    
    'Use NoSQL quando': [
        'Escalabilidade horizontal necessária',
        'Schema flexível ou evolutivo',
        'High throughput writes (>10K/s)',
        'Latência ultra-baixa (<10ms)',
        'Dados não-estruturados ou semi-estruturados',
        'Eventual consistency aceitável',
        'Casos de uso específicos (time-series, graphs, cache)'
    ],
    
    'Use Ambos (Polyglot Persistence)': [
        'Critical data: SQL (orders, payments)',
        'High-volume data: NoSQL (logs, events)',
        'Cache layer: Redis/Memcached',
        'Search: Elasticsearch',
        'Analytics: Redshift/BigQuery',
        'Example: E-commerce usa 5-10 databases diferentes!'
    ]
}
```

### AWS NoSQL Services Summary

| Service | Type | Use Case | Max Throughput | Consistency |
|---------|------|----------|----------------|-------------|
| DynamoDB | Key-Value/Document | Web/mobile apps | 20M req/s | Eventual or Strong |
| DocumentDB | Document | Content management | 100K req/s | Eventual |
| ElastiCache (Redis) | Key-Value | Cache, sessions | 1M req/s | Eventual |
| Keyspaces | Wide-Column | Time-series, IoT | 100K req/s | Eventual or Local Quorum |
| Neptune | Graph | Social, recommendations | 100K req/s | Eventual |
| Timestream | Time-Series | Metrics, IoT | 1M writes/s | Eventual |

### Best Practices

**Design:**
1. **Escolha o modelo certo**: Key-Value, Document, Column, ou Graph baseado em access patterns
2. **Denormalize estrategicamente**: Para performance, mas com awareness de trade-offs
3. **Partition key design**: Evite hot partitions, distribua uniformemente
4. **Secondary indexes**: Minimize (cada index = custo + latência)

**Performance:**
1. **Batch operations**: Use batch writes/reads quando possível
2. **Connection pooling**: Reutilize conexões (overhead de nova conexão é alto)
3. **Caching**: Redis/Memcached para dados frequentemente acessados
4. **Query optimization**: Minimize data transfer, use projection

**Consistency:**
1. **Choose appropriate level**: Nem tudo precisa de strong consistency
2. **Implement retry logic**: Para eventual consistency scenarios
3. **Version tracking**: Para read-your-writes quando necessário
4. **Idempotency**: Writes devem ser idempotentes para retries

**Monitoring:**
1. **Replication lag**: Alert se > threshold aceitável
2. **Throttling**: Monitor rejected requests
3. **Hot partitions**: Identify e redistribuir
4. **Costs**: NoSQL pode ser caro em high-throughput scenarios

### Recursos Adicionais

**Livros:**
- *NoSQL Distilled* - Pramod J. Sadalage & Martin Fowler (2012)
- *Designing Data-Intensive Applications* - Martin Kleppmann (2017)
- *Cassandra: The Definitive Guide* - Jeff Carpenter & Eben Hewitt (2020)
- *MongoDB: The Definitive Guide* - Shannon Bradshaw et al. (2019)

**Papers Acadêmicos:**
- "Dynamo: Amazon's Highly Available Key-value Store" - DeCandia et al. (2007)
- "Bigtable: A Distributed Storage System" - Chang et al. (2006)
- "Cassandra: A Decentralized Structured Storage System" - Lakshman & Malik (2010)
- "CAP Twelve Years Later: How the 'Rules' Have Changed" - Brewer (2012)

**AWS Documentation:**
- DynamoDB Best Practices Guide
- DocumentDB Developer Guide
- Amazon Neptune Best Practices
- AWS Database Blog

**Online Resources:**
- DynamoDB Guide (https://dynamodbguide.com)
- MongoDB University (free courses)
- DataStax Academy (Cassandra)
- AWS re:Invent talks on databases

---

**Fonte:** System Design on AWS - Jayanth Kumar, Mandeep Singh; NoSQL Distilled - Pramod J. Sadalage & Martin Fowler

**Autor do Capítulo:** AWS Learning Repository Contributors

**Última Atualização:** 2024

---

*Este capítulo fornece uma compreensão profunda de bancos de dados não relacionais, seus princípios fundamentais, modelos de dados, estratégias de escalabilidade e o modelo BASE. A combinação de teoria, exemplos práticos da AWS, code samples extensivos e questões de estudo prepara você para projetar sistemas distribuídos modernos que aproveitam o melhor dos bancos NoSQL.*

