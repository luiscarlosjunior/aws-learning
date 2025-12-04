# Amazon Neptune

## O que é Neptune?

Amazon Neptune é um banco de dados de grafos totalmente gerenciado que facilita construir e executar aplicações que trabalham com datasets altamente conectados. Suporta grafos de propriedades (Gremlin) e grafos RDF (SPARQL).

## Modelos de Grafo

### Property Graph (Gremlin)

Grafo com vértices e arestas com propriedades.

**Componentes:**
- **Vertices**: Entidades (pessoas, produtos, lugares)
- **Edges**: Relacionamentos entre vertices
- **Properties**: Atributos de vertices e edges
- **Labels**: Tipos de vertices e edges

**Exemplo:**
```
Vertex(id: 1, label: 'person', properties: {name: 'Alice', age: 30})
  ├─ Edge(label: 'knows', properties: {since: 2020})
  └─> Vertex(id: 2, label: 'person', properties: {name: 'Bob', age: 25})
```

### RDF Graph (SPARQL)

Grafo baseado em triplas (subject-predicate-object).

**Estrutura:**
```
Subject: <http://example.org/alice>
Predicate: <http://example.org/knows>
Object: <http://example.org/bob>
```

## Casos de Uso

### 1. Rede Social

```python
from gremlin_python.driver import client, serializer

# Conectar ao Neptune
neptune_client = client.Client(
    'wss://my-neptune.cluster-xxx.us-east-1.neptune.amazonaws.com:8182/gremlin',
    'g',
    message_serializer=serializer.GraphSONSerializersV2d0()
)

# Criar usuários
neptune_client.submit("""
    g.addV('user').property('id', 'user1').property('name', 'Alice').property('email', 'alice@example.com')
""").all().result()

neptune_client.submit("""
    g.addV('user').property('id', 'user2').property('name', 'Bob').property('email', 'bob@example.com')
""").all().result()

# Criar relacionamento
neptune_client.submit("""
    g.V().has('user', 'id', 'user1')
     .addE('follows').to(g.V().has('user', 'id', 'user2'))
     .property('since', '2024-01-01')
""").all().result()

# Query: Quem Alice segue?
result = neptune_client.submit("""
    g.V().has('user', 'id', 'user1')
     .out('follows')
     .values('name')
""").all().result()
print(f"Alice follows: {result}")

# Query: Amigos em comum
common_friends = neptune_client.submit("""
    g.V().has('user', 'id', 'user1')
     .out('follows')
     .where(__.in('follows').has('id', 'user3'))
     .values('name')
""").all().result()
```

### 2. Sistema de Recomendação

```python
def recommend_products(user_id, limit=5):
    """
    Recomenda produtos baseado em:
    - Produtos comprados por usuários similares
    - Produtos relacionados aos já comprados
    """
    query = f"""
    g.V().has('user', 'id', '{user_id}')
     .out('purchased').aggregate('bought')
     .in('purchased')
     .out('purchased')
     .where(without('bought'))
     .groupCount()
     .order(local).by(values, desc)
     .limit(local, {limit})
     .unfold()
     .select(keys)
     .values('name')
    """
    
    result = neptune_client.submit(query).all().result()
    return result

recommendations = recommend_products('user1', limit=10)
print(f"Recommended products: {recommendations}")
```

### 3. Fraud Detection

```python
def detect_suspicious_patterns(account_id):
    """
    Detecta padrões suspeitos:
    - Múltiplos níveis de transferência
    - Valores altos
    - Timeframe curto
    """
    query = f"""
    g.V().has('account', 'id', '{account_id}')
     .repeat(
         out('transferred_to')
         .simplePath()
     ).times(3).emit()
     .path()
     .by('id')
    """
    
    paths = neptune_client.submit(query).all().result()
    
    # Analisa caminhos
    suspicious = []
    for path in paths:
        if len(path) >= 3:  # 3+ níveis
            suspicious.append({
                'path': path,
                'risk_level': 'HIGH'
            })
    
    return suspicious

# Detectar fraude
fraud_cases = detect_suspicious_patterns('account123')
print(f"Suspicious patterns: {fraud_cases}")
```

### 4. Knowledge Graph

```python
# SPARQL para RDF graph
from SPARQLWrapper import SPARQLWrapper, JSON

sparql = SPARQLWrapper("https://my-neptune.cluster-xxx.us-east-1.neptune.amazonaws.com:8182/sparql")

# Insert triplas
sparql.setQuery("""
    PREFIX ex: <http://example.org/>
    PREFIX foaf: <http://xmlns.com/foaf/0.1/>
    
    INSERT DATA {
        ex:alice foaf:name "Alice" .
        ex:alice foaf:knows ex:bob .
        ex:alice ex:worksFor ex:company1 .
        ex:company1 ex:name "Tech Corp" .
    }
""")
sparql.method = 'POST'
sparql.query()

# Query conhecimento
sparql.setQuery("""
    PREFIX ex: <http://example.org/>
    PREFIX foaf: <http://xmlns.com/foaf/0.1/>
    
    SELECT ?person ?company WHERE {
        ?person foaf:name "Alice" .
        ?person ex:worksFor ?company .
        ?company ex:name ?companyName .
    }
""")
sparql.setReturnFormat(JSON)
results = sparql.query().convert()

for result in results["results"]["bindings"]:
    print(f"Person: {result['person']['value']}")
    print(f"Company: {result['company']['value']}")
```

### 5. Route Optimization

```python
def find_shortest_path(start_city, end_city):
    """Encontra menor caminho entre cidades"""
    query = f"""
    g.V().has('city', 'name', '{start_city}')
     .repeat(out('connects_to').simplePath())
     .until(has('name', '{end_city}'))
     .path()
     .by('name')
     .by('distance')
     .limit(1)
    """
    
    path = neptune_client.submit(query).all().result()
    return path[0] if path else None

def calculate_total_distance(path):
    """Calcula distância total"""
    total = sum(edge['distance'] for edge in path if isinstance(edge, dict))
    return total

# Encontrar rota
route = find_shortest_path('New York', 'Los Angeles')
if route:
    print(f"Route: {route}")
    print(f"Total distance: {calculate_total_distance(route)} km")
```

## Recursos Avançados

### Neptune ML

Machine Learning sobre grafos.

**Casos de uso:**
- Node classification
- Link prediction  
- Graph classification

**Exemplo:**
```python
# Preparar dados para ML
export_query = """
g.V().has('user').as('u')
 .project('user_id', 'features', 'label')
 .by(values('id'))
 .by(valueMap('age', 'activity_score'))
 .by(values('churn'))
"""

# Neptune ML treina modelos usando SageMaker
# Predição inline em queries
prediction_query = """
g.V().has('user', 'id', 'user1')
 .property(single, 'churn_prediction', ml.prediction('churn_model'))
"""
```

### Full-Text Search

Integração com OpenSearch.

```python
# Query com full-text search
query = """
g.withSideEffect('Neptune#fts.endpoint', 'search-domain.us-east-1.es.amazonaws.com')
 .withSideEffect('Neptune#fts.queryType', 'query_string')
 .call('Neptune#fts')
 .params('Alice')
 .unfold()
"""
```

### Streams

Capture mudanças em tempo real.

```python
import boto3

neptune_streams = boto3.client('neptune-streams')

# Ler stream
response = neptune_streams.get_records(
    streamArn='arn:aws:neptune:us-east-1:123456789012:cluster:my-cluster'
)

for record in response['records']:
    print(f"Operation: {record['op']}")
    print(f"Data: {record['data']}")
```

## Performance

### Gremlin Best Practices

```python
# ❌ Mau: Full scan
bad_query = "g.V().has('name', 'Alice')"

# ✅ Bom: Usar index
good_query = "g.V().has('user', 'id', 'user1')"

# ❌ Mau: Multiple round trips
for user_id in user_ids:
    result = g.V().has('user', 'id', user_id).next()

# ✅ Bom: Batch query
query = f"g.V().has('user', 'id', within({user_ids}))"

# ✅ Usar .profile() para análise
profiled = neptune_client.submit("g.V().has('user', 'id', 'user1').profile()").all().result()
```

### Read Replicas

```bash
# Criar read replica
aws neptune create-db-instance \
    --db-instance-identifier neptune-replica-1 \
    --db-instance-class db.r5.large \
    --engine neptune \
    --db-cluster-identifier my-neptune-cluster
```

## Backup e Recovery

```bash
# Criar snapshot
aws neptune create-db-cluster-snapshot \
    --db-cluster-identifier my-neptune \
    --db-cluster-snapshot-identifier my-snapshot

# Restore
aws neptune restore-db-cluster-from-snapshot \
    --db-cluster-identifier my-neptune-restored \
    --snapshot-identifier my-snapshot \
    --engine neptune
```

## Migração

### De Neo4j

```python
# Export de Neo4j
from neo4j import GraphDatabase

neo4j_driver = GraphDatabase.driver("bolt://localhost:7687", auth=("neo4j", "password"))

with neo4j_driver.session() as session:
    # Export nodes
    nodes = session.run("MATCH (n) RETURN n")
    
    # Import para Neptune
    for node in nodes:
        neptune_client.submit(f"""
            g.addV('{node['n'].labels[0]}')
             .property('id', '{node['n']['id']}')
        """).all().result()
```

## Segurança

```bash
# VPC only, encryption at rest
aws neptune create-db-cluster \
    --db-cluster-identifier my-neptune \
    --engine neptune \
    --vpc-security-group-ids sg-12345678 \
    --db-subnet-group-name my-subnet-group \
    --storage-encrypted \
    --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678
```

### IAM Authentication

```python
from botocore.auth import SigV4Auth
from botocore.awsrequest import AWSRequest
import boto3

session = boto3.Session()
credentials = session.get_credentials()

request = AWSRequest(
    method='GET',
    url='wss://my-neptune.cluster-xxx.us-east-1.neptune.amazonaws.com:8182/gremlin'
)

SigV4Auth(credentials, 'neptune-db', 'us-east-1').add_auth(request)

# Use signed headers in connection
```

## Melhores Práticas

1. **Use índices** apropriados para queries frequentes
2. **Batch operations** para bulk loading
3. **Read replicas** para read scaling
4. **Monitor performance** via CloudWatch
5. **Regular backups** automatizados
6. **Connection pooling** essencial
7. **Profile queries** para otimização

## Limitações

- Max graph size: Baseado em instance storage
- Max vertices/edges: Bilhões (depende de instance)
- Max properties per vertex/edge: Ilimitado (prático)
- Query timeout: Configurável

## Perguntas e Respostas

### P: Quando usar Neptune vs RDS?
**R:** Neptune para dados altamente conectados (social networks, fraud detection, recommendations). RDS para dados tabulares/relacionais. Neptune otimizado para traversal de grafos, RDS para queries SQL e joins.

### P: Gremlin vs SPARQL?
**R:** Gremlin para property graphs (mais flexível, queries imperativas). SPARQL para RDF graphs (semantic web, ontologies, queries declarativas). Escolha baseado em modelo de dados e use case.

### P: Como escalar Neptune?
**R:** Vertical: aumentar instance class. Horizontal: adicionar read replicas (até 15). Storage auto-scale. Para write scaling, partition graph logicamente em múltiplos clusters.

### P: Neptune vs DynamoDB para relações?
**R:** Neptune para queries de traversal profundo (friends of friends of friends). DynamoDB para relações simples (1-2 níveis). Neptune tem melhor expressividade para grafos complexos.

### P: Como migrar de Neo4j?
**R:** Export via Cypher queries ou CSV, transform para formato Neptune (GraphML ou CSV com headers específicos), import usando Neptune Bulk Loader. Test queries e performance.

### P: Performance de queries?
**R:** Use .profile() para análise. Optimize: 1) Use índices corretos, 2) Limit traversal depth, 3) Filter cedo no traversal, 4) Use .has() steps eficientemente, 5) Avoid full scans.

### P: Backup strategy?
**R:** Automated backups (1-35 dias). Manual snapshots para long-term. Copy para outras regiões (DR). Enable streams para CDC. Test restore procedures.

### P: Como debuggar queries lentas?
**R:** Use .profile() step, check execution plan. Monitor CloudWatch metrics (Gremlin requests, latency). Enable query logging. Optimize traversal patterns e índices.

### P: Neptune vs Amazon Keyspaces?
**R:** Neptune: graph database, traversals. Keyspaces: wide-column (Cassandra-compatible), time-series, IoT. Diferentes modelos de dados e use cases.

### P: Custo de Neptune?
**R:** Instance-based pricing (db.r5.large ~$0.348/hr). Storage separado. Read replicas charged separately. Mais caro que DynamoDB mas worth it para graph workloads.

## Recursos de Aprendizado

- [Neptune Documentation](https://docs.aws.amazon.com/neptune/)
- [Gremlin Documentation](https://tinkerpop.apache.org/docs/current/reference/)
- [SPARQL Tutorial](https://www.w3.org/TR/sparql11-query/)
- [Neptune Best Practices](https://docs.aws.amazon.com/neptune/latest/userguide/best-practices.html)
- [AWS re:Invent Neptune Sessions](https://www.youtube.com/results?search_query=reinvent+neptune)
