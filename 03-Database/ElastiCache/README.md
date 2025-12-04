# Amazon ElastiCache

## O que é ElastiCache?

Amazon ElastiCache é um serviço de cache in-memory totalmente gerenciado que suporta Redis e Memcached. Oferece microsegundos de latência e alta taxa de transferência para aplicações que exigem performance extrema.

## Engines

### Redis

Cache avançado com persistência e estruturas de dados complexas.

**Características:**
- Estruturas de dados: Strings, Lists, Sets, Sorted Sets, Hashes, Bitmaps, HyperLogLogs, Streams
- Persistência (RDB snapshots, AOF)
- Replicação automática
- Failover automático
- Pub/Sub messaging
- Lua scripting
- Transactions (MULTI/EXEC)
- TTL por chave

**Modos:**
- **Cluster Mode Disabled**: Single shard, até 5 read replicas
- **Cluster Mode Enabled**: Multi-shard (até 500 nodes), horizontal scaling

### Memcached

Cache simples e distribuído para uso geral.

**Características:**
- Multi-threaded
- Simple key-value store
- Auto-discovery
- Horizontal scaling (até 20 nodes)
- No persistence
- No replication

**Quando usar:**
- Cache simples
- Multi-threaded workloads
- Horizontal scaling necessário

## Arquitetura

### Redis Cluster Mode Disabled

```
┌─────────────┐
│  Primary    │ ──writes──▶ Application
│   Node      │ ◀──reads───
└─────┬───────┘
      │ replication
      ├──▶ ┌─────────────┐
      │    │  Replica 1  │ ◀──reads─── Application
      │    └─────────────┘
      └──▶ ┌─────────────┐
           │  Replica 2  │ ◀──reads─── Application
           └─────────────┘
```

### Redis Cluster Mode Enabled

```
┌──────────────────────────────────────┐
│           Redis Cluster               │
│                                       │
│  Shard 1        Shard 2      Shard 3 │
│  ┌──────┐      ┌──────┐     ┌──────┐│
│  │Primary│      │Primary│     │Primary││
│  └───┬──┘      └───┬──┘     └───┬──┘│
│      │             │             │    │
│  ┌───▼──┐      ┌──▼───┐     ┌──▼───┐│
│  │Replica│     │Replica│     │Replica││
│  └──────┘      └──────┘     └──────┘│
└──────────────────────────────────────┘
    Hash slots distributed: 0-16383
```

### Memcached

```
┌──────────────────────────────────────┐
│         Memcached Cluster             │
│                                       │
│  ┌────────┐  ┌────────┐  ┌────────┐ │
│  │ Node 1 │  │ Node 2 │  │ Node 3 │ │
│  └────────┘  └────────┘  └────────┘ │
│                                       │
│  Consistent hashing distributes keys  │
└──────────────────────────────────────┘
```

## Node Types

### Redis

**R7g (Graviton3 - Recomendado):**
- cache.r7g.large: 2 vCPU, 13.07 GB
- cache.r7g.xlarge: 4 vCPU, 26.32 GB
- cache.r7g.2xlarge: 8 vCPU, 52.82 GB
- cache.r7g.16xlarge: 64 vCPU, 427.4 GB

**R6g (Graviton2):**
- Similar pricing, slightly older generation

**M7g/M6g (Balanced):**
- Lower memory-to-CPU ratio
- Cost-effective para workloads não memory-intensive

### Memcached

**R7g nodes** também disponíveis para Memcached

## Casos de Uso

### 1. Database Caching

```python
import redis
import pymysql
import json

# Conexões
redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

def get_user(user_id):
    """Get user com cache"""
    cache_key = f"user:{user_id}"
    
    # Try cache first
    cached = redis_client.get(cache_key)
    if cached:
        print("Cache hit!")
        return json.loads(cached)
    
    # Cache miss - query database
    print("Cache miss - querying database")
    db_conn = pymysql.connect(host='db-host', user='user', password='pass', database='myapp')
    with db_conn.cursor(pymysql.cursors.DictCursor) as cursor:
        cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
        user = cursor.fetchone()
    
    # Store in cache (TTL 1 hour)
    if user:
        redis_client.setex(cache_key, 3600, json.dumps(user))
    
    return user

# Cache-aside pattern
user = get_user(123)
print(user)
```

### 2. Session Store

```python
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)

# Configure Redis session store
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_REDIS'] = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379
)
Session(app)

@app.route('/login', methods=['POST'])
def login():
    # Store session data in Redis
    session['user_id'] = 123
    session['username'] = 'john'
    return 'Logged in'

@app.route('/profile')
def profile():
    # Read from Redis
    user_id = session.get('user_id')
    if not user_id:
        return 'Not logged in', 401
    return f"User: {session['username']}"
```

### 3. Leaderboard (Sorted Sets)

```python
import redis

redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

def update_score(user_id, score):
    """Atualiza score do usuário"""
    redis_client.zadd('leaderboard', {user_id: score})

def get_top_players(n=10):
    """Top N jogadores"""
    return redis_client.zrevrange('leaderboard', 0, n-1, withscores=True)

def get_user_rank(user_id):
    """Ranking do usuário"""
    rank = redis_client.zrevrank('leaderboard', user_id)
    return rank + 1 if rank is not None else None

def get_user_score(user_id):
    """Score do usuário"""
    return redis_client.zscore('leaderboard', user_id)

# Uso
update_score('user:123', 1500)
update_score('user:456', 2000)
update_score('user:789', 1800)

print("Top 10:", get_top_players(10))
print("User rank:", get_user_rank('user:456'))
print("User score:", get_user_score('user:456'))
```

### 4. Rate Limiting

```python
import redis
from datetime import datetime

redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379
)

def check_rate_limit(user_id, max_requests=100, window_seconds=3600):
    """
    Rate limit: max_requests por window_seconds
    """
    key = f"rate_limit:{user_id}"
    current_time = datetime.now().timestamp()
    
    # Remove requests antigos (fora da janela)
    redis_client.zremrangebyscore(key, 0, current_time - window_seconds)
    
    # Conta requests na janela atual
    request_count = redis_client.zcard(key)
    
    if request_count < max_requests:
        # Adiciona request atual
        redis_client.zadd(key, {current_time: current_time})
        redis_client.expire(key, window_seconds)
        return True
    else:
        return False

# Uso
user_id = "user:123"
if check_rate_limit(user_id):
    print("Request allowed")
    # Process request
else:
    print("Rate limit exceeded")
    # Return 429 Too Many Requests
```

### 5. Pub/Sub Messaging

```python
import redis
import threading
import time

redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

def subscriber():
    """Subscriber thread"""
    pubsub = redis_client.pubsub()
    pubsub.subscribe('notifications')
    
    print("Subscriber waiting for messages...")
    for message in pubsub.listen():
        if message['type'] == 'message':
            print(f"Received: {message['data']}")

def publisher():
    """Publisher thread"""
    time.sleep(1)  # Wait for subscriber
    for i in range(5):
        message = f"Notification {i+1}"
        redis_client.publish('notifications', message)
        print(f"Published: {message}")
        time.sleep(1)

# Run subscriber e publisher
sub_thread = threading.Thread(target=subscriber, daemon=True)
pub_thread = threading.Thread(target=publisher)

sub_thread.start()
pub_thread.start()
pub_thread.join()
```

### 6. Distributed Locking

```python
import redis
import time
import uuid

redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

class RedisLock:
    def __init__(self, key, timeout=10):
        self.key = f"lock:{key}"
        self.timeout = timeout
        self.identifier = str(uuid.uuid4())
    
    def acquire(self, blocking=True, timeout=None):
        """Adquire lock"""
        end_time = time.time() + timeout if timeout else None
        
        while True:
            # Try to set lock (NX = only if not exists)
            if redis_client.set(self.key, self.identifier, nx=True, ex=self.timeout):
                return True
            
            if not blocking or (end_time and time.time() > end_time):
                return False
            
            time.sleep(0.001)  # Short sleep before retry
    
    def release(self):
        """Release lock"""
        # Lua script para atomic check e delete
        lua_script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        redis_client.eval(lua_script, 1, self.key, self.identifier)

# Uso
lock = RedisLock("critical_section", timeout=30)

if lock.acquire(timeout=5):
    try:
        print("Lock acquired - executing critical section")
        # Critical section code
        time.sleep(2)
    finally:
        lock.release()
        print("Lock released")
else:
    print("Could not acquire lock")
```

### 7. Caching com TTL Strategies

```python
import redis
import json
from functools import wraps
import hashlib

redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

def cache_result(ttl=3600, key_prefix='cache'):
    """Decorator para cache de função"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Generate cache key
            key_parts = [key_prefix, func.__name__]
            key_parts.extend(str(arg) for arg in args)
            key_parts.extend(f"{k}:{v}" for k, v in sorted(kwargs.items()))
            cache_key = hashlib.md5(':'.join(key_parts).encode()).hexdigest()
            
            # Try cache
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)
            
            # Execute function
            result = func(*args, **kwargs)
            
            # Store result
            redis_client.setex(cache_key, ttl, json.dumps(result))
            
            return result
        return wrapper
    return decorator

# Uso
@cache_result(ttl=300)
def expensive_computation(x, y):
    """Função cara que será cached"""
    import time
    time.sleep(2)  # Simulate expensive operation
    return x + y

# Primeira chamada: lenta (cache miss)
result = expensive_computation(10, 20)  # Takes 2 seconds

# Segunda chamada: rápida (cache hit)
result = expensive_computation(10, 20)  # Instant
```

### 8. Redis Streams para Event Processing

```python
import redis
import json
import time

redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    decode_responses=True
)

def produce_events(stream_name='events'):
    """Produtor de eventos"""
    for i in range(10):
        event = {
            'event_id': i,
            'event_type': 'user_action',
            'user_id': f'user:{i}',
            'timestamp': time.time()
        }
        
        # Add to stream
        event_id = redis_client.xadd(stream_name, event)
        print(f"Produced event: {event_id}")
        time.sleep(0.5)

def consume_events(stream_name='events', group='processors', consumer='consumer1'):
    """Consumidor de eventos"""
    # Criar consumer group
    try:
        redis_client.xgroup_create(stream_name, group, id='0', mkstream=True)
    except redis.ResponseError:
        pass  # Group já existe
    
    print(f"Consumer {consumer} waiting for events...")
    
    while True:
        # Read from stream
        messages = redis_client.xreadgroup(
            group, consumer,
            {stream_name: '>'},
            count=10,
            block=5000
        )
        
        if messages:
            for stream, events in messages:
                for event_id, event_data in events:
                    print(f"Processing: {event_id} - {event_data}")
                    
                    # Process event
                    # ... your processing logic ...
                    
                    # Acknowledge
                    redis_client.xack(stream_name, group, event_id)

# Uso em threads separadas
import threading
producer_thread = threading.Thread(target=produce_events)
consumer_thread = threading.Thread(target=consume_events, daemon=True)

consumer_thread.start()
time.sleep(1)
producer_thread.start()
producer_thread.join()
```

## Backup e Recovery

### Redis

**Automated Backups:**
- Daily snapshots
- Retenção: 1-35 dias
- Manual snapshots: retenção ilimitada

**Recovery:**
```bash
# Restore de snapshot
aws elasticache create-cache-cluster \
    --cache-cluster-id my-redis-restored \
    --snapshot-name my-snapshot-20240101
```

### Memcached

**Não suporta persistência** - dados perdidos se cluster reiniciar

## Segurança

### Network Isolation

```bash
# Redis em VPC
aws elasticache create-cache-cluster \
    --cache-cluster-id my-redis \
    --engine redis \
    --cache-node-type cache.r7g.large \
    --num-cache-nodes 1 \
    --cache-subnet-group-name my-subnet-group \
    --security-group-ids sg-12345678
```

### Encryption

**At Rest:**
- Redis: KMS encryption
- Memcached: Não suportado

**In Transit:**
- TLS/SSL connections
- Redis AUTH token

**Exemplo com encryption:**
```python
import redis

redis_client = redis.Redis(
    host='my-redis.abc123.cache.amazonaws.com',
    port=6379,
    password='my-auth-token',
    ssl=True,
    ssl_cert_reqs='required',
    decode_responses=True
)
```

### Redis AUTH

```bash
# Criar cluster com AUTH
aws elasticache create-replication-group \
    --replication-group-id my-redis \
    --replication-group-description "Redis with AUTH" \
    --engine redis \
    --cache-node-type cache.r7g.large \
    --num-cache-clusters 2 \
    --auth-token "MySecureToken123456"
```

## Monitoramento

### CloudWatch Metrics

**Redis:**
- CPUUtilization
- DatabaseMemoryUsagePercentage
- CurrConnections
- Evictions
- CacheHits / CacheMisses
- ReplicationLag

**Memcached:**
- CPUUtilization
- BytesUsedForCacheItems
- CurrConnections
- Evictions
- CmdGet / CmdSet

### Alarms

```bash
# Alarm para evictions alto
aws cloudwatch put-metric-alarm \
    --alarm-name redis-high-evictions \
    --alarm-description "Redis evictions > 1000" \
    --metric-name Evictions \
    --namespace AWS/ElastiCache \
    --statistic Sum \
    --period 300 \
    --threshold 1000 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=CacheClusterId,Value=my-redis
```

## Melhores Práticas

### Redis
1. **Use Cluster Mode** para > 90 GB data
2. **Enable Multi-AZ** para production
3. **Connection pooling** (reuse connections)
4. **Set appropriate TTLs** para evitar memory pressure
5. **Monitor evictions** (idealmente zero)
6. **Use pipelining** para múltiplas operações
7. **Lazy deletion** para large keys

### Memcached
1. **Add nodes horizontally** para escalar
2. **Connection pooling** essencial
3. **Consistent hashing** para minimize redistribution
4. **Monitor hit rate** (> 80% ideal)

### Segurança
1. **VPC deployment** apenas
2. **Enable encryption** in transit e at rest (Redis)
3. **Redis AUTH token** enabled
4. **Security groups** com least privilege
5. **Disable unnecessary commands** (Redis)

### Performance
1. **Choose appropriate node type** (memory vs compute)
2. **Monitor memory usage** (< 75% ideal)
3. **Use read replicas** para read-heavy workloads
4. **Connection pooling** (max connections limited)
5. **Avoid large keys** (> 100 KB)

## Limitações

**Redis:**
- Max key size: 512 MB
- Max value size: 512 MB
- Max connections: Depende do node type
- Cluster mode: Max 500 nodes

**Memcached:**
- Max key size: 250 bytes
- Max value size: 1 MB
- Max nodes: 20 per cluster

## Perguntas e Respostas

### P: Redis vs Memcached?
**R:** Redis: mais features (estruturas de dados, persistence, pub/sub, replication), single-threaded. Memcached: simples, multi-threaded, sem persistence. Use Redis para maioria dos casos. Memcached apenas se precisa multi-threading ou cache muito simples.

### P: O que é eviction?
**R:** Quando memória cheia, cache remove keys antigos para liberar espaço. Monitore evictions - alto número indica memória insuficiente. Soluções: aumentar memory, ajustar TTLs, adicionar nodes (cluster), otimizar data sizes.

### P: Cache-aside vs Write-through?
**R:** Cache-aside (lazy loading): app lê cache, se miss, lê DB e popula cache. Write-through: app escreve em cache e DB simultaneamente. Cache-aside mais comum, write-through garante cache sempre atualizado mas mais writes.

### P: Como evitar thundering herd?
**R:** Quando cache expira, múltiplas requests batem DB simultaneamente. Soluções: 1) Lock durante cache refresh, 2) Stagger TTLs, 3) Probabilistic early expiration, 4) Cache warming, 5) Use cache null results (short TTL).

### P: Redis persistence RDB vs AOF?
**R:** RDB: snapshot periódico, menor overhead, pode perder últimos writes. AOF: log de writes, durabilidade melhor, arquivo maior. Recomendado: ambos habilitados. ElastiCache usa RDB para backups.

### P: Como escalar Redis?
**R:** Vertical: aumentar node type. Horizontal: Cluster Mode Enabled (shard data), ou add read replicas (read scaling). Cluster Mode para > 90 GB ou high throughput. Read replicas para read-heavy.

### P: O que é Redis Cluster Mode?
**R:** Sharding nativo: data distribuído em múltiplos shards (até 500 nodes). Cada shard tem primary + replicas. Hash slots (16384) distribuídos entre shards. Use para large datasets ou high throughput.

### P: Como funciona failover no Redis?
**R:** ElastiCache detecta failure do primary, promove replica automaticamente, atualiza DNS. Multi-AZ replication group recomendado. Failover leva ~30-60 segundos. Aplicação deve retry connections.

### P: Posso usar Redis como database primário?
**R:** Possível com persistence, mas não recomendado. Redis otimizado para cache/in-memory workloads. Para durabilidade ACID, use RDS/DynamoDB. Redis bom para: cache, sessions, queues, temporary data.

### P: Como debuggar slow commands?
**R:** Use SLOWLOG: `SLOWLOG GET 10`. Identifica queries lentas. Causas comuns: KEYS * (use SCAN), large collections sem paging, blocking operations. Monitor EngineeCPUUtilization no CloudWatch.

### P: Connection pooling é necessário?
**R:** Sim! Redis tem limite de connections. Criar/fechar connections cara. Use libraries com pooling (redis-py ConnectionPool, node-redis). Configure max connections baseado em node type.

### P: Como atualizar Redis version?
**R:** ElastiCache permite in-place upgrade minor versions. Major versions requerem backup/restore. Test em non-prod primeiro. Considere downtime durante upgrade ou use blue/green deployment.

### P: Redis Cluster vs Replication Group?
**R:** Replication Group: single shard (primary + replicas), até 5 replicas, vertical scaling. Redis Cluster: multi-shard, horizontal scaling, até 500 nodes. Use Cluster para > 90 GB ou sharding necessário.

### P: Como implementar cache invalidation?
**R:** Estratégias: 1) TTL-based (simples), 2) Event-driven (pub/sub, streams, SNS), 3) Version keys, 4) DEL specific keys, 5) FLUSHDB (drástico). Combine TTL + events para melhor controle.

### P: ElastiCache vs DAX?
**R:** DAX: específico para DynamoDB, microsecond latency, write-through. ElastiCache: geral purpose, qualquer datasource, mais flexible. Use DAX se já usa DynamoDB intensivamente. ElastiCache para outros casos.

### P: Como migrar para ElastiCache?
**R:** 1) Self-hosted Redis: backup RDB/AOF, restore no ElastiCache, ou use replicação. 2) Configure app connection string. 3) Test performance. 4) Cutover DNS/endpoint. 5) Monitor metrics.

### P: Posso acessar Redis de Lambda?
**R:** Sim, mas Lambda deve estar em VPC (mesma do Redis). Configure NAT Gateway para Lambda acessar internet. Use connection reuse (global variable) para evitar connection overhead.

### P: Como calcular tamanho necessário?
**R:** Estime: num_keys × avg_key_size + num_keys × avg_value_size. Add 20% overhead para Redis metadata. Monitor DatabaseMemoryUsagePercentage, mantenha < 75%. Start small, scale up baseado em métricas.

### P: TTL curto vs longo?
**R:** Curto: dados mais atualizados, mais cache misses, mais load no DB. Longo: menos misses, pode servir stale data. Balance baseado em tolerance para staleness. Use diferentes TTLs por tipo de data.

### P: Como fazer backup do Redis?
**R:** ElastiCache automated backups (daily). Manual snapshots quando necessário. Copy snapshots para outras regiões (DR). Test restore procedures. Snapshots não impactam performance (tirados de replica).

## Recursos de Aprendizado

- [ElastiCache Documentation](https://docs.aws.amazon.com/elasticache/)
- [Redis Documentation](https://redis.io/documentation)
- [ElastiCache Best Practices](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/BestPractices.html)
- [AWS re:Invent ElastiCache Sessions](https://www.youtube.com/results?search_query=reinvent+elasticache)
- [ElastiCache Pricing](https://aws.amazon.com/elasticache/pricing/)
