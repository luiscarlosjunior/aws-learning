# Amazon RDS (Relational Database Service)

## O que é RDS?

Amazon RDS é um serviço de banco de dados relacional gerenciado que facilita a configuração, operação e escalabilidade de bancos de dados relacionais na nuvem. Ele automatiza tarefas administrativas como provisionamento de hardware, configuração de banco de dados, patches e backups.

## Engines Suportados

### 1. Amazon Aurora
- MySQL e PostgreSQL compatível
- Performance até 5x superior ao MySQL padrão
- Alta disponibilidade integrada

### 2. MySQL
- Versões 5.7 e 8.0
- Community Edition
- InnoDB storage engine

### 3. PostgreSQL
- Versões 11 a 16
- Extensões suportadas
- JSON e tipos avançados

### 4. MariaDB
- Versões 10.4 a 10.11
- Fork do MySQL
- Recursos adicionais

### 5. Oracle
- Standard Edition (SE2)
- Enterprise Edition (EE)
- Bring Your Own License (BYOL)

### 6. SQL Server
- Express, Web, Standard, Enterprise
- Versões 2016, 2017, 2019, 2022
- Windows Authentication

## Conceitos Principais

### DB Instance
Ambiente isolado de banco de dados na nuvem.

**Componentes:**
- Engine específico
- Compute resources (CPU, RAM)
- Storage (SSD ou HDD)
- Network configuration

### Instance Classes

**General Purpose (T3, T4g, M5, M6g):**
- Balanceado compute, memory, network
- Burstable (T classes)
- Casos de uso gerais

**Memory Optimized (R5, R6g, X1e):**
- Workloads intensivos em memória
- In-memory databases
- Análises complexas

**Burstable Performance (T3, T4g):**
- Development/test
- Low-traffic apps
- Cost-effective

### Storage Types

**General Purpose SSD (gp3):**
- 3,000 IOPS base
- 125 MB/s throughput
- Cost-effective para maioria dos casos

**General Purpose SSD (gp2):**
- IOPS escalável com storage
- 3 IOPS por GB
- Burst capability

**Provisioned IOPS SSD (io1, io2):**
- Alta performance
- Até 64,000 IOPS
- Latência consistente
- Databases críticos

**Magnetic (Standard):**
- Legacy
- Não recomendado para novos deployments

### Multi-AZ Deployments

Alta disponibilidade através de replicação síncrona.

**Características:**
- Standby replica em outra AZ
- Failover automático
- Sem downtime durante patches
- Backup da standby replica
- RTO: 1-2 minutos

**Como funciona:**
```
Primary DB Instance (AZ-A)
         |
    Sync Replication
         |
Standby DB Instance (AZ-B)
```

**Failover Automático em caso de:**
- Falha na AZ primária
- Falha na instância primária
- Perda de conectividade de rede
- Storage failure

### Read Replicas

Cópias somente leitura para escalar reads.

**Características:**
- Replicação assíncrona
- Até 15 read replicas (Aurora: 15)
- Pode ser promovida a standalone
- Cross-region support
- Pode ter Multi-AZ habilitado

**Casos de Uso:**
- Distribuir carga de leitura
- Analytics e reporting
- Disaster recovery
- Geographic distribution

**Diferenças Multi-AZ vs Read Replicas:**

| Característica | Multi-AZ | Read Replicas |
|----------------|----------|---------------|
| **Replicação** | Síncrona | Assíncrona |
| **Propósito** | HA/DR | Scaling de leitura |
| **Região** | Same region | Cross-region OK |
| **Acesso** | Não | Sim (somente leitura) |
| **Failover** | Automático | Manual (promote) |
| **Custo** | 2x | Per replica |

## Backup e Recovery

### Automated Backups

**Características:**
- Backup diário completo
- Transaction logs a cada 5 minutos
- Retenção: 1 a 35 dias
- Point-in-time recovery (PITR)
- Armazenado no S3
- Sem custo adicional (até 100% storage)

**Backup Window:**
- Define horário preferencial
- Impacto mínimo em performance
- 30 minutos de duração

### Manual Snapshots

**Características:**
- Criados pelo usuário
- Retenção indefinida
- Pode ser compartilhado entre contas
- Pode ser copiado entre regiões
- Incremental após primeiro snapshot

### Restore

**Point-in-Time Recovery:**
```bash
# Restore para 5 minutos atrás
aws rds restore-db-instance-to-point-in-time \
    --source-db-instance-identifier mydb \
    --target-db-instance-identifier mydb-restored \
    --restore-time 2024-01-01T10:00:00Z
```

**Snapshot Restore:**
```bash
# Restore de snapshot
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier mydb-restored \
    --db-snapshot-identifier mydb-snapshot-2024
```

## Segurança

### Network Isolation

**VPC Integration:**
- DB Subnet Groups
- Multiple AZs
- Private subnets recommended
- Security Groups para acesso

**Exemplo:**
```
VPC (10.0.0.0/16)
  |
  ├─ Private Subnet AZ-A (10.0.1.0/24) - Primary DB
  └─ Private Subnet AZ-B (10.0.2.0/24) - Standby DB
```

### Security Groups

```bash
# Permitir acesso da aplicação
aws ec2 authorize-security-group-ingress \
    --group-id sg-12345678 \
    --protocol tcp \
    --port 3306 \
    --source-group sg-app-servers
```

### Encryption

**At Rest:**
- AWS KMS encryption
- Enabled at creation
- Não pode ser desabilitado depois
- Snapshots também encrypted

**In Transit:**
- SSL/TLS connections
- Force SSL connection
- Certificate validation

### IAM Database Authentication

**Benefícios:**
- Sem passwords no código
- Token de autenticação (15 min)
- Integração com IAM
- Audit trail no CloudTrail

**Exemplo Python:**
```python
import boto3

rds_client = boto3.client('rds')

token = rds_client.generate_db_auth_token(
    DBHostname='mydb.abc123.us-east-1.rds.amazonaws.com',
    Port=3306,
    DBUsername='myuser',
    Region='us-east-1'
)

# Use token como password
```

## Performance

### Enhanced Monitoring

Métricas do OS em tempo real (1 segundo).

**Métricas:**
- CPU utilization
- Memory
- Disk I/O
- Network throughput
- Active sessions

### Performance Insights

Análise de performance do banco de dados.

**Recursos:**
- Top SQL queries
- Wait events
- Database load
- Historical analysis
- 7 dias gratuito (até 2 anos paid)

### Parameter Groups

Configuração de parâmetros do engine.

**Tipos:**
- Default parameter groups (read-only)
- Custom parameter groups

**Exemplo MySQL:**
```bash
# Criar custom parameter group
aws rds create-db-parameter-group \
    --db-parameter-group-name my-mysql-params \
    --db-parameter-group-family mysql8.0 \
    --description "Custom MySQL parameters"

# Modificar parâmetro
aws rds modify-db-parameter-group \
    --db-parameter-group-name my-mysql-params \
    --parameters "ParameterName=max_connections,ParameterValue=500,ApplyMethod=immediate"
```

### Option Groups

Recursos adicionais do engine.

**Exemplos:**
- Oracle: Advanced Security, Transparent Data Encryption
- SQL Server: SQL Server Audit, Native Backup/Restore
- MySQL: MariaDB Audit Plugin

## Manutenção

### Maintenance Window

Janela semanal para manutenção automática.

**Atividades:**
- OS patches
- Database engine patches
- Minor version upgrades
- Scaling operations

**Configuração:**
```bash
aws rds modify-db-instance \
    --db-instance-identifier mydb \
    --preferred-maintenance-window sun:03:00-sun:04:00
```

### Upgrades

**Minor Version Upgrades:**
- Automático (opt-in)
- Durante maintenance window
- Patches de segurança
- Bug fixes

**Major Version Upgrades:**
- Manual only
- Pode requerer downtime
- Testar em staging primeiro
- Snapshot backup recomendado

## Monitoramento

### CloudWatch Metrics

**Métricas Principais:**
- CPUUtilization
- DatabaseConnections
- FreeableMemory
- FreeStorageSpace
- ReadIOPS / WriteIOPS
- ReadLatency / WriteLatency
- NetworkReceiveThroughput / NetworkTransmitThroughput

### Logs

**Tipos de Logs:**
- Error log
- Slow query log
- General log
- Audit log (MariaDB/MySQL)

**Publicar para CloudWatch:**
```bash
aws rds modify-db-instance \
    --db-instance-identifier mydb \
    --cloudwatch-logs-export-configuration '{"EnableLogTypes":["error","slowquery"]}'
```

### Alarms

```bash
# Alarm para CPU alto
aws cloudwatch put-metric-alarm \
    --alarm-name rds-cpu-high \
    --alarm-description "RDS CPU > 80%" \
    --metric-name CPUUtilization \
    --namespace AWS/RDS \
    --statistic Average \
    --period 300 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 2 \
    --dimensions Name=DBInstanceIdentifier,Value=mydb
```

## Exemplos Práticos

### 1. Criar RDS MySQL com Multi-AZ

```bash
aws rds create-db-instance \
    --db-instance-identifier mydb-mysql \
    --db-instance-class db.t3.medium \
    --engine mysql \
    --engine-version 8.0.35 \
    --master-username admin \
    --master-user-password MySecurePassword123! \
    --allocated-storage 100 \
    --storage-type gp3 \
    --storage-encrypted \
    --multi-az \
    --vpc-security-group-ids sg-12345678 \
    --db-subnet-group-name my-subnet-group \
    --backup-retention-period 7 \
    --preferred-backup-window "03:00-04:00" \
    --preferred-maintenance-window "sun:04:00-sun:05:00" \
    --enable-cloudwatch-logs-exports error slowquery \
    --enable-performance-insights \
    --performance-insights-retention-period 7
```

### 2. Criar Read Replica

```bash
# Criar read replica na mesma região
aws rds create-db-instance-read-replica \
    --db-instance-identifier mydb-read-replica \
    --source-db-instance-identifier mydb-mysql \
    --db-instance-class db.t3.medium

# Criar read replica em outra região
aws rds create-db-instance-read-replica \
    --db-instance-identifier mydb-cross-region-replica \
    --source-db-instance-identifier arn:aws:rds:us-east-1:123456789012:db:mydb-mysql \
    --db-instance-class db.t3.medium \
    --region eu-west-1
```

### 3. Python - Conectar ao RDS MySQL

```python
import pymysql
import boto3
from contextlib import contextmanager

# Configuração
RDS_HOST = "mydb.abc123.us-east-1.rds.amazonaws.com"
DB_NAME = "myapp"
DB_USER = "admin"
DB_PASSWORD = "MySecurePassword123!"

@contextmanager
def get_db_connection():
    """Context manager para conexão com RDS"""
    connection = None
    try:
        connection = pymysql.connect(
            host=RDS_HOST,
            user=DB_USER,
            password=DB_PASSWORD,
            database=DB_NAME,
            connect_timeout=5,
            cursorclass=pymysql.cursors.DictCursor
        )
        yield connection
    finally:
        if connection:
            connection.close()

# Uso
with get_db_connection() as conn:
    with conn.cursor() as cursor:
        # Insert
        cursor.execute(
            "INSERT INTO users (name, email) VALUES (%s, %s)",
            ("João Silva", "joao@example.com")
        )
        conn.commit()
        
        # Select
        cursor.execute("SELECT * FROM users WHERE email = %s", ("joao@example.com",))
        result = cursor.fetchone()
        print(result)
```

### 4. Python - Usar IAM Authentication

```python
import pymysql
import boto3

def get_iam_auth_token():
    """Gera token de autenticação IAM"""
    rds_client = boto3.client('rds', region_name='us-east-1')
    
    token = rds_client.generate_db_auth_token(
        DBHostname='mydb.abc123.us-east-1.rds.amazonaws.com',
        Port=3306,
        DBUsername='iamuser',
        Region='us-east-1'
    )
    return token

# Conectar usando IAM auth
connection = pymysql.connect(
    host='mydb.abc123.us-east-1.rds.amazonaws.com',
    user='iamuser',
    password=get_iam_auth_token(),
    database='myapp',
    ssl={'ssl': {'ca': 'rds-ca-2019-root.pem'}}
)
```

### 5. Node.js - Conectar ao RDS PostgreSQL

```javascript
const { Pool } = require('pg');

// Configuração do pool de conexões
const pool = new Pool({
  host: 'mydb.abc123.us-east-1.rds.amazonaws.com',
  port: 5432,
  database: 'myapp',
  user: 'admin',
  password: 'MySecurePassword123!',
  max: 20, // Máximo de conexões no pool
  idleTimeoutMillis: 30000,
  connectionTimeoutMillis: 2000,
});

// Query simples
async function getUsers() {
  const client = await pool.connect();
  try {
    const result = await client.query('SELECT * FROM users WHERE active = $1', [true]);
    return result.rows;
  } finally {
    client.release();
  }
}

// Transaction
async function createUserWithProfile(name, email, profile) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    
    const userResult = await client.query(
      'INSERT INTO users (name, email) VALUES ($1, $2) RETURNING id',
      [name, email]
    );
    const userId = userResult.rows[0].id;
    
    await client.query(
      'INSERT INTO profiles (user_id, bio) VALUES ($1, $2)',
      [userId, profile]
    );
    
    await client.query('COMMIT');
    return userId;
  } catch (error) {
    await client.query('ROLLBACK');
    throw error;
  } finally {
    client.release();
  }
}

// Uso
getUsers()
  .then(users => console.log('Users:', users))
  .catch(err => console.error('Error:', err));
```

### 6. Lambda Function com RDS

```python
import json
import pymysql
import os

# Configuração de ambiente
RDS_HOST = os.environ['RDS_HOST']
DB_NAME = os.environ['DB_NAME']
DB_USER = os.environ['DB_USER']
DB_PASSWORD = os.environ['DB_PASSWORD']

# Conexão global (reuso entre invocações)
connection = None

def get_connection():
    """Obtém ou reusa conexão"""
    global connection
    
    try:
        if connection is None or not connection.open:
            connection = pymysql.connect(
                host=RDS_HOST,
                user=DB_USER,
                password=DB_PASSWORD,
                database=DB_NAME,
                connect_timeout=5
            )
    except pymysql.MySQLError as e:
        print(f"Error connecting to database: {e}")
        raise
    
    return connection

def lambda_handler(event, context):
    """Lambda handler"""
    try:
        conn = get_connection()
        
        with conn.cursor() as cursor:
            # Query baseada no evento
            user_id = event.get('userId')
            cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
            result = cursor.fetchone()
            
            if result:
                return {
                    'statusCode': 200,
                    'body': json.dumps({
                        'user': {
                            'id': result[0],
                            'name': result[1],
                            'email': result[2]
                        }
                    })
                }
            else:
                return {
                    'statusCode': 404,
                    'body': json.dumps({'message': 'User not found'})
                }
                
    except Exception as e:
        print(f"Error: {e}")
        return {
            'statusCode': 500,
            'body': json.dumps({'message': 'Internal server error'})
        }
```

## Melhores Práticas

### Alta Disponibilidade
1. **Use Multi-AZ** para production workloads
2. **Configure automated backups** com retenção adequada
3. **Teste failover procedures** regularmente
4. **Use read replicas** para disaster recovery
5. **Monitor health checks** e configure alarms

### Performance
1. **Right-size instances** baseado em workload
2. **Use Provisioned IOPS** para workloads críticos
3. **Implemente connection pooling** na aplicação
4. **Monitor slow queries** e otimize
5. **Use Performance Insights** para análise
6. **Configure parameter groups** apropriadamente
7. **Use read replicas** para distribuir leitura

### Segurança
1. **Use VPC** e private subnets
2. **Enable encryption** at rest e in transit
3. **Use IAM authentication** quando possível
4. **Restrict security groups** (least privilege)
5. **Regular security patching** (auto minor upgrades)
6. **Enable audit logging** para compliance
7. **Rotate credentials** regularmente
8. **Use Secrets Manager** para passwords

### Backup e Recovery
1. **Automated backups enabled** (7-35 dias)
2. **Test restore procedures** regularmente
3. **Take manual snapshots** antes de changes
4. **Copy snapshots** para outras regiões (DR)
5. **Document recovery procedures**
6. **Define RTO/RPO** requirements

### Custo
1. **Use Reserved Instances** (1-3 anos, até 69% desconto)
2. **Right-size instances** regularmente
3. **Delete unused read replicas**
4. **Use gp3 instead of gp2** (20% cheaper)
5. **Stop dev/test instances** quando não em uso
6. **Monitor storage growth** (auto-scaling)
7. **Clean up old snapshots**

### Operational Excellence
1. **Tag all resources** para cost allocation
2. **Use parameter groups** para configuração consistente
3. **Automate common tasks** com CloudFormation/Terraform
4. **Document architecture** e procedures
5. **Regular maintenance windows**
6. **Monitor CloudWatch metrics** e set alarms
7. **Use CloudTrail** para audit trail

## Limitações

### Instance Limits
- DB instances por região: 40 (pode aumentar)
- Read replicas por master: 15 (5 para SQL Server)
- Manual snapshots: 100 por região

### Storage
- Min storage: 20 GB (100 GB para SQL Server)
- Max storage: 64 TB (16 TB para SQL Server)
- Storage auto-scaling: Até 65,536 GB

### Backup
- Automated backup retention: 0-35 dias
- Manual snapshots: Unlimited retention
- Point-in-time restore: Até retention period

### Connection
- Max connections depende de instance class
- Example: db.t3.medium MySQL = ~420 connections

## Perguntas e Respostas

### P: Qual a diferença entre Multi-AZ e Read Replicas?
**R:** Multi-AZ é para alta disponibilidade com failover automático e replicação síncrona para uma standby instance que não pode ser acessada. Read Replicas são para escalabilidade de leitura com replicação assíncrona e podem ser acessadas para queries de leitura. Multi-AZ protege contra falhas de AZ, enquanto Read Replicas distribuem carga de leitura.

### P: Como funciona o failover em Multi-AZ?
**R:** O DNS CNAME é automaticamente redirecionado da instância primária para a standby em 1-2 minutos. A aplicação usa o mesmo endpoint, sem necessidade de mudanças. O failover é acionado em caso de falha de AZ, instância, storage ou perda de rede.

### P: Posso escalar RDS verticalmente sem downtime?
**R:** Para Single-AZ, há downtime durante escala vertical. Para Multi-AZ, o RDS primeiro faz upgrade na standby, depois failover (1-2 min downtime), e então upgrade na antiga primary. O impacto é minimizado mas há um breve período de indisponibilidade durante o failover.

### P: Como funciona o backup automático?
**R:** RDS faz snapshot diário completo durante a backup window e captura transaction logs a cada 5 minutos. Isso permite point-in-time recovery para qualquer segundo dentro do retention period (1-35 dias). Backups são armazenados no S3 e não impactam performance.

### P: Qual o custo de Multi-AZ?
**R:** Multi-AZ dobra o custo de compute (duas instâncias) mas não cobra extra por replicação de dados. Storage é cobrado apenas uma vez. Vale o custo para produção devido à alta disponibilidade.

### P: Posso acessar a standby instance em Multi-AZ?
**R:** Não. A standby instance não pode ser acessada diretamente. Para distribuir leitura, use Read Replicas. A standby é apenas para failover automático.

### P: Como migrar de Single-AZ para Multi-AZ?
**R:** Basta modificar a instância e habilitar Multi-AZ. O RDS criará automaticamente a standby replica com downtime mínimo (snapshot, restore, sync replication). Leva alguns minutos dependendo do tamanho do banco.

### P: RDS suporta SSL/TLS?
**R:** Sim. RDS fornece certificados SSL/TLS para todas as engines. Você pode forçar SSL editando parameter groups (require_secure_transport=ON para MySQL). Certificados devem ser baixados da AWS para validação.

### P: Como conectar Lambda ao RDS?
**R:** Lambda deve estar na mesma VPC do RDS. Configure Lambda para usar subnets privadas com NAT Gateway para internet access se necessário. Use RDS Proxy para gerenciar connection pooling eficientemente e evitar esgotar conexões.

### P: O que é RDS Proxy?
**R:** Serviço gerenciado que fica entre aplicação e RDS, gerenciando connection pooling e failover. Ideal para aplicações serverless (Lambda) que abrem/fecham conexões frequentemente. Reduz overhead de conexão e melhora scalability.

### P: Posso mudar o engine depois de criar RDS?
**R:** Não diretamente. Você precisa usar AWS DMS (Database Migration Service) para migrar dados para nova instância com engine diferente. Processo requer planejamento para conversão de schema e teste.

### P: Como funciona storage auto-scaling?
**R:** Quando habilitado, RDS monitora storage e automaticamente aumenta quando fica < 10% livre, ou < 6 GB livre, ou por mais de 5 minutos. Aumenta em 10% ou 10 GB (maior). Tem cooldown de 6 horas entre aumentos.

### P: Qual a diferença entre gp2 e gp3?
**R:** gp3 oferece IOPS e throughput configuráveis independente de storage size, é ~20% mais barato e permite até 16,000 IOPS (vs 16,000 para gp2). gp3 é recomendado para novos deployments. gp2 tem IOPS baseado em storage (3 IOPS/GB).

### P: Como fazer upgrade de major version?
**R:** Major version upgrades são manuais. Recomenda-se: 1) Testar em ambiente staging, 2) Criar snapshot manual, 3) Revisar breaking changes, 4) Planejar maintenance window, 5) Executar upgrade. Pode haver downtime dependendo do tamanho.

### P: RDS é serverless?
**R:** RDS tradicional não é serverless (você provisiona instâncias). Aurora Serverless é a opção serverless da AWS que auto-scales baseado em workload. Para RDS tradicional, você gerencia instance types e capacity.

### P: Como implementar disaster recovery?
**R:** 1) Enable Multi-AZ para same-region HA, 2) Create cross-region read replicas, 3) Copy automated snapshots para outra região, 4) Document failover procedures, 5) Test DR regularly. Defina RTO/RPO e escolha estratégia apropriada.

### P: Posso parar/iniciar RDS para economizar?
**R:** Sim, exceto para Multi-AZ SQL Server. Instância parada não cobra compute mas cobra storage. Após 7 dias parada, AWS automaticamente inicia a instância para manutenção. Útil para dev/test environments.

### P: Como monitorar performance?
**R:** Use 1) CloudWatch metrics (CPU, connections, IOPS, latency), 2) Enhanced Monitoring (OS-level metrics, 1-sec granularity), 3) Performance Insights (database-level analysis, slow queries), 4) Database logs (slow query log, error log). Configure alarms para métricas críticas.

### P: O que é Parameter Group vs Option Group?
**R:** Parameter Group configura parâmetros do database engine (max_connections, buffer sizes, etc). Option Group adiciona features opcionais específicas do engine (Oracle TDE, SQL Server Audit). Parameter groups são mais comuns de modificar.

### P: Como lidar com muitas conexões?
**R:** 1) Implement connection pooling na app, 2) Use RDS Proxy, 3) Right-size instance (mais RAM = mais conexões), 4) Monitor connections no CloudWatch, 5) Configure max_connections adequadamente, 6) Considere scaling vertical se necessário.

## Recursos de Aprendizado

- [RDS Documentation](https://docs.aws.amazon.com/rds/)
- [RDS Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html)
- [RDS FAQ](https://aws.amazon.com/rds/faqs/)
- [AWS re:Invent RDS Sessions](https://www.youtube.com/results?search_query=reinvent+rds)
- [RDS Pricing](https://aws.amazon.com/rds/pricing/)
