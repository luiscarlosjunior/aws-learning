# Amazon Aurora

## O que é Aurora?

Amazon Aurora é um banco de dados relacional compatível com MySQL e PostgreSQL, construído para a nuvem, que combina a performance e disponibilidade de bancos comerciais com a simplicidade e custo-efetividade de bancos open source. Aurora oferece até 5x o throughput do MySQL padrão e até 3x do PostgreSQL padrão.

## Arquitetura

### Storage Layer

**Características:**
- Distributed, fault-tolerant, self-healing storage
- 6 cópias dos dados em 3 AZs
- Replicação síncrona e automática
- Storage auto-scaling (10 GB até 128 TB)
- Sem impacto em performance durante scaling

**Como funciona:**
```
┌─────────────────────────────────────┐
│         Aurora Instance             │
│    (Compute + Cache Layer)          │
└──────────────┬──────────────────────┘
               │
┌──────────────┴──────────────────────┐
│     Shared Storage Volume           │
│  (Auto-replicado em 3 AZs)         │
│                                     │
│  AZ-1: [Copy1][Copy2]              │
│  AZ-2: [Copy3][Copy4]              │
│  AZ-3: [Copy5][Copy6]              │
└─────────────────────────────────────┘
```

### Compute Layer

**Primary (Writer) Instance:**
- Uma por cluster
- Read e write operations
- Failover target: Read replica

**Read Replicas:**
- Até 15 replicas
- Milliseconds lag (replicação storage-level)
- Auto-scaling support
- Priority tiers para failover

## Compatibilidade

### Aurora MySQL

**Versões compatíveis:**
- MySQL 5.7
- MySQL 8.0

**Recursos adicionais:**
- Parallel Query
- Backtrack
- Fast clone
- Performance superior

### Aurora PostgreSQL

**Versões compatíveis:**
- PostgreSQL 11, 12, 13, 14, 15, 16

**Recursos adicionais:**
- Babelfish (SQL Server compatibility)
- Partitioning nativo
- Logical replication
- Extensions support

## Aurora Serverless

Configuração auto-scaling para workloads intermitentes ou imprevisíveis.

### Aurora Serverless v2

**Características:**
- Scaling instantâneo (fração de segundo)
- Fine-grained scaling (0.5 ACU increments)
- Min: 0.5 ACU, Max: 128 ACU
- Suporta read replicas
- Multi-AZ support
- Compatible com todas features Aurora

**ACU (Aurora Capacity Unit):**
- 1 ACU = ~2 GB RAM + corresponding CPU + networking
- Billing por ACU-second

**Casos de Uso:**
- Variable workloads
- Development/test environments
- Multi-tenant applications
- Microservices
- Infrequently used applications

**Exemplo de configuração:**
```bash
aws rds create-db-cluster \
    --db-cluster-identifier myapp-serverless \
    --engine aurora-mysql \
    --engine-version 8.0.mysql_aurora.3.04.0 \
    --master-username admin \
    --master-user-password password \
    --serverless-v2-scaling-configuration \
        MinCapacity=0.5,MaxCapacity=16
```

### Aurora Serverless v1 (Legacy)

**Características:**
- Pause/resume automático após inatividade
- Min: 1 ACU, Max: 256 ACU
- Scaling em minutes
- Warm start após pause

**Limitações:**
- Não suporta read replicas
- Não suporta multi-AZ
- Não suporta backtrack
- Não suporta global database

## Recursos Avançados

### Global Database

Multi-region replication com < 1 segundo latency.

**Características:**
- Primary region: Read/write
- Secondary regions: Read-only (até 5)
- Replicação storage-level
- Recovery Point Objective (RPO): 1 segundo
- Recovery Time Objective (RTO): < 1 minuto
- Disaster recovery
- Low-latency global reads

**Arquitetura:**
```
Primary Region (us-east-1)
    ├─ Writer Instance
    └─ Read Replicas (0-15)
        ↓ (< 1s replication)
Secondary Region (eu-west-1)
    └─ Read Replicas (0-15)
        ↓ (< 1s replication)
Secondary Region (ap-southeast-1)
    └─ Read Replicas (0-15)
```

**Failover para outra região:**
```bash
# Promover secondary region para primary
aws rds failover-global-cluster \
    --global-cluster-identifier myglobaldb \
    --target-db-cluster-identifier arn:aws:rds:eu-west-1:123456789012:cluster:mydb-eu
```

### Backtrack

Volta no tempo sem restore de backup (MySQL only).

**Características:**
- Point-in-time "rewind"
- Segundos para completar
- Não cria nova instância
- Window: Até 72 horas
- Útil para corrigir erros (DROP TABLE, DELETE sem WHERE)

**Exemplo:**
```bash
# Backtrack 1 hora
aws rds backtrack-db-cluster \
    --db-cluster-identifier mydb \
    --backtrack-to "2024-01-01T10:00:00Z"
```

**Casos de uso:**
- Undo accidental deletes
- Testing/development
- Data analysis (temporal queries)

### Database Cloning

Clone rápido sem copiar dados.

**Características:**
- Copy-on-write protocol
- Cria clone em minutes (não hours)
- Compartilha storage inicialmente
- Diverge apenas quando dados mudam
- Cost-effective (paga apenas por mudanças)

**Exemplo:**
```bash
# Criar clone
aws rds restore-db-cluster-to-point-in-time \
    --source-db-cluster-identifier mydb-prod \
    --db-cluster-identifier mydb-dev-clone \
    --restore-type copy-on-write \
    --use-latest-restorable-time
```

**Casos de uso:**
- Development/test environments
- Production debugging
- Data analysis
- Pre-deployment testing

### Parallel Query (MySQL)

Acelera analytical queries distribuindo para storage nodes.

**Características:**
- Até 2x mais rápido em scans
- Push-down de WHERE, projections, aggregations
- Automático para queries elegíveis
- Sem mudanças na aplicação

**Quando usar:**
- Large table scans
- Analytical queries
- Reporting queries
- Data exports

**Exemplo de query elegível:**
```sql
-- Esta query pode usar Parallel Query
SELECT 
    region, 
    SUM(sales) 
FROM large_sales_table 
WHERE year = 2024 
GROUP BY region;
```

### Performance Insights

Análise profunda de performance do database.

**Métricas:**
- Database load (Average Active Sessions)
- Top SQL statements
- Wait events
- Dimensions (user, host, database)
- Historical trends

**Uso:**
```sql
-- Identificar queries lentas
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC
LIMIT 10;
```

## High Availability

### Automatic Failover

**Características:**
- Failover automático para read replica
- Priority tiers (0-15, menor = maior prioridade)
- RTO: 30-120 segundos
- Sem perda de dados
- Endpoint único (DNS CNAME update)

**Exemplo de failover priorities:**
```bash
# Primary instance
Instance 1: Writer (tier 0)

# Read replicas com prioridade
Instance 2: Reader (tier 0) - First failover target
Instance 3: Reader (tier 1) - Second failover target
Instance 4: Reader (tier 5) - Lower priority
```

### Storage Auto-Repair

Aurora detecta e repara automaticamente disk failures.

**Processo:**
1. Detect corrupted data block
2. Repair from uncorrupted copies
3. Continue serving requests
4. Zero downtime
5. Transparent para aplicação

### Endpoints

**Cluster Endpoint (Writer):**
- Read/write operations
- Aponta para primary instance
- Failover automático

**Reader Endpoint:**
- Load balancing entre read replicas
- Round-robin distribution

**Instance Endpoints:**
- Direct connection para instância específica
- Useful para workload distribution

**Custom Endpoints:**
- Subconjunto de instances
- Custom load balancing
- Diferentes instance types para diferentes workloads

```bash
# Criar custom endpoint
aws rds create-db-cluster-endpoint \
    --db-cluster-identifier mydb \
    --db-cluster-endpoint-identifier analytics-endpoint \
    --endpoint-type READER \
    --static-members instance-1 instance-2
```

## Performance

### Cache Warming

Aurora mantém cache após failover/restart.

**Buffer cache:**
- Sobrevive instance restarts
- Reduce warm-up time
- Performance consistente após failover

### Fast DDL

Alterações de schema otimizadas.

**Operações rápidas (MySQL):**
- ADD COLUMN (nullable, no default)
- DROP COLUMN
- ALTER COLUMN DEFAULT

**Exemplo:**
```sql
-- Instant DDL
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20);

-- vs traditional (slow)
ALTER TABLE users 
ADD COLUMN phone VARCHAR(20) DEFAULT '+55';
```

### Query Cache (MySQL)

Aurora tem query cache distribuído.

**Características:**
- Cache compartilhado entre instances
- Cache invalidation automático
- Reduz load no primary

## Monitoramento

### CloudWatch Metrics

**Métricas específicas do Aurora:**
- AuroraReplicaLag (milliseconds)
- BufferCacheHitRatio
- DatabaseConnections
- EngineUptime
- RollbackSegmentHistoryListLength
- AuroraGlobalDBReplicationLag (Global Database)
- AuroraGlobalDBDataTransferBytes

### Enhanced Monitoring

OS-level metrics (1-60 seconds).

**Métricas:**
- processList
- cpuUtilization
- memory (detailed breakdown)
- fileSys
- diskIO
- network

## Exemplos Práticos

### 1. Criar Aurora MySQL Cluster

```bash
# Criar cluster
aws rds create-db-cluster \
    --db-cluster-identifier myaurora \
    --engine aurora-mysql \
    --engine-version 8.0.mysql_aurora.3.04.0 \
    --master-username admin \
    --master-user-password MySecurePass123! \
    --database-name myapp \
    --vpc-security-group-ids sg-12345678 \
    --db-subnet-group-name my-subnet-group \
    --backup-retention-period 7 \
    --enable-cloudwatch-logs-exports error slowquery audit \
    --enable-iam-database-authentication \
    --storage-encrypted

# Criar primary instance
aws rds create-db-instance \
    --db-instance-identifier myaurora-instance-1 \
    --db-cluster-identifier myaurora \
    --db-instance-class db.r5.large \
    --engine aurora-mysql \
    --publicly-accessible false

# Criar read replicas
aws rds create-db-instance \
    --db-instance-identifier myaurora-instance-2 \
    --db-cluster-identifier myaurora \
    --db-instance-class db.r5.large \
    --engine aurora-mysql \
    --promotion-tier 0
```

### 2. Python - Conectar ao Aurora

```python
import pymysql
from contextlib import contextmanager

# Endpoints
WRITER_ENDPOINT = "myaurora.cluster-xxx.us-east-1.rds.amazonaws.com"
READER_ENDPOINT = "myaurora.cluster-ro-xxx.us-east-1.rds.amazonaws.com"

@contextmanager
def get_writer_connection():
    """Conexão para writes"""
    conn = pymysql.connect(
        host=WRITER_ENDPOINT,
        user='admin',
        password='MySecurePass123!',
        database='myapp',
        cursorclass=pymysql.cursors.DictCursor
    )
    try:
        yield conn
    finally:
        conn.close()

@contextmanager
def get_reader_connection():
    """Conexão para reads (load balanced)"""
    conn = pymysql.connect(
        host=READER_ENDPOINT,
        user='admin',
        password='MySecurePass123!',
        database='myapp',
        cursorclass=pymysql.cursors.DictCursor
    )
    try:
        yield conn
    finally:
        conn.close()

# Write operation
with get_writer_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute(
            "INSERT INTO orders (user_id, amount) VALUES (%s, %s)",
            (12345, 99.99)
        )
        conn.commit()

# Read operation
with get_reader_connection() as conn:
    with conn.cursor() as cursor:
        cursor.execute("SELECT * FROM orders WHERE user_id = %s", (12345,))
        orders = cursor.fetchall()
        print(orders)
```

### 3. Aurora Serverless v2 - Auto-scaling

```python
import boto3
import pymysql
import time

# Monitorar ACU usage
cloudwatch = boto3.client('cloudwatch')

def get_current_acu(cluster_id):
    """Get current ACU usage"""
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/RDS',
        MetricName='ServerlessDatabaseCapacity',
        Dimensions=[
            {'Name': 'DBClusterIdentifier', 'Value': cluster_id}
        ],
        StartTime=time.time() - 300,  # Last 5 minutes
        EndTime=time.time(),
        Period=60,
        Statistics=['Average']
    )
    
    if response['Datapoints']:
        return response['Datapoints'][-1]['Average']
    return 0

# Connection para Aurora Serverless
conn = pymysql.connect(
    host='myapp-serverless.cluster-xxx.us-east-1.rds.amazonaws.com',
    user='admin',
    password='password',
    database='myapp'
)

print(f"Current ACU: {get_current_acu('myapp-serverless')}")

# Workload - Aurora auto-scales
with conn.cursor() as cursor:
    cursor.execute("SELECT COUNT(*) FROM large_table")
    result = cursor.fetchone()
    print(f"Result: {result}")
    print(f"ACU after query: {get_current_acu('myapp-serverless')}")
```

### 4. Global Database - Multi-region

```python
import boto3
import pymysql
from dataclasses import dataclass

@dataclass
class RegionEndpoint:
    region: str
    endpoint: str
    read_only: bool

# Multi-region endpoints
regions = [
    RegionEndpoint('us-east-1', 'primary.cluster-xxx.us-east-1.rds.amazonaws.com', False),
    RegionEndpoint('eu-west-1', 'secondary.cluster-xxx.eu-west-1.rds.amazonaws.com', True),
    RegionEndpoint('ap-southeast-1', 'secondary.cluster-xxx.ap-southeast-1.rds.amazonaws.com', True),
]

class GlobalDatabaseClient:
    def __init__(self, user, password, database):
        self.user = user
        self.password = password
        self.database = database
        self.regions = regions
    
    def write(self, query, params=None):
        """Write para primary region"""
        primary = next(r for r in self.regions if not r.read_only)
        conn = pymysql.connect(
            host=primary.endpoint,
            user=self.user,
            password=self.password,
            database=self.database
        )
        try:
            with conn.cursor() as cursor:
                cursor.execute(query, params)
                conn.commit()
        finally:
            conn.close()
    
    def read_from_closest(self, query, params=None, preferred_region='us-east-1'):
        """Read da região mais próxima"""
        region = next((r for r in self.regions if r.region == preferred_region), self.regions[0])
        conn = pymysql.connect(
            host=region.endpoint,
            user=self.user,
            password=self.password,
            database=self.database
        )
        try:
            with conn.cursor() as cursor:
                cursor.execute(query, params)
                return cursor.fetchall()
        finally:
            conn.close()

# Uso
client = GlobalDatabaseClient('admin', 'password', 'myapp')

# Write no primary
client.write("INSERT INTO users (name, email) VALUES (%s, %s)", 
             ('João Silva', 'joao@example.com'))

# Read da região mais próxima do usuário
users = client.read_from_closest(
    "SELECT * FROM users WHERE email = %s",
    ('joao@example.com',),
    preferred_region='eu-west-1'
)
print(users)
```

### 5. Backtrack - Time Travel

```python
import boto3
from datetime import datetime, timedelta

rds = boto3.client('rds')

def backtrack_cluster(cluster_id, minutes_ago):
    """Volta cluster no tempo"""
    target_time = datetime.utcnow() - timedelta(minutes=minutes_ago)
    
    print(f"Backtracking {cluster_id} to {target_time}")
    
    response = rds.backtrack_db_cluster(
        DBClusterIdentifier=cluster_id,
        BacktrackTo=target_time,
        Force=False,  # Se True, interrompe operações em andamento
        UseEarliestTimeOnPointInTimeUnavailable=True
    )
    
    print(f"Backtrack initiated:")
    print(f"Status: {response['Status']}")
    print(f"BacktrackTo: {response['BacktrackTo']}")
    print(f"BacktrackedFrom: {response['BacktrackedFrom']}")
    
    return response

# Exemplo: Desfazer delete acidental
# 1. Problema detectado: DROP TABLE executado acidentalmente às 10:30
# 2. Backtrack para 10:25 (5 minutos antes)
backtrack_cluster('myaurora', minutes_ago=5)

# Verificar se dados foram restaurados
import pymysql
conn = pymysql.connect(
    host='myaurora.cluster-xxx.us-east-1.rds.amazonaws.com',
    user='admin',
    password='password',
    database='myapp'
)

with conn.cursor() as cursor:
    cursor.execute("SELECT COUNT(*) FROM my_table")
    count = cursor.fetchone()[0]
    print(f"Table has {count} rows after backtrack")
```

### 6. Database Cloning

```bash
#!/bin/bash

# Clone production para development
echo "Creating clone of production database..."

aws rds restore-db-cluster-to-point-in-time \
    --source-db-cluster-identifier myapp-prod \
    --db-cluster-identifier myapp-dev-clone-$(date +%Y%m%d) \
    --restore-type copy-on-write \
    --use-latest-restorable-time

# Criar instance para o clone
aws rds create-db-instance \
    --db-instance-identifier myapp-dev-clone-$(date +%Y%m%d)-instance-1 \
    --db-cluster-identifier myapp-dev-clone-$(date +%Y%m%d) \
    --db-instance-class db.t3.medium \
    --engine aurora-mysql

echo "Clone created! Available in a few minutes."
echo "Remember: Clone shares storage initially. Only divergent data is stored separately."
```

### 7. Custom Endpoint para Analytics

```python
import boto3
import pymysql

rds = boto3.client('rds')

# Criar custom endpoint para analytics workload
def create_analytics_endpoint(cluster_id, instance_ids):
    """Cria endpoint customizado para analytics"""
    response = rds.create_db_cluster_endpoint(
        DBClusterIdentifier=cluster_id,
        DBClusterEndpointIdentifier=f"{cluster_id}-analytics",
        EndpointType='READER',
        StaticMembers=instance_ids  # Instâncias específicas para analytics
    )
    return response['Endpoint']

# Configurar instances
analytics_instances = ['myaurora-instance-3', 'myaurora-instance-4']
analytics_endpoint = create_analytics_endpoint('myaurora', analytics_instances)

# Aplicação usa diferentes endpoints
OLTP_ENDPOINT = "myaurora.cluster-ro-xxx.us-east-1.rds.amazonaws.com"  # Regular reads
ANALYTICS_ENDPOINT = analytics_endpoint  # Heavy analytics queries

# OLTP queries
oltp_conn = pymysql.connect(host=OLTP_ENDPOINT, user='admin', password='pwd', database='myapp')
with oltp_conn.cursor() as cursor:
    cursor.execute("SELECT * FROM orders WHERE user_id = %s", (123,))

# Analytics queries
analytics_conn = pymysql.connect(host=ANALYTICS_ENDPOINT, user='admin', password='pwd', database='myapp')
with analytics_conn.cursor() as cursor:
    cursor.execute("""
        SELECT DATE(created_at), COUNT(*), SUM(amount)
        FROM orders
        WHERE created_at >= DATE_SUB(NOW(), INTERVAL 1 YEAR)
        GROUP BY DATE(created_at)
    """)
```

## Melhores Práticas

### Arquitetura
1. **Use reader endpoint** para distribuir leitura
2. **Configure failover priorities** adequadamente
3. **Use custom endpoints** para workload separation
4. **Enable Enhanced Monitoring** (1 min) e Performance Insights
5. **Multi-AZ deployment** com múltiplas read replicas

### Performance
1. **Connection pooling** essencial (RDS Proxy recomendado)
2. **Use reader endpoint** para reads, writer para writes
3. **Parallel Query** para analytical workloads (MySQL)
4. **Monitor AuroraReplicaLag** metric
5. **Right-size instances** baseado em workload
6. **Enable Query Cache** (MySQL)

### Alta Disponibilidade
1. **Minimum 2 replicas** em diferentes AZs
2. **Configure failover tiers** (tier 0 para replicas críticas)
3. **Test failover** regularmente
4. **Global Database** para disaster recovery
5. **Automated backups** enabled (1-35 dias)

### Custo
1. **Aurora Serverless v2** para workloads variáveis
2. **Right-size instances** - monitor utilization
3. **Reserved Instances** para production (1-3 anos)
4. **Delete unused clones** regularmente
5. **Storage auto-scales** - sem over-provisioning
6. **Use T3/T4g instances** para dev/test

### Segurança
1. **VPC e private subnets** para isolamento
2. **IAM database authentication** quando possível
3. **Encryption at rest** (KMS) e in transit (SSL)
4. **Security groups** com least privilege
5. **Enable audit logging** (MySQL) ou pgAudit (PostgreSQL)
6. **Regular patching** via maintenance window

### Operacional
1. **Tag resources** para cost tracking
2. **CloudWatch alarms** para métricas críticas
3. **Automate common tasks** (CloudFormation/Terraform)
4. **Document runbooks** para failover e recovery
5. **Regular testing** de DR procedures
6. **Monitor slow queries** via Performance Insights

## Migração

### De MySQL/PostgreSQL para Aurora

**Opções:**
1. **Snapshot migration** (downtime)
2. **Aurora Read Replica** (minimal downtime)
3. **AWS DMS** (continuous replication)
4. **Percona XtraBackup** (MySQL)
5. **pg_dump/pg_restore** (PostgreSQL)

**Exemplo - Read Replica promotion:**
```bash
# 1. Criar Aurora read replica do RDS MySQL
aws rds create-db-instance-read-replica \
    --db-instance-identifier aurora-replica \
    --source-db-instance-identifier mysql-source \
    --db-instance-class db.r5.large \
    --engine aurora-mysql

# 2. Aguardar sync completo (monitor lag)

# 3. Promover replica para standalone Aurora cluster
aws rds promote-read-replica \
    --db-instance-identifier aurora-replica
```

## Limitações

- Max cluster size: 128 TB (auto-scaling)
- Max read replicas: 15
- Max databases per cluster: Unlimited (mas schema size limits apply)
- Max connections: Depende de instance class
- Backtrack window: 72 horas max
- Global Database: 5 secondary regions max
- Clone: Copy-on-write (storage diverges over time)

## Perguntas e Respostas

### P: Qual a diferença entre Aurora e RDS MySQL/PostgreSQL?
**R:** Aurora usa arquitetura distribuída de storage com 6 cópias em 3 AZs, oferecendo até 5x melhor performance, failover mais rápido (30-120s vs 1-2 min), até 15 read replicas (vs 5), auto-scaling storage, e recursos únicos como backtrack e cloning. Aurora é mais caro mas oferece melhor performance e disponibilidade.

### P: Aurora Serverless v1 vs v2?
**R:** V2 escala instantaneamente (frações de segundo vs minutos), suporta todas features Aurora (Global Database, read replicas, Multi-AZ), permite granularidade de 0.5 ACU, e não precisa "warm up". V2 é recomendado para novos projetos. V1 ainda útil se precisa pause/resume automático.

### P: Como funciona o failover no Aurora?
**R:** Failover é para uma read replica (30-120 segundos). Aurora atualiza DNS do cluster endpoint. Prioridade definida por tier (0-15). Se não há replicas, cria nova instância (10-15 minutos). Sem perda de dados pois storage é compartilhado.

### P: Quando usar Global Database?
**R:** Use para disaster recovery com RTO < 1 minuto, low-latency reads globalmente, ou compliance/data residency. Replicação < 1 segundo. Custo: pay for secondary region instances + data transfer (pequeno, storage-level replication).

### P: Backtrack vs Point-in-Time Recovery?
**R:** Backtrack "volta no tempo" a mesma instância em segundos (até 72h), útil para undo rápido. PITR cria nova instância de snapshot+logs (mais lento mas qualquer retention period). Backtrack só MySQL. Use backtrack para erros recentes, PITR para recovery mais antigo.

### P: Como funciona database cloning?
**R:** Clone usa copy-on-write: compartilha storage inicial, só armazena mudanças. Cria clone em minutos independente de tamanho. Cost-effective para dev/test. Clone diverge do source ao fazer writes. Não afeta source.

### P: Aurora suporta todas features MySQL/PostgreSQL?
**R:** Maioria sim, mas algumas diferenças. MySQL: não suporta MyISAM (use InnoDB). PostgreSQL: algumas extensions não suportadas. Check compatibility guide antes de migrar. Performance e features adicionais compensam pequenas limitações.

### P: Reader endpoint distribui carga como?
**R:** Round-robin entre todas read replicas saudáveis. Não considera carga atual. Para controle fino, use custom endpoints com subset de instances ou instance endpoints diretos.

### P: Posso usar Aurora para data warehousing?
**R:** Aurora adequado para OLTP, mas não otimizado para OLAP. Para warehousing, use Redshift. Aurora Parallel Query ajuda em analytical queries mas não substitui warehouse dedicado. Use Aurora para transactional, Redshift para analytics.

### P: Como monitorar Aurora replication lag?
**R:** Métrica AuroraReplicaLag no CloudWatch (milliseconds típico). Alta lag indica replica não acompanhando writes. Causas: workload pesado, instance size pequeno, network issues. Aumente instance size ou adicione mais replicas.

### P: Aurora precisa de Multi-AZ?
**R:** Storage Aurora já é multi-AZ (6 cópias em 3 AZs). Mas para HA de compute, tenha read replicas em outras AZs para failover. Minimum: 1 primary + 1 replica em AZ diferente.

### P: Como migrar de RDS para Aurora sem downtime?
**R:** Crie Aurora read replica do RDS, aguarde sync, pause aplicação brevemente, promova replica para standalone, atualize connection string. Ou use DMS para replicação contínua e cutover quando pronto.

### P: Aurora storage pode diminuir?
**R:** Não. Storage só aumenta automaticamente. Para reduzir, crie snapshot, restore em novo cluster. Storage liberado retorna ao free pool do cluster mas high water mark permanece.

### P: Qual instance class escolher?
**R:** Depende de workload. R5/R6g para memory-intensive, T3/T4g para dev/test ou low traffic, X2g para extrema memory. Start small, monitor CloudWatch (CPU, memory, connections), ajuste conforme necessário. Aurora Serverless v2 elimina este problema.

### P: Como lidar com grandes writes?
**R:** Aurora otimizado para writes (log-based, async to storage nodes). Para bulk loads: 1) Disable binary logging temporário, 2) Use LOAD DATA INFILE, 3) Batch inserts, 4) Increase instance size se necessário, 5) Monitor WriteIOPS e WriteLatency.

### P: Posso acessar Aurora de outra região?
**R:** Sim, mas latency alta. Para low latency, use Global Database (replica em outra região) ou VPC peering/Transit Gateway. Aplicação conecta a replica local.

### P: Diferença entre promotion e failover?
**R:** Failover: automático, 30-120s, em caso de falha, mantém cluster. Promotion: manual, converte read replica em standalone cluster (novo writer endpoint), usado para migração ou split workloads.

### P: Aurora suporta cross-region backups?
**R:** Sim. Copy automated snapshots ou manual snapshots para outras regiões. Também pode usar Global Database como backup strategy com < 1s RPO.

### P: Como debuggar slow queries no Aurora?
**R:** Use Performance Insights para identificar top queries. Enable slow query log. Check explain plans. Para MySQL, Parallel Query pode ajudar em scans. PostgreSQL: use pg_stat_statements. Optimize indexes e queries.

### P: Aurora com Lambda?
**R:** Aurora trabalha bem com Lambda via RDS Proxy (gerencia connection pooling, essencial para Lambda que cria muitas conexões). Ou use Aurora Data API (HTTP API, sem persistent connections, good para intermittent access).

## Recursos de Aprendizado

- [Aurora Documentation](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/)
- [Aurora MySQL Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.BestPractices.html)
- [Aurora PostgreSQL Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraPostgreSQL.BestPractices.html)
- [Aurora Serverless v2](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/aurora-serverless-v2.html)
- [AWS re:Invent Aurora Sessions](https://www.youtube.com/results?search_query=reinvent+aurora)
- [Aurora Pricing](https://aws.amazon.com/rds/aurora/pricing/)
