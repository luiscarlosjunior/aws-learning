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

