# Capítulo 2: Storage Types And Relational Stores

O armazenamento de dados é um dos pilares fundamentais de qualquer sistema distribuído moderno. A escolha do tipo de armazenamento adequado pode determinar o sucesso ou fracasso de uma aplicação, impactando diretamente performance, escalabilidade, custo e disponibilidade. Este capítulo explora os diferentes tipos de armazenamento de dados e mergulha profundamente em bancos de dados relacionais, seus conceitos, arquitetura, otimização e estratégias de escalabilidade.

Como afirma Martin Kleppmann em "Designing Data-Intensive Applications": "The choice of storage system is one of the most important architectural decisions you will make" ("A escolha do sistema de armazenamento é uma das decisões arquiteturais mais importantes que você fará") (Kleppmann, 2017).

---

## Data Storage Format (Formato de Armazenamento de Dados)

### Fundamentos Teóricos

O formato de armazenamento de dados refere-se à maneira como os dados são organizados, estruturados e persistidos em mídia física ou virtual. A escolha do formato tem impacto direto em:

- **Performance de Leitura/Escrita**: Throughput e latência de operações I/O
- **Eficiência de Espaço**: Utilização de disco e custos de armazenamento
- **Capacidade de Consulta**: Facilidade de buscar e filtrar dados
- **Escalabilidade**: Capacidade de crescer horizontalmente ou verticalmente
- **Durabilidade**: Garantias de persistência dos dados

### Principais Formatos de Armazenamento

#### 1. Row-Oriented Storage (Armazenamento Orientado a Linhas)

**Conceito:**
Armazena dados linha por linha, onde todos os campos de um registro ficam fisicamente próximos no disco.

**Estrutura:**
```
Disco: [id1, nome1, email1, idade1] [id2, nome2, email2, idade2] [id3, nome3, email3, idade3]
```

**Vantagens:**
- Excelente para operações OLTP (Online Transaction Processing)
- Rápido para operações que envolvem registro completo (INSERT, UPDATE, DELETE)
- Natural para sistemas transacionais tradicionais

**Desvantagens:**
- Ineficiente para consultas analíticas que leem apenas algumas colunas
- I/O desperdiçado ao ler colunas não necessárias

**Exemplos AWS:**
- Amazon RDS (MySQL, PostgreSQL, Oracle, SQL Server)
- Amazon Aurora
- Amazon DynamoDB (NoSQL, mas row-oriented em conceito)

**Caso Real - E-commerce Transacional:**
```python
# Sistema de pedidos - Row-oriented é ideal
# Cada operação acessa registro completo

def create_order(order_data):
    """
    INSERT: Escreve todos os campos de uma vez
    Row-oriented é perfeito para este caso
    """
    db.execute("""
        INSERT INTO orders (id, customer_id, product_id, quantity, price, status, created_at)
        VALUES (?, ?, ?, ?, ?, ?, ?)
    """, order_data)
    # Performance: 5-10ms (single row write)

def get_order(order_id):
    """
    SELECT: Lê registro completo
    Row-oriented recupera tudo em uma leitura sequencial
    """
    return db.query("SELECT * FROM orders WHERE id = ?", order_id)
    # Performance: 2-5ms (single row read)
```

**Referência:** Stonebraker, M. et al. (2007). "C-Store: A Column-oriented DBMS". *VLDB*, 553-564.

#### 2. Column-Oriented Storage (Armazenamento Orientado a Colunas)

**Conceito:**
Armazena dados coluna por coluna, onde valores da mesma coluna ficam fisicamente próximos.

**Estrutura:**
```
Disco: 
Coluna id: [id1, id2, id3, id4, id5, ...]
Coluna nome: [nome1, nome2, nome3, nome4, nome5, ...]
Coluna email: [email1, email2, email3, email4, email5, ...]
```

**Vantagens:**
- Excelente para OLAP (Online Analytical Processing)
- Lê apenas colunas necessárias (I/O reduzido em 10-100x)
- Compressão altamente eficiente (mesmos tipos de dados juntos)
- Agregações rápidas (SUM, AVG, COUNT)

**Desvantagens:**
- Lento para operações que envolvem registro completo
- INSERTs e UPDATEs são mais custosos

**Exemplos AWS:**
- Amazon Redshift (data warehousing)
- Amazon Athena (queries em S3)
- Apache Parquet (formato de arquivo)

**Caso Real - Analytics Dashboard:**
```python
# Análise de vendas - Column-oriented é ideal
# Query lê apenas 2 colunas de 100 milhões de registros

def get_sales_by_region():
    """
    Query analítica: apenas 2 colunas necessárias
    Column-oriented lê apenas essas 2 colunas
    """
    return redshift.query("""
        SELECT region, SUM(sales_amount) as total
        FROM sales
        WHERE date >= '2023-01-01'
        GROUP BY region
    """)
    
    # Row-oriented: Leria TODAS as colunas (50GB)
    # Column-oriented: Lê apenas region + sales_amount (2GB)
    # Speedup: 25x mais rápido

# Performance real no Redshift:
# - Row-oriented: 300-600 segundos
# - Column-oriented: 10-30 segundos
```

**Benchmark Real:**
Estudo da Amazon mostrou que Redshift (columnar) é 10-100x mais rápido que RDS (row-oriented) para queries analíticas, mas 2-5x mais lento para transações individuais (AWS, 2015).

**Referência:** Abadi, D. et al. (2008). "Column-Stores vs. Row-Stores: How Different Are They Really?". *ACM SIGMOD*, 967-980.


### Comparação de Formatos

| Formato | Use Case Ideal | Latência Leitura | Throughput Escrita | Compressão | Complexidade Queries |
|---------|----------------|------------------|-------------------|------------|---------------------|
| Row-Oriented | OLTP, transações | 2-10ms | Alta | Média | Alta (SQL completo) |
| Column-Oriented | OLAP, analytics | 10-100ms (bulk) | Baixa | Muito Alta | Alta (SQL analítico) |
| Log-Structured | Logs, time-series | 1-5ms | Muito Alta | Baixa | Baixa |
| Key-Value | Cache, sessions | 0.2-1ms | Muito Alta | Baixa | Muito Baixa |

---

## File-Based Storage (Armazenamento Baseado em Arquivos)

### Fundamentos

File-based storage organiza dados em arquivos e diretórios, similar a um sistema de arquivos tradicional. É o modelo mais antigo e ainda amplamente utilizado para certos tipos de workloads.

### Amazon EFS (Elastic File System)

**Conceito:**
Sistema de arquivos NFSv4 totalmente gerenciado pela AWS, com escalabilidade automática e acesso compartilhado.

**Características:**
- **Escalabilidade Automática**: Cresce de GB a petabytes automaticamente
- **Performance Modes**:
  - General Purpose: Até 7,000 operações por segundo
  - Max I/O: Centenas de milhares de operações por segundo
- **Throughput Modes**:
  - Bursting: Performance escala com tamanho do file system
  - Provisioned: Throughput independente do tamanho
- **Durabilidade**: 99.999999999% (11 noves)
- **Disponibilidade**: 99.99% com redundância Multi-AZ

**Caso de Uso - Content Management System:**

```python
import os
import boto3

class CMSStorage:
    def __init__(self, efs_mount='/mnt/efs'):
        self.efs_mount = efs_mount
        self.efs_client = boto3.client('efs')
    
    def store_article(self, article_id, content, media_files):
        """
        Múltiplos servidores web podem escrever simultaneamente
        EFS gerencia consistência e concorrência
        """
        article_path = f'{self.efs_mount}/articles/{article_id}'
        os.makedirs(article_path, exist_ok=True)
        
        # Escrever conteúdo
        with open(f'{article_path}/content.html', 'w') as f:
            f.write(content)
        
        # Escrever arquivos de mídia
        media_path = f'{article_path}/media'
        os.makedirs(media_path, exist_ok=True)
        for filename, data in media_files.items():
            with open(f'{media_path}/{filename}', 'wb') as f:
                f.write(data)
        
        return {'status': 'success', 'path': article_path}
    
    def get_article(self, article_id):
        """
        Qualquer servidor pode ler de qualquer artigo
        EFS garante consistência read-after-write
        """
        article_path = f'{self.efs_mount}/articles/{article_id}'
        with open(f'{article_path}/content.html', 'r') as f:
            content = f.read()
        
        # Listar arquivos de mídia
        media_path = f'{article_path}/media'
        media_files = os.listdir(media_path) if os.path.exists(media_path) else []
        
        return {'content': content, 'media': media_files}

# Benefícios do EFS:
# - Servidores web stateless (qualquer um pode servir qualquer requisição)
# - Auto-scaling sem complexidade de compartilhamento de dados
# - Backup automático com AWS Backup
# - Lifecycle management para economia de custos
```

**Performance Benchmarks:**
- Read throughput: 500 MB/s (General Purpose), 3 GB/s (Max I/O)
- Write throughput: 300 MB/s (General Purpose), 1 GB/s (Max I/O)
- Latência: 10-100ms para operações de arquivo

**Referência:** AWS EFS Documentation (2023). "Amazon Elastic File System". *AWS Architecture Center*.

### Amazon FSx

A AWS oferece várias opções de FSx otimizadas para workloads específicos:

#### 1. FSx for Lustre - High-Performance Computing

**Características:**
- Throughput: 200 MB/s a 1 TB/s por file system
- Latência: Sub-milissegundo
- IOPS: Milhões de IOPS
- Integração com S3: Lazy loading e write-back automáticos
- Use Cases: Machine Learning, HPC, Video Processing, Genomics

**Exemplo - ML Training Pipeline:**

```python
class MLTrainingWithFSx:
    def __init__(self):
        self.fsx_mount = '/mnt/fsx'
        self.s3_bucket = 's3://ml-training-data'
    
    def prepare_training_data(self, dataset_name):
        """
        FSx for Lustre lazy-loads dados do S3 conforme necessário
        Não precisa copiar TBs de dados antecipadamente
        """
        # FSx está linked ao bucket S3
        # Dados aparecem como arquivos mas são carregados sob demanda
        dataset_path = f'{self.fsx_mount}/datasets/{dataset_name}'
        
        # Primeira leitura: Busca do S3 (10-50ms + transfer time)
        # Leituras subsequentes: Cache local no FSx (<1ms)
        
        return dataset_path
    
    def train_distributed_model(self, model, dataset_path):
        """
        100+ GPUs lendo dados simultaneamente
        FSx fornece throughput agregado massivo
        """
        # Cada GPU treina em paralelo
        # FSx throughput: 500+ GB/s aggregate
        # vs EFS: 10 GB/s maximum
        # Speedup: 50x
        
        # Training time:
        # - Com EFS: 200 horas
        # - Com FSx: 4 horas
        pass

# Netflix usa FSx for Lustre para encoding de vídeo
# Throughput: 100+ GB/s para processar 1 PB+ por dia
```

#### 2. FSx for Windows File Server

**Características:**
- Protocolo SMB totalmente compatível
- Active Directory integration
- Shadow copies, quotas, ACLs
- Multi-AZ deployment para HA
- Performance: Até 2 GB/s e centenas de milhares de IOPS

**Use Cases:**
- Aplicações Windows legadas
- Home directories
- SQL Server em EC2
- SharePoint

#### 3. FSx for NetApp ONTAP

**Características:**
- Multi-protocol: NFS, SMB, iSCSI
- Snapshots, replicação, tiering
- Deduplicação e compressão integradas
- Performance: Até 36 GB/s

---

## Block-Based Storage (Armazenamento Baseado em Blocos)

### Fundamentos Teóricos

Block storage divide dados em blocos de tamanho fixo, cada um com endereço único. O sistema operacional gerencia esses blocos como se fossem um disco físico.

**Características:**
- **Granularidade**: Blocos de 4KB a 1MB
- **Performance**: Acesso direto, latência muito baixa
- **Flexibilidade**: Sistema operacional gerencia file system
- **IOPS**: Otimizado para operações de I/O aleatórias

### Amazon EBS (Elastic Block Store)

O EBS fornece volumes de bloco persistentes para instâncias EC2.

#### Tipos de Volumes EBS

**1. gp3 (General Purpose SSD) - Recomendado**

```python
# Características do gp3:
volume_gp3 = {
    'type': 'gp3',
    'size': 1000,  # GB
    'iops': 16000,  # 3,000-16,000 IOPS
    'throughput': 1000,  # 125-1,000 MB/s
    'durability': '99.8-99.9%',
    'cost': '$0.08/GB/month',
    'latency': '1ms (single-digit)'
}

# Use cases:
# - Boot volumes
# - Bancos de dados de médio porte
# - Ambientes de desenvolvimento
# - Aplicações de propósito geral
```

**2. io2 Block Express (Provisioned IOPS SSD)**

```python
# Características do io2 Block Express:
volume_io2 = {
    'type': 'io2',
    'size': 64000,  # Até 64 TB
    'iops': 256000,  # Até 256,000 IOPS
    'throughput': 4000,  # Até 4,000 MB/s
    'durability': '99.999%',
    'cost': '$0.125/GB/month + $0.065/provisioned IOPS',
    'latency': '<1ms (sub-millisecond)'
}

# Use cases:
# - Bancos de dados críticos (SAP HANA, Oracle)
# - Aplicações de baixa latência
# - Workloads I/O-intensive
```

**3. st1 (Throughput Optimized HDD)**

```python
# Características do st1:
volume_st1 = {
    'type': 'st1',
    'size': 16000,  # 125 GB - 16 TB
    'throughput': 500,  # Até 500 MB/s
    'iops': 500,  # Máximo 500 IOPS
    'cost': '$0.045/GB/month',
    'latency': '10-20ms'
}

# Use cases:
# - Big Data
# - Data warehouses
# - Log processing
# - Workloads sequenciais
```

**4. sc1 (Cold HDD)**

```python
# Características do sc1:
volume_sc1 = {
    'type': 'sc1',
    'size': 16000,  # 125 GB - 16 TB
    'throughput': 250,  # Até 250 MB/s
    'iops': 250,  # Máximo 250 IOPS
    'cost': '$0.015/GB/month',
    'latency': '10-20ms'
}

# Use cases:
# - Arquivos acessados raramente
# - Dados "cold" que precisam de acesso ocasional
# - Menor custo de armazenamento em bloco
```

### Comparação de Tipos EBS

| Tipo | IOPS | Throughput | Latência | Custo/GB | Use Case |
|------|------|-----------|----------|----------|----------|
| gp3 | 16K | 1 GB/s | 1ms | $0.08 | Propósito geral |
| io2 | 256K | 4 GB/s | <1ms | $0.125+ | Crítico, DB |
| st1 | 500 | 500 MB/s | 10ms | $0.045 | Big Data |
| sc1 | 250 | 250 MB/s | 10ms | $0.015 | Cold storage |

### Padrões de Uso do EBS

#### 1. Database Server

```python
class DatabaseServer:
    """
    Configuração típica de banco de dados com EBS
    """
    def __init__(self):
        self.volumes = {
            'root': {
                'type': 'gp3',
                'size': 50,  # GB
                'iops': 3000,
                'purpose': 'Sistema operacional e binários'
            },
            'data': {
                'type': 'io2',
                'size': 1000,  # GB
                'iops': 64000,
                'purpose': 'Dados do banco (requer baixa latência)'
            },
            'logs': {
                'type': 'gp3',
                'size': 200,  # GB
                'iops': 10000,
                'purpose': 'Transaction logs (write-heavy)'
            },
            'backups': {
                'type': 'st1',
                'size': 2000,  # GB
                'purpose': 'Backups (sequential access)'
            }
        }
    
    def optimize_performance(self):
        """
        Dicas de otimização para databases no EBS:
        """
        optimizations = [
            "Use io2 Block Express para datafiles",
            "Separe logs em volume dedicado",
            "Enable EBS optimization na instância",
            "Use instance types com EBS-optimized bandwidth",
            "Configure RAID 0 para throughput maior",
            "Use snapshot para backups ao invés de backup local"
        ]
        return optimizations

# Performance esperada:
# - Read latency: 0.5-1ms
# - Write latency: 1-2ms
# - IOPS: 64,000 para datafiles
# - Throughput: 1 GB/s
```

#### 2. RAID Configurations

```bash
# RAID 0 para máximo throughput
# Aumenta performance mas NÃO aumenta durabilidade

# 4 volumes io2 de 1TB cada = 4TB total
# IOPS: 64,000 × 4 = 256,000 IOPS
# Throughput: 1 GB/s × 4 = 4 GB/s

# Criar RAID 0:
sudo mdadm --create /dev/md0 --level=0 --raid-devices=4 \
    /dev/xvdf /dev/xvdg /dev/xvdh /dev/xvdi

# Format e mount:
sudo mkfs.ext4 /dev/md0
sudo mount /dev/md0 /data

# WARNING: RAID 0 não tem redundância
# Perda de 1 volume = perda total de dados
# Use apenas com snapshots regulares
```

### EBS Snapshots

**Conceito:**
Backups incrementais armazenados no S3, que capturam apenas mudanças desde último snapshot.

```python
import boto3

class EBSSnapshotManager:
    def __init__(self):
        self.ec2 = boto3.client('ec2')
    
    def create_snapshot(self, volume_id, description):
        """
        Criar snapshot incremental
        """
        snapshot = self.ec2.create_snapshot(
            VolumeId=volume_id,
            Description=description,
            TagSpecifications=[{
                'ResourceType': 'snapshot',
                'Tags': [
                    {'Key': 'Name', 'Value': f'backup-{volume_id}'},
                    {'Key': 'CreatedBy', 'Value': 'AutomatedBackup'}
                ]
            }]
        )
        
        # Características:
        # - Primeiro snapshot: Copia todos os dados
        # - Snapshots subsequentes: Apenas blocos modificados
        # - Armazenamento: Amazon S3 (11 noves de durabilidade)
        # - Custo: $0.05/GB/mês
        
        return snapshot['SnapshotId']
    
    def restore_from_snapshot(self, snapshot_id, availability_zone):
        """
        Criar novo volume a partir de snapshot
        """
        volume = self.ec2.create_volume(
            SnapshotId=snapshot_id,
            AvailabilityZone=availability_zone,
            VolumeType='gp3',
            Iops=16000,
            Throughput=1000
        )
        
        # Volume está disponível imediatamente
        # Dados são lazy-loaded do S3 conforme acessados
        # Performance total após "warming up"
        
        return volume['VolumeId']
    
    def copy_snapshot_cross_region(self, snapshot_id, source_region, dest_region):
        """
        Copiar snapshot para outra região para DR
        """
        dest_ec2 = boto3.client('ec2', region_name=dest_region)
        
        copied_snapshot = dest_ec2.copy_snapshot(
            SourceRegion=source_region,
            SourceSnapshotId=snapshot_id,
            Description=f'DR copy from {source_region}'
        )
        
        # Use case: Disaster Recovery
        # Snapshots em múltiplas regiões garantem recuperação
        # RTO: 15-30 minutos para criar volume e attach
        # RPO: Frequência dos snapshots (ex: diário = 24h)
        
        return copied_snapshot['SnapshotId']

# Estratégia de Backup Automatizado:
# - Hourly snapshots (mantidos por 1 dia)
# - Daily snapshots (mantidos por 7 dias)
# - Weekly snapshots (mantidos por 4 semanas)
# - Monthly snapshots (mantidos por 12 meses)
# - Cross-region copy dos monthly snapshots
```

### Amazon EBS Multi-Attach

**Conceito:**
Permite anexar um único volume io2 a múltiplas instâncias EC2 simultaneamente (até 16 instâncias).

```python
# Use case: Clustered applications
# Exemplo: Oracle RAC, RHEL Cluster

# Criar volume multi-attach:
volume = ec2.create_volume(
    VolumeType='io2',
    Size=1000,
    Iops=64000,
    MultiAttachEnabled=True,
    AvailabilityZone='us-east-1a'
)

# Attach a múltiplas instâncias:
ec2.attach_volume(VolumeId=volume_id, InstanceId='i-instance1', Device='/dev/sdf')
ec2.attach_volume(VolumeId=volume_id, InstanceId='i-instance2', Device='/dev/sdf')

# Importante:
# - Aplicação deve gerenciar sincronização (cluster-aware file system)
# - Use clustered file systems: GFS2, OCFS2
# - Não use com file systems standard (ext4, xfs) - corrupção de dados!
```

---

## Object-Based Storage (Armazenamento Baseado em Objetos)

### Fundamentos Teóricos

Object storage armazena dados como objetos discretos, cada um com:
- **Data**: Os dados reais (file content)
- **Metadata**: Informações sobre o objeto (content-type, custom tags)
- **Unique ID**: Identificador único (key) para recuperação

**Diferenças vs File/Block Storage:**
- Não há hierarquia de diretórios (flat namespace com prefixes simulando pastas)
- Não é possível modificar objeto parcialmente (somente substituir inteiro)
- Altamente escalável (exabytes+)
- Acessado via HTTP REST APIs

### Amazon S3 (Simple Storage Service)

O S3 é o serviço de object storage mais popular e escalável do mundo.

#### Conceitos Fundamentais

**1. Buckets e Objects:**

```python
import boto3

s3 = boto3.client('s3')

# Criar bucket
s3.create_bucket(
    Bucket='my-application-data',
    CreateBucketConfiguration={'LocationConstraint': 'us-west-2'}
)

# Upload object
s3.put_object(
    Bucket='my-application-data',
    Key='users/user123/profile.json',  # Key é o identificador único
    Body=json.dumps({'name': 'John', 'email': 'john@example.com'}),
    ContentType='application/json',
    Metadata={
        'user-id': '123',
        'uploaded-by': 'application'
    }
)

# Download object
response = s3.get_object(
    Bucket='my-application-data',
    Key='users/user123/profile.json'
)
data = json.loads(response['Body'].read())
```

**2. Storage Classes - Otimização de Custos:**

| Storage Class | Durabilidade | Disponibilidade | Latência | Custo/GB/mês | Use Case |
|--------------|--------------|-----------------|----------|--------------|----------|
| S3 Standard | 11 noves | 99.99% | ms | $0.023 | Dados acessados frequentemente |
| S3 Intelligent-Tiering | 11 noves | 99.9% | ms | $0.023-0.0125 | Padrão de acesso desconhecido |
| S3 Standard-IA | 11 noves | 99.9% | ms | $0.0125 | Acesso infrequente |
| S3 One Zone-IA | 11 noves | 99.5% | ms | $0.01 | Dados não-críticos, IA |
| S3 Glacier Instant | 11 noves | 99.9% | ms | $0.004 | Archive com acesso raro |
| S3 Glacier Flexible | 11 noves | 99.99% | min-hr | $0.0036 | Archive, retrieval 1-5min |
| S3 Glacier Deep Archive | 11 noves | 99.99% | hr | $0.00099 | Archive longo prazo, <12hr |

**3. S3 Lifecycle Policies:**

```python
# Política de lifecycle para economia de custos
lifecycle_policy = {
    'Rules': [
        {
            'Id': 'MoveToIA',
            'Status': 'Enabled',
            'Transitions': [
                {
                    'Days': 30,
                    'StorageClass': 'STANDARD_IA'
                },
                {
                    'Days': 90,
                    'StorageClass': 'GLACIER_IR'
                },
                {
                    'Days': 365,
                    'StorageClass': 'DEEP_ARCHIVE'
                }
            ],
            'Expiration': {
                'Days': 2555  # 7 anos
            }
        }
    ]
}

s3.put_bucket_lifecycle_configuration(
    Bucket='my-application-data',
    LifecycleConfiguration=lifecycle_policy
)

# Economia:
# - Mês 0-1: Standard ($0.023/GB) = $23/TB
# - Mês 1-3: Standard-IA ($0.0125/GB) = $12.50/TB
# - Mês 3-12: Glacier IR ($0.004/GB) = $4/TB
# - Ano 1+: Deep Archive ($0.001/GB) = $1/TB
# Economia total: 95%+ para dados antigos
```

#### Padrões de Uso do S3

**1. Static Website Hosting:**

```python
# Configurar bucket para hosting de site estático
s3.put_bucket_website(
    Bucket='my-website',
    WebsiteConfiguration={
        'IndexDocument': {'Suffix': 'index.html'},
        'ErrorDocument': {'Key': 'error.html'}
    }
)

# Upload HTML, CSS, JS
s3.put_object(
    Bucket='my-website',
    Key='index.html',
    Body=open('index.html', 'rb'),
    ContentType='text/html',
    CacheControl='max-age=3600'
)

# Adicionar CloudFront para performance global
cloudfront.create_distribution(
    DistributionConfig={
        'Origins': {
            'Items': [{
                'Id': 's3-website',
                'DomainName': 'my-website.s3-website-us-east-1.amazonaws.com',
                'S3OriginConfig': {}
            }]
        },
        'Enabled': True,
        'Comment': 'Static website CDN'
    }
)

# Benefícios:
# - Custo: 10-100x mais barato que servidores web
# - Escalabilidade: Ilimitada (S3 + CloudFront)
# - Performance: <50ms para usuários globais
# - Durabilidade: 11 noves (nunca perde arquivos)
```

**2. Data Lake Architecture:**

```python
class DataLake:
    """
    Arquitetura de Data Lake no S3
    """
    def __init__(self, bucket_name):
        self.bucket = bucket_name
        self.s3 = boto3.client('s3')
        self.glue = boto3.client('glue')
        self.athena = boto3.client('athena')
    
    def ingest_data(self, source, data, partition_keys):
        """
        Ingerir dados com particionamento para queries eficientes
        """
        # Particionar por data para performance
        year = partition_keys['year']
        month = partition_keys['month']
        day = partition_keys['day']
        
        key = f'raw/{source}/year={year}/month={month}/day={day}/data.parquet'
        
        # Salvar em formato colunar (Parquet) para analytics
        self.s3.put_object(
            Bucket=self.bucket,
            Key=key,
            Body=data
        )
        
        # Registrar no Glue Data Catalog
        self.glue.create_partition(
            DatabaseName='datalake',
            TableName=source,
            PartitionInput={
                'Values': [year, month, day],
                'StorageDescriptor': {
                    'Location': f's3://{self.bucket}/raw/{source}/year={year}/month={month}/day={day}/',
                    'InputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetInputFormat',
                    'OutputFormat': 'org.apache.hadoop.hive.ql.io.parquet.MapredParquetOutputFormat'
                }
            }
        )
    
    def query_data(self, sql_query):
        """
        Query data usando Athena (serverless SQL)
        """
        response = self.athena.start_query_execution(
            QueryString=sql_query,
            QueryExecutionContext={'Database': 'datalake'},
            ResultConfiguration={
                'OutputLocation': f's3://{self.bucket}/athena-results/'
            }
        )
        
        # Athena processa terabytes de dados diretamente no S3
        # Paga apenas pelo volume de dados escaneados
        # Custo: $5 por TB escaneado
        
        return response['QueryExecutionId']

# Exemplo de query:
sql = """
    SELECT product_category, SUM(sales_amount) as total_sales
    FROM sales_data
    WHERE year = '2023' AND month = '12'
    GROUP BY product_category
    ORDER BY total_sales DESC
"""

# Performance:
# - Dataset: 10 TB de dados
# - Particionamento: Apenas 100 GB escaneados (1 mês)
# - Query time: 30-60 segundos
# - Custo: $0.50 (100 GB × $5/TB)
```

**3. Backup e Archive:**

```python
class BackupManager:
    """
    Gerenciamento de backups com S3 e Glacier
    """
    def __init__(self):
        self.s3 = boto3.client('s3')
        self.backup_bucket = 'company-backups'
    
    def backup_database(self, db_name, backup_data):
        """
        Backup de banco de dados com múltiplas camadas
        """
        timestamp = datetime.now().strftime('%Y-%m-%d-%H-%M-%S')
        key = f'databases/{db_name}/backup-{timestamp}.sql.gz'
        
        # Comprimir backup
        compressed = gzip.compress(backup_data.encode())
        
        # Upload para S3
        self.s3.put_object(
            Bucket=self.backup_bucket,
            Key=key,
            Body=compressed,
            StorageClass='STANDARD_IA',  # Acesso infrequente
            ServerSideEncryption='AES256',  # Criptografia
            Metadata={
                'database': db_name,
                'backup-type': 'full',
                'compression': 'gzip'
            }
        )
        
        # Lifecycle automático:
        # - 0-7 dias: Standard-IA
        # - 7-90 dias: Glacier Flexible Retrieval
        # - 90+ dias: Glacier Deep Archive
        
        return key
    
    def restore_database(self, backup_key):
        """
        Restaurar backup (com retrieval time baseado em storage class)
        """
        # Verificar storage class
        head = self.s3.head_object(Bucket=self.backup_bucket, Key=backup_key)
        storage_class = head['StorageClass']
        
        if storage_class == 'DEEP_ARCHIVE':
            # Iniciar restoration (12 horas)
            self.s3.restore_object(
                Bucket=self.backup_bucket,
                Key=backup_key,
                RestoreRequest={
                    'Days': 1,
                    'GlacierJobParameters': {'Tier': 'Standard'}
                }
            )
            return {'status': 'restoring', 'eta_hours': 12}
        elif storage_class == 'GLACIER':
            # Restoration rápida (1-5 minutos)
            self.s3.restore_object(
                Bucket=self.backup_bucket,
                Key=backup_key,
                RestoreRequest={
                    'Days': 1,
                    'GlacierJobParameters': {'Tier': 'Expedited'}
                }
            )
            return {'status': 'restoring', 'eta_minutes': 5}
        else:
            # Disponível imediatamente
            response = self.s3.get_object(Bucket=self.backup_bucket, Key=backup_key)
            compressed_data = response['Body'].read()
            backup_data = gzip.decompress(compressed_data).decode()
            return {'status': 'ready', 'data': backup_data}

# Custo Comparison:
# 100 TB de backups por 7 anos:
# - Sem lifecycle: $23,000/mês (Standard)
# - Com lifecycle: $1,200/mês (Deep Archive)
# Economia: 95%
```

#### S3 Performance Optimization

**1. Multipart Upload para Arquivos Grandes:**

```python
def upload_large_file(file_path, bucket, key):
    """
    Upload paralelo de arquivo grande (>100 MB)
    """
    # Configuração
    GB = 1024 ** 3
    file_size = os.path.getsize(file_path)
    part_size = 100 * 1024 * 1024  # 100 MB por parte
    
    # Iniciar multipart upload
    mpu = s3.create_multipart_upload(Bucket=bucket, Key=key)
    upload_id = mpu['UploadId']
    
    # Upload parts em paralelo
    parts = []
    with open(file_path, 'rb') as f:
        part_number = 1
        while True:
            data = f.read(part_size)
            if not data:
                break
            
            # Upload part
            part = s3.upload_part(
                Bucket=bucket,
                Key=key,
                PartNumber=part_number,
                UploadId=upload_id,
                Body=data
            )
            
            parts.append({
                'PartNumber': part_number,
                'ETag': part['ETag']
            })
            part_number += 1
    
    # Completar upload
    s3.complete_multipart_upload(
        Bucket=bucket,
        Key=key,
        UploadId=upload_id,
        MultipartUpload={'Parts': parts}
    )
    
    # Benefícios:
    # - Paralelismo: 10x mais rápido
    # - Resiliência: Retry individual parts
    # - Tamanho: Até 5 TB por object
    # Performance: 
    # - Single-part: 50 MB/s
    # - Multi-part (10 threads): 500 MB/s
```

**2. S3 Transfer Acceleration:**

```python
# Habilitar Transfer Acceleration
s3.put_bucket_accelerate_configuration(
    Bucket='my-bucket',
    AccelerateConfiguration={'Status': 'Enabled'}
)

# Upload usando Transfer Acceleration endpoint
s3_accelerate = boto3.client(
    's3',
    endpoint_url='https://my-bucket.s3-accelerate.amazonaws.com'
)

s3_accelerate.upload_file('large-file.zip', 'my-bucket', 'file.zip')

# Performance Gains:
# Upload de Tokyo para US-EAST-1:
# - Standard S3: 80 MB/s
# - Transfer Acceleration: 300 MB/s
# Speedup: 3-4x

# Custo adicional: $0.04/GB (somente quando é mais rápido)
```

**3. S3 Request Rate Performance:**

```python
# S3 suporta:
# - 3,500 PUT/COPY/POST/DELETE requests por segundo por prefix
# - 5,500 GET/HEAD requests por segundo por prefix

# Para workloads de alto throughput, use múltiplos prefixes:

# ❌ Ruim: Todos objects no mesmo prefix
# s3://bucket/images/image1.jpg
# s3://bucket/images/image2.jpg
# ... (limitado a 5,500 GET/s)

# ✓ Bom: Distribuir em múltiplos prefixes
# s3://bucket/images-a/image1.jpg
# s3://bucket/images-b/image2.jpg
# s3://bucket/images-c/image3.jpg
# ... (55,000 GET/s com 10 prefixes)

def generate_distributed_key(filename, num_prefixes=10):
    """
    Distribuir objects em múltiplos prefixes para performance
    """
    import hashlib
    hash_value = hashlib.md5(filename.encode()).hexdigest()
    prefix_id = int(hash_value, 16) % num_prefixes
    return f'images-{prefix_id}/{filename}'

# Com 10 prefixes:
# - GET capacity: 55,000 requests/s
# - PUT capacity: 35,000 requests/s
```

#### S3 Security e Compliance

**1. Encryption:**

```python
# Server-Side Encryption (SSE)

# SSE-S3: Managed by AWS
s3.put_object(
    Bucket='my-bucket',
    Key='sensitive-data.txt',
    Body='confidential information',
    ServerSideEncryption='AES256'
)

# SSE-KMS: Customer Managed Keys
s3.put_object(
    Bucket='my-bucket',
    Key='top-secret.txt',
    Body='classified information',
    ServerSideEncryption='aws:kms',
    SSEKMSKeyId='arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012'
)

# Client-Side Encryption: Encrypt before upload
from cryptography.fernet import Fernet

key = Fernet.generate_key()
cipher = Fernet(key)
encrypted_data = cipher.encrypt(b'secret data')

s3.put_object(
    Bucket='my-bucket',
    Key='client-encrypted.bin',
    Body=encrypted_data
)
```

**2. Access Control:**

```python
# Bucket Policy: S3-level permissions
bucket_policy = {
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "PublicReadGetObject",
            "Effect": "Allow",
            "Principal": "*",
            "Action": "s3:GetObject",
            "Resource": "arn:aws:s3:::my-public-bucket/*",
            "Condition": {
                "IpAddress": {
                    "aws:SourceIp": ["54.240.143.0/24"]
                }
            }
        }
    ]
}

s3.put_bucket_policy(
    Bucket='my-public-bucket',
    Policy=json.dumps(bucket_policy)
)

# Object ACL: Object-level permissions
s3.put_object_acl(
    Bucket='my-bucket',
    Key='shared-file.pdf',
    ACL='public-read'
)

# Pre-signed URLs: Temporary access
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'private-file.pdf'},
    ExpiresIn=3600  # 1 hora
)
# URL é válida apenas por 1 hora
```

**3. Versioning e Object Lock:**

```python
# Habilitar versioning (proteção contra deleção acidental)
s3.put_bucket_versioning(
    Bucket='my-important-bucket',
    VersioningConfiguration={'Status': 'Enabled'}
)

# Todas as versões são mantidas
s3.put_object(Bucket='my-important-bucket', Key='doc.txt', Body='version 1')
s3.put_object(Bucket='my-important-bucket', Key='doc.txt', Body='version 2')

# Listar versões
versions = s3.list_object_versions(Bucket='my-important-bucket', Prefix='doc.txt')
# Retorna: version 2 (current), version 1 (previous)

# Object Lock: WORM (Write Once Read Many)
# Compliance mode: Nem root pode deletar durante retention period
s3.put_object(
    Bucket='compliance-bucket',
    Key='audit-log-2023.json',
    Body=audit_data,
    ObjectLockMode='COMPLIANCE',
    ObjectLockRetainUntilDate=datetime(2030, 12, 31)
)

# Caso de uso: Regulamentações (HIPAA, SOX, FINRA)
# Logs não podem ser modificados ou deletados por 7 anos
```

---

## Relational Databases (Bancos de Dados Relacionais)

### Conceitos Fundamentais

Bancos de dados relacionais organizam dados em tabelas (relações) com linhas (registros) e colunas (atributos). Baseiam-se em teoria matemática de conjuntos e álgebra relacional.

**Princípios Fundamentais:**

1. **Estrutura Tabular**: Dados organizados em tabelas com schema definido
2. **Integridade Referencial**: Constraints garantem consistência entre tabelas
3. **ACID Properties**: Transações garantem confiabilidade
4. **SQL**: Linguagem declarativa padronizada para queries
5. **Normalization**: Redução de redundância através de decomposição

### Relational Database Concepts (Conceitos de Banco de Dados Relacional)

#### 1. ACID Properties

**Atomicity (Atomicidade):**
Transação é tudo ou nada - todas operações completam ou nenhuma completa.

```sql
-- Exemplo: Transferência bancária
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 100 WHERE account_id = 'A123';
    UPDATE accounts SET balance = balance + 100 WHERE account_id = 'B456';
COMMIT;

-- Se qualquer UPDATE falhar, ambos são revertidos (rollback)
-- Nunca teremos situação onde dinheiro desaparece ou duplica
```

**Consistency (Consistência):**
Transação leva banco de um estado válido para outro estado válido.

```sql
-- Constraint garante consistência
ALTER TABLE accounts ADD CONSTRAINT check_positive_balance 
    CHECK (balance >= 0);

-- Esta transação falhará se resultar em balance negativo
BEGIN TRANSACTION;
    UPDATE accounts SET balance = balance - 1000 WHERE account_id = 'A123';
    -- Se balance atual é 500, transação é rejeitada
ROLLBACK;
```

**Isolation (Isolamento):**
Transações concorrentes não interferem entre si.

```sql
-- Níveis de isolamento (do mais fraco ao mais forte):

-- 1. READ UNCOMMITTED: Pode ler dados não-committed (dirty reads)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;

-- 2. READ COMMITTED: Lê apenas dados committed
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- 3. REPEATABLE READ: Mesma query retorna mesmo resultado na transação
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

-- 4. SERIALIZABLE: Transações executam como se fossem sequenciais
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- Exemplo de problema sem isolamento (Lost Update):
-- Transaction 1:
BEGIN;
SELECT balance FROM accounts WHERE id = 'A123';  -- Lê 1000
-- (outro usuario modifica balance para 500)
UPDATE accounts SET balance = 1100 WHERE id = 'A123';  -- Sobrescreve para 1100
COMMIT;
-- Perdemos a atualização do outro usuário!

-- Com isolamento adequado:
BEGIN;
SELECT balance FROM accounts WHERE id = 'A123' FOR UPDATE;  -- Lock
-- Outro usuário espera até COMMIT
UPDATE accounts SET balance = balance + 100 WHERE id = 'A123';
COMMIT;
```

**Durability (Durabilidade):**
Dados committed sobrevivem a falhas (crash, power loss).

```sql
-- Após COMMIT, dados estão persistidos
BEGIN TRANSACTION;
    INSERT INTO orders VALUES (12345, 'customer', 'product', NOW());
COMMIT;
-- Mesmo se servidor crashar agora, order 12345 está salva

-- Implementação (Write-Ahead Logging):
-- 1. Escrever mudanças para transaction log (WAL)
-- 2. Confirmar ao cliente (COMMIT)
-- 3. Async: Aplicar mudanças aos datafiles
-- Se crash acontecer, log é replayed no restart
```

**Referência:** Härder, T., & Reuter, A. (1983). "Principles of Transaction-Oriented Database Recovery". *ACM Computing Surveys*, 15(4), 287-317.

#### 2. Normalization (Normalização)

Processo de organizar dados para reduzir redundância e dependências.

**First Normal Form (1NF):**
Eliminar grupos repetidos - cada célula contém valor atômico.

```sql
-- ❌ Não-1NF: Múltiplos valores em uma célula
CREATE TABLE orders_bad (
    order_id INT,
    customer_name VARCHAR(100),
    products VARCHAR(1000)  -- "product1, product2, product3"
);

-- ✓ 1NF: Um valor por célula
CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(100)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

**Second Normal Form (2NF):**
1NF + eliminar dependências parciais de chave composta.

```sql
-- ❌ Não-2NF: product_name depende apenas de product_id, não de (order_id, product_id)
CREATE TABLE order_items_bad (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),  -- Dependência parcial!
    quantity INT,
    PRIMARY KEY (order_id, product_id)
);

-- ✓ 2NF: Separar products em tabela própria
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    price DECIMAL(10,2)
);

CREATE TABLE order_items (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);
```

**Third Normal Form (3NF):**
2NF + eliminar dependências transitivas.

```sql
-- ❌ Não-3NF: city e state têm dependência transitiva via zip_code
CREATE TABLE customers_bad (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10),
    city VARCHAR(100),      -- Depende de zip_code
    state VARCHAR(2)        -- Depende de zip_code
);

-- ✓ 3NF: Separar informação de localização
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    name VARCHAR(100),
    zip_code VARCHAR(10),
    FOREIGN KEY (zip_code) REFERENCES zip_codes(zip_code)
);

CREATE TABLE zip_codes (
    zip_code VARCHAR(10) PRIMARY KEY,
    city VARCHAR(100),
    state VARCHAR(2)
);
```

**Denormalization para Performance:**

Às vezes, viola-se normalização intencionalmente para performance.

```sql
-- Denormalização: Duplicar dados para evitar JOINs

-- Normalizado (3NF): Requer JOIN
SELECT o.order_id, o.order_date, c.customer_name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;
-- Performance: 10-50ms (JOIN overhead)

-- Denormalizado: Duplicar customer_name em orders
CREATE TABLE orders_denorm (
    order_id INT PRIMARY KEY,
    customer_id INT,
    customer_name VARCHAR(100),  -- Duplicado!
    order_date DATE
);

SELECT order_id, order_date, customer_name
FROM orders_denorm;
-- Performance: 2-5ms (sem JOIN)

-- Trade-off:
-- - Performance: 5x mais rápido
-- - Redundância: customer_name duplicado
-- - Consistência: Deve atualizar em múltiplos lugares
```

#### 3. Indexes (Índices)

Estruturas de dados que aceleram buscas à custa de espaço e overhead de escrita.

**B-Tree Index (Padrão):**

```sql
-- Criar índice
CREATE INDEX idx_customer_email ON customers(email);

-- Query sem índice (Table Scan):
EXPLAIN SELECT * FROM customers WHERE email = 'john@example.com';
-- Seq Scan on customers (cost=0.00..100000.00 rows=1)
-- Escaneia toda tabela (milhões de rows)
-- Tempo: 5-20 segundos

-- Query com índice (Index Scan):
EXPLAIN SELECT * FROM customers WHERE email = 'john@example.com';
-- Index Scan using idx_customer_email (cost=0.42..8.44 rows=1)
-- Vai direto ao registro correto
-- Tempo: 2-10ms

-- Trade-off:
-- - Read: 1000x mais rápido
-- - Write: 10-20% mais lento (manter índice)
-- - Storage: +10-30% espaço
```

**Covering Index:**
Índice contém todas as colunas necessárias, evitando acesso à tabela.

```sql
-- Query precisa de: customer_id, email, name
SELECT customer_id, email, name 
FROM customers 
WHERE email = 'john@example.com';

-- Índice simples: Encontra no índice, depois busca na tabela
CREATE INDEX idx_email ON customers(email);
-- 2 disk accesses: índice + tabela

-- Covering index: Todas colunas no índice
CREATE INDEX idx_email_covering ON customers(email, name);
-- 1 disk access: apenas índice
-- Performance: 2x mais rápido
```

**Composite Index:**
Índice em múltiplas colunas.

```sql
-- Query com múltiplas condições
SELECT * FROM orders 
WHERE customer_id = 123 AND status = 'pending';

-- Índice composto
CREATE INDEX idx_customer_status ON orders(customer_id, status);

-- Ordem importa!
-- ✓ Usa índice: WHERE customer_id = 123 AND status = 'pending'
-- ✓ Usa índice: WHERE customer_id = 123
-- ❌ Não usa índice: WHERE status = 'pending'
-- (Ordem é customer_id primeiro, depois status)

-- Regra: Índice composto (a, b, c) funciona para:
-- - WHERE a = ... AND b = ... AND c = ...
-- - WHERE a = ... AND b = ...
-- - WHERE a = ...
-- Mas NÃO funciona para:
-- - WHERE b = ...
-- - WHERE c = ...
```

**Partial Index:**
Índice apenas em subset dos dados.

```sql
-- Índice apenas para orders ativas (economia de espaço)
CREATE INDEX idx_active_orders ON orders(order_date)
WHERE status != 'completed';

-- 90% dos orders são completed
-- Índice é 10% do tamanho de índice completo
-- Performance: Igual para queries de orders ativos
-- Storage: 90% de economia
```

**Referência:** Comer, D. (1979). "The Ubiquitous B-Tree". *ACM Computing Surveys*, 11(2), 121-137.



### Relational Database Management System Architecture

#### 1. Query Processor

O Query Processor é responsável por interpretar, otimizar e executar queries SQL.

**Componentes:**

```
SQL Query
    ↓
[Parser] → Syntax Tree
    ↓
[Query Rewriter] → Canonical Form
    ↓
[Query Optimizer] → Execution Plan
    ↓
[Query Executor] → Results
```

**Query Optimization Example:**

```sql
-- Query original
SELECT c.name, o.order_date, o.total
FROM customers c
JOIN orders o ON c.customer_id = o.customer_id
WHERE c.country = 'USA' AND o.total > 1000;

-- Optimizer avalia múltiplos planos:

-- Plano 1: Filter primeiro, depois JOIN
-- Cost: 100 (melhor)
1. Index scan on customers WHERE country = 'USA' (100 rows)
2. Index scan on orders WHERE total > 1000 (500 rows)
3. Hash JOIN on customer_id (50 matching rows)

-- Plano 2: JOIN primeiro, depois filter
-- Cost: 10,000 (pior)
1. Full table scan on customers (1M rows)
2. Full table scan on orders (10M rows)
3. Hash JOIN (produces 8M rows)
4. Filter WHERE country = 'USA' AND total > 1000 (50 rows)

-- Optimizer escolhe Plano 1 (100x mais eficiente)
```

**Cost-Based Optimization:**

```sql
-- Optimizer usa estatísticas para estimar custo

-- Atualizar estatísticas (importante!)
ANALYZE customers;
ANALYZE orders;

-- Ver plano de execução
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123;

-- Output:
-- Index Scan using idx_customer on orders
-- (cost=0.42..8.44 rows=1 width=100) 
-- (actual time=0.025..0.027 rows=1 loops=1)
-- Planning Time: 0.123 ms
-- Execution Time: 0.052 ms

-- Métricas:
-- - cost: Estimativa de recursos necessários
-- - rows: Número estimado de rows
-- - actual time: Tempo real de execução
-- - loops: Quantas vezes operação foi executada
```

**Referência:** Selinger, P. G. et al. (1979). "Access Path Selection in a Relational Database Management System". *ACM SIGMOD*, 23-34.

#### 2. Storage Engine

Gerencia como dados são armazenados fisicamente em disco e memória.

**Buffer Pool (Cache):**

```python
# Conceito: Cache de páginas de disco em memória
# Tamanho típico: 60-80% da RAM disponível

# PostgreSQL configuration:
shared_buffers = 8GB  # 25% of RAM (32GB total)
effective_cache_size = 24GB  # 75% of RAM

# Quando query acessa data:
# 1. Verifica se página está no buffer pool (cache hit)
# 2. Se não, carrega do disco (cache miss)
# 3. Mantém páginas quentes (frequently accessed) em cache
# 4. Evict páginas frias (rarely accessed) quando cheio

# Métricas:
# - Cache hit ratio: 99%+ é ideal
# - Cache miss: Requer disk I/O (10,000x mais lento)

SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
FROM pg_statio_user_tables;

-- cache_hit_ratio de 0.99 = 99% cache hit rate (excelente)
-- cache_hit_ratio de 0.80 = 80% cache hit rate (adicionar mais RAM)
```

**Page Structure:**

```
Página de disco (tipicamente 8KB):
[Page Header: 24 bytes]
    - PageHeaderData: LSN, checksum, flags
[Item Pointers: 4 bytes cada]
    - Offsets para tuples na página
[Free Space]
    - Espaço disponível para novos tuples
[Tuples]
    - Registros reais de dados
[Special Space]
    - Usado por índices (opcional)
```

**Write-Ahead Logging (WAL):**

```sql
-- Processo de escrita durável:

-- 1. Transaction começa
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 'A123';

-- 2. Modificações escritas no WAL buffer
-- WAL entry: {LSN: 1000, TXN: 42, Operation: UPDATE, ...}

-- 3. COMMIT force-writes WAL para disco
COMMIT;
-- fsync() garante que WAL está em disco físico
-- Após isso, transaction é considerada committed

-- 4. Async: Modificações aplicadas aos datafiles (checkpoint)
-- Não bloqueia transaction - pode acontecer depois

-- 5. Recovery após crash:
-- - Ler WAL desde último checkpoint
-- - Replay todas transactions committed
-- - Discard transactions não-committed
```

**Referência:** Mohan, C. et al. (1992). "ARIES: A Transaction Recovery Method". *ACM Transactions on Database Systems*, 17(1), 94-162.

#### 3. Transaction Manager

Gerencia concorrência e recuperação de transações.

**Multi-Version Concurrency Control (MVCC):**

PostgreSQL e muitos bancos modernos usam MVCC para permitir alta concorrência sem locks.

```sql
-- Conceito: Múltiplas versões do mesmo row coexistem

-- Transaction 1: SELECT (time 0)
BEGIN;
SELECT balance FROM accounts WHERE id = 'A123';
-- Vê versão 1: balance = 1000

-- Transaction 2: UPDATE (time 1)
BEGIN;
UPDATE accounts SET balance = 500 WHERE id = 'A123';
-- Cria versão 2: balance = 500
-- Versão 1 ainda existe para Transaction 1!
COMMIT;

-- Transaction 1: SELECT novamente (time 2)
SELECT balance FROM accounts WHERE id = 'A123';
-- REPEATABLE READ: Ainda vê versão 1 (balance = 1000)
-- READ COMMITTED: Vê versão 2 (balance = 500)
COMMIT;

-- Vacuum process: Limpa versões antigas não mais necessárias
```

**Lock Management:**

```sql
-- Tipos de locks em PostgreSQL:

-- 1. Access Share Lock (SELECT)
-- - Apenas conflita com EXCLUSIVE lock
-- - Múltiplas transactions podem ler simultaneamente

-- 2. Row Share Lock (SELECT ... FOR UPDATE)
-- - Previne outros de deletar ou atualizar rows
BEGIN;
SELECT * FROM accounts WHERE id = 'A123' FOR UPDATE;
-- Row está locked até COMMIT

-- 3. Row Exclusive Lock (INSERT, UPDATE, DELETE)
-- - Conflita com EXCLUSIVE lock
UPDATE accounts SET balance = 500 WHERE id = 'A123';

-- 4. Exclusive Lock (ALTER TABLE, DROP TABLE)
-- - Conflita com tudo
ALTER TABLE accounts ADD COLUMN new_col INT;
-- Tabela completamente locked

-- Deadlock Detection:
-- Transaction 1:
BEGIN;
UPDATE accounts SET balance = 1000 WHERE id = 'A123';  -- Lock A123
-- Esperando...
UPDATE accounts SET balance = 2000 WHERE id = 'B456';  -- Tenta lock B456

-- Transaction 2:
BEGIN;
UPDATE accounts SET balance = 3000 WHERE id = 'B456';  -- Lock B456
-- Esperando...
UPDATE accounts SET balance = 4000 WHERE id = 'A123';  -- Tenta lock A123

-- Deadlock detectado!
-- Database escolhe uma transaction para abortar (rollback)
-- ERROR: deadlock detected
```

---

### Optimizing Relational Databases

#### 1. Query Optimization Techniques

**Avoid SELECT ***

```sql
-- ❌ Ruim: Busca todas as colunas
SELECT * FROM products WHERE category = 'electronics';
-- Transfere 500 bytes por row × 10,000 rows = 5 MB

-- ✓ Bom: Busca apenas necessário
SELECT product_id, name, price FROM products WHERE category = 'electronics';
-- Transfere 100 bytes por row × 10,000 rows = 1 MB
-- Speedup: 5x
```

**Use Indexes Effectively:**

```sql
-- ❌ Ruim: Função na coluna indexed impede uso do índice
SELECT * FROM users WHERE UPPER(email) = 'JOHN@EXAMPLE.COM';
-- Faz table scan mesmo com índice em email

-- ✓ Bom: Usar expression index ou comparar sem função
CREATE INDEX idx_email_upper ON users(UPPER(email));
-- Agora pode usar índice

-- Ou melhor: Normalizar dados (sempre lowercase)
SELECT * FROM users WHERE email = 'john@example.com';
```

**Avoid N+1 Query Problem:**

```python
# ❌ Ruim: N+1 queries
orders = db.query("SELECT * FROM orders LIMIT 100")
for order in orders:
    customer = db.query("SELECT * FROM customers WHERE id = %s", (order.customer_id,))
    # 1 query para orders + 100 queries para customers = 101 queries
# Time: 100 × 5ms = 500ms

# ✓ Bom: JOIN ou bulk fetch
orders = db.query("""
    SELECT o.*, c.name as customer_name, c.email as customer_email
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id
    LIMIT 100
""")
# 1 query total
# Time: 10ms
# Speedup: 50x
```

**Use Query Caching:**

```python
import redis
import hashlib

class QueryCache:
    def __init__(self):
        self.redis = redis.Redis()
        self.ttl = 300  # 5 minutos
    
    def cached_query(self, sql, params):
        # Gerar cache key do SQL + params
        # MD5 é usado aqui apenas para geração rápida de chave de cache, não para fins de segurança/criptografia.
        cache_key = hashlib.md5(f"{sql}{params}".encode()).hexdigest()
        
        # Tentar buscar do cache
        cached = self.redis.get(f"query:{cache_key}")
        if cached:
            return json.loads(cached)
        
        # Cache miss: Executar query
        result = db.execute(sql, params)
        
        # Armazenar em cache
        self.redis.setex(f"query:{cache_key}", self.ttl, json.dumps(result))
        
        return result

# Performance:
# - Cache hit: 1-2ms (Redis)
# - Cache miss: 50-200ms (Database)
# - Hit rate típico: 80-95%
# - Tempo médio: 0.8 × 2ms + 0.2 × 100ms = 21.6ms
# vs Sem cache: 100ms
# Speedup: 4.6x
```

#### 2. Index Optimization

**Analyze Index Usage:**

```sql
-- PostgreSQL: Ver índices não utilizados
SELECT
    schemaname,
    tablename,
    indexname,
    idx_scan,  -- Número de vezes usado
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE idx_scan = 0  -- Nunca usado!
ORDER BY pg_relation_size(indexrelid) DESC;

-- Índices não utilizados desperdiçam:
-- - Espaço em disco
-- - RAM (buffer pool)
-- - Performance de writes (manter índice)

-- Remover índices não utilizados:
DROP INDEX idx_unused;
```

**Partial Index para Queries Específicas:**

```sql
-- 90% dos orders são "completed"
-- Queries quase sempre filtram por outros status

-- ❌ Ruim: Índice completo
CREATE INDEX idx_orders_status ON orders(status);
-- Tamanho: 1 GB

-- ✓ Bom: Índice parcial (apenas status != 'completed')
CREATE INDEX idx_orders_active_status ON orders(status)
WHERE status != 'completed';
-- Tamanho: 100 MB
-- Queries em orders ativos: Mesma performance
-- Economia: 90% espaço, writes 90% mais rápidos
```

**Index Maintenance:**

```sql
-- Índices B-Tree podem fragmentar ao longo do tempo
-- Especialmente após muitos UPDATEs/DELETEs

-- Ver bloat de índice (PostgreSQL):
SELECT
    schemaname,
    tablename,
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY pg_relation_size(indexrelid) DESC;

-- Rebuild índice fragmentado:
REINDEX INDEX idx_customers_email;

-- Ou rebuild todos índices de uma tabela:
REINDEX TABLE customers;

-- Schedule regular reindexing:
-- - Diário para tabelas write-heavy
-- - Semanal para tabelas normais
-- - Mensal para tabelas read-mostly
```

#### 3. Connection Pooling

**Problema: Connection Overhead**

```python
# ❌ Ruim: Nova conexão para cada request
def handle_request():
    # Criar conexão: 50-200ms!
    conn = psycopg2.connect(
        host="db.example.com",
        database="mydb",
        user="app",
        password="secret"
    )
    
    # Query: 5ms
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = 123")
    result = cursor.fetchone()
    
    # Close: 10ms
    conn.close()
    
    return result

# Total time: 50-200ms para conexão + 5ms para query = 55-205ms
# Desperdício: 90%+ do tempo é overhead de conexão
```

**Solução: Connection Pool**

```python
# ✓ Bom: Pool de conexões reutilizáveis
from psycopg2 import pool

# Criar pool na inicialização
connection_pool = pool.ThreadedConnectionPool(
    minconn=10,   # Conexões sempre abertas
    maxconn=100,  # Máximo de conexões
    host="db.example.com",
    database="mydb",
    user="app",
    password="secret"
)

def handle_request():
    # Pegar conexão do pool: 0.1ms (já está aberta!)
    conn = connection_pool.getconn()
    
    try:
        # Query: 5ms
        cursor = conn.cursor()
        cursor.execute("SELECT * FROM users WHERE id = 123")
        result = cursor.fetchone()
        return result
    finally:
        # Retornar ao pool (não fecha): 0.1ms
        connection_pool.putconn(conn)

# Total time: 0.1ms + 5ms + 0.1ms = 5.2ms
# vs Sem pool: 55-205ms
# Speedup: 10-40x
```

**PgBouncer - External Connection Pooler:**

```ini
# pgbouncer.ini
[databases]
mydb = host=db.example.com port=5432 dbname=mydb

[pgbouncer]
listen_addr = *
listen_port = 6432
auth_type = md5
auth_file = /etc/pgbouncer/userlist.txt
pool_mode = transaction  # Reutiliza conexão entre transactions
max_client_conn = 10000  # Aplicação pode ter 10k conexões
default_pool_size = 100  # Mas apenas 100 conexões reais ao DB

# Benefícios:
# - 10,000 app connections → 100 DB connections
# - Reduz carga no database
# - Permite scaling horizontal de apps sem escalar DB
```

#### 4. Partitioning

**Conceito:**
Dividir tabela grande em múltiplas tabelas menores (partições).

**Range Partitioning por Data:**

```sql
-- Tabela de logs com bilhões de rows
-- Particionar por mês

-- Tabela pai (vazia, apenas schema)
CREATE TABLE logs (
    log_id BIGSERIAL,
    timestamp TIMESTAMP NOT NULL,
    level VARCHAR(10),
    message TEXT
) PARTITION BY RANGE (timestamp);

-- Partições (uma por mês)
CREATE TABLE logs_2023_01 PARTITION OF logs
    FOR VALUES FROM ('2023-01-01') TO ('2023-02-01');

CREATE TABLE logs_2023_02 PARTITION OF logs
    FOR VALUES FROM ('2023-02-01') TO ('2023-03-01');

CREATE TABLE logs_2023_03 PARTITION OF logs
    FOR VALUES FROM ('2023-03-01') TO ('2023-04-01');
-- ... etc

-- Query automáticamente usa apenas partições relevantes
SELECT * FROM logs WHERE timestamp >= '2023-02-15' AND timestamp < '2023-02-20';
-- Partition pruning: Apenas logs_2023_02 é escaneada
-- Speedup: 100x (escaneia 1 mês ao invés de anos de dados)

-- Manutenção: Drop partições antigas
DROP TABLE logs_2022_01;  -- Remove dados de Jan 2022
-- Muito mais rápido que DELETE (que precisa escanear tudo)
```

**List Partitioning por Categoria:**

```sql
-- Particionar produtos por categoria
CREATE TABLE products (
    product_id INT,
    name VARCHAR(200),
    category VARCHAR(50),
    price DECIMAL(10,2)
) PARTITION BY LIST (category);

CREATE TABLE products_electronics PARTITION OF products
    FOR VALUES IN ('electronics', 'computers', 'phones');

CREATE TABLE products_clothing PARTITION OF products
    FOR VALUES IN ('clothing', 'shoes', 'accessories');

CREATE TABLE products_food PARTITION OF products
    FOR VALUES IN ('food', 'beverages', 'snacks');

-- Query por categoria usa apenas partição relevante
SELECT * FROM products WHERE category = 'electronics';
-- Escaneia apenas products_electronics
```

**Hash Partitioning para Distribuição Uniforme:**

```sql
-- Particionar users por hash de user_id
CREATE TABLE users (
    user_id BIGINT PRIMARY KEY,
    email VARCHAR(200),
    created_at TIMESTAMP
) PARTITION BY HASH (user_id);

-- 16 partições
CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 16, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 16, REMAINDER 1);
-- ... até p15

-- Benefícios:
-- - Distribui dados uniformemente
-- - Queries paralelas em múltiplas partições
-- - I/O distribuído (menos contenção)
```

---

### Scaling Relational Databases

#### 1. Vertical Scaling (Scale Up)

**Conceito:**
Aumentar recursos da máquina (CPU, RAM, disco).

**Amazon RDS Instance Types:**

```python
# Progression de vertical scaling:

# Workloads muito pequenos ou desenvolvimento
db.t3.micro = {
    'vcpu': 2,
    'ram_gb': 1,
    'cost_monthly': 15,
    'throughput': '100 QPS'
}

# Produção pequena
db.t3.large = {
    'vcpu': 2,
    'ram_gb': 8,
    'cost_monthly': 124,
    'throughput': '1,000 QPS'
}

# Produção média
db.m5.4xlarge = {
    'vcpu': 16,
    'ram_gb': 64,
    'cost_monthly': 1,338,
    'throughput': '10,000 QPS'
}

# Produção alta
db.r5.16xlarge = {
    'vcpu': 64,
    'ram_gb': 512,
    'cost_monthly': 9,216,
    'throughput': '100,000 QPS'
}

# Máximo (para workloads extremos)
db.r5.24xlarge = {
    'vcpu': 96,
    'ram_gb': 768,
    'cost_monthly': 13,824,
    'throughput': '150,000+ QPS'
}

# Trade-offs:
# Vantagens:
# - Simples (apenas mudar instance type)
# - Sem mudanças na aplicação
# - Performance melhor por custo até certo ponto

# Desvantagens:
# - Limite físico (máximo 96 vCPUs, 768 GB RAM)
# - Custo não-linear (dobrar performance pode triplicar custo)
# - Single point of failure
# - Downtime para upgrade (Multi-AZ reduz mas não elimina)
```

**Quando Vertical Scaling é Suficiente:**

- Aplicações até ~100K QPS
- Dataset cabe em RAM de instância grande (até 768 GB)
- Transações requerem strong consistency
- Orçamento permite instâncias grandes
- Simplicidade operacional é prioridade

#### 2. Read Replicas

**Conceito:**
Replicação assíncrona para distribuir carga de leitura.

**Arquitetura:**

```
                    [Primary/Master]
                    (Writes apenas)
                    ↓  ↓  ↓  (replicação)
        ┌───────────┼──────────┬──────────┐
        ↓           ↓          ↓          ↓
   [Replica 1] [Replica 2] [Replica 3] [Replica 4]
   (Reads)     (Reads)     (Reads)     (Reads)
```

**Implementação com Amazon RDS:**

```python
import boto3

rds = boto3.client('rds')

# Criar read replica
rds.create_db_instance_read_replica(
    DBInstanceIdentifier='mydb-replica-1',
    SourceDBInstanceIdentifier='mydb-primary',
    DBInstanceClass='db.r5.2xlarge',
    AvailabilityZone='us-east-1b',
    PubliclyAccessible=False,
    Tags=[{'Key': 'Purpose', 'Value': 'ReadReplica'}]
)

# Application code: Separar reads e writes
class DatabaseRouter:
    def __init__(self):
        self.primary = psycopg2.connect(host='mydb-primary.rds.amazonaws.com')
        self.replicas = [
            psycopg2.connect(host='mydb-replica-1.rds.amazonaws.com'),
            psycopg2.connect(host='mydb-replica-2.rds.amazonaws.com'),
            psycopg2.connect(host='mydb-replica-3.rds.amazonaws.com'),
        ]
        self.replica_index = 0
    
    def execute_write(self, sql, params):
        """Writes sempre vão para primary"""
        cursor = self.primary.cursor()
        cursor.execute(sql, params)
        self.primary.commit()
    
    def execute_read(self, sql, params):
        """Reads distribuídos entre replicas (round-robin)"""
        replica = self.replicas[self.replica_index]
        self.replica_index = (self.replica_index + 1) % len(self.replicas)
        
        cursor = replica.cursor()
        cursor.execute(sql, params)
        return cursor.fetchall()

# Scaling capacity:
# - 1 primary: 10,000 writes/s
# - 3 replicas: 10,000 × 3 = 30,000 reads/s
# - Total: 10K writes + 30K reads = 40K QPS

# Workload típico (read-heavy):
# - 90% reads, 10% writes
# - Com 3 replicas: 9K writes + 90K reads = 99K QPS possível
# Speedup: 10x
```

**Replication Lag:**

```python
# Monitorar replica lag
def check_replication_lag():
    # CloudWatch metric
    cloudwatch = boto3.client('cloudwatch')
    
    response = cloudwatch.get_metric_statistics(
        Namespace='AWS/RDS',
        MetricName='ReplicaLag',
        Dimensions=[
            {'Name': 'DBInstanceIdentifier', 'Value': 'mydb-replica-1'}
        ],
        StartTime=datetime.now() - timedelta(minutes=5),
        EndTime=datetime.now(),
        Period=60,
        Statistics=['Average']
    )
    
    lag_seconds = response['Datapoints'][0]['Average']
    
    # Lag típico: 0.1-2 segundos
    # Lag alto (>10s): Indica problema
    
    if lag_seconds > 10:
        # Possíveis causas:
        # - Primary overloaded (muitos writes)
        # - Replica underpowered (smaller instance)
        # - Network issues
        # - Long-running transactions no primary
        
        # Soluções:
        # - Upgrade replica instance type
        # - Adicionar mais replicas
        # - Otimizar queries write-heavy
        pass
    
    return lag_seconds

# Trade-off: Eventual Consistency
# - Replica pode estar seconds atrás do primary
# - Read pode retornar dados desatualizados
# - Aplicação deve tolerar eventual consistency
# - Para strong consistency: Ler do primary
```

**Cross-Region Read Replicas:**

```python
# Criar replica em outra região (para DR e latência global)
rds.create_db_instance_read_replica(
    DBInstanceIdentifier='mydb-replica-eu',
    SourceDBInstanceIdentifier='mydb-primary',
    SourceRegion='us-east-1',  # Primary region
    DBInstanceClass='db.r5.2xlarge',
    # Esta replica está em região diferente
    # Benefícios:
    # - Disaster Recovery: Pode promover para primary se US falhar
    # - Latência: Usuários na Europa leem de replica local
    # - Global distribution de leitura
)

# Latência de replicação cross-region: 1-5 segundos
# (maior que same-region devido à distância)
```

#### 3. Sharding (Horizontal Partitioning)

**Conceito:**
Dividir dados entre múltiplos bancos de dados independentes.

**Sharding por User ID:**

```python
class ShardedDatabase:
    """
    Implementação de sharding por user_id
    """
    def __init__(self, num_shards=4):
        self.num_shards = num_shards
        self.shards = [
            psycopg2.connect(host=f'shard{i}.example.com')
            for i in range(num_shards)
        ]
    
    def get_shard(self, user_id):
        """Determinar shard baseado em user_id"""
        shard_id = user_id % self.num_shards
        return self.shards[shard_id]
    
    def get_user(self, user_id):
        """Get user do shard correto"""
        shard = self.get_shard(user_id)
        cursor = shard.cursor()
        cursor.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
        return cursor.fetchone()
    
    def get_user_orders(self, user_id):
        """Get orders do shard correto"""
        shard = self.get_shard(user_id)
        cursor = shard.cursor()
        cursor.execute("SELECT * FROM orders WHERE user_id = %s", (user_id,))
        return cursor.fetchall()
    
    def get_global_stats(self):
        """
        Agregação cross-shard é cara!
        Precisa query em todos os shards
        """
        results = []
        for shard in self.shards:
            cursor = shard.cursor()
            cursor.execute("SELECT COUNT(*) FROM users")
            results.append(cursor.fetchone()[0])
        
        total_users = sum(results)
        return {'total_users': total_users}

# Scaling:
# - 1 shard: 10,000 writes/s
# - 4 shards: 40,000 writes/s
# - 10 shards: 100,000 writes/s
# Escalabilidade quase linear

# Trade-offs:
# Vantagens:
# - Escalabilidade horizontal ilimitada
# - Cada shard é independente (fault isolation)
# - Custo-efetivo (usar instances menores)

# Desvantagens:
# - Complexidade: Aplicação gerencia routing
# - Rebalancing: Adicionar shards requer migração
# - Cross-shard queries: Difíceis ou impossíveis
# - Transactions cross-shard: Muito complexas
# - JOINs cross-shard: Impossíveis
```

**Sharding Key Selection:**

```python
# Escolha do shard key é crítica!

# ❌ Ruim: Data como shard key
# - Shard atual recebe TODOS os writes (hot shard)
# - Shards antigos ficam inativos (cold shards)
# - Desbalanceamento severo

# ✓ Bom: User ID como shard key
# - Distribuição uniforme de writes
# - Dados de um usuário sempre no mesmo shard (locality)
# - Fácil de query dados de um usuário

# ✓ Bom: Hash de composite key
def get_shard_key(user_id, tenant_id):
    # Para aplicações multi-tenant
    # Shard por combinação de user + tenant
    combined = f"{tenant_id}:{user_id}"
    hash_value = hashlib.md5(combined.encode()).hexdigest()
    shard_id = int(hash_value, 16) % num_shards
    return shard_id
```

**Consistent Hashing para Rebalancing:**

```python
import hashlib

class ConsistentHashRing:
    """
    Consistent hashing minimiza redistribuição ao adicionar shards
    """
    def __init__(self, shards, virtual_nodes=150):
        self.ring = {}
        self.sorted_keys = []
        self.shards = shards
        
        # Criar virtual nodes para melhor distribuição
        for shard in shards:
            for i in range(virtual_nodes):
                key = self._hash(f"{shard}:{i}")
                self.ring[key] = shard
        
        self.sorted_keys = sorted(self.ring.keys())
    
    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)
    
    def get_shard(self, item_key):
        """Encontrar shard para um key"""
        if not self.ring:
            return None
        
        hash_val = self._hash(str(item_key))
        
        # Busca binária para encontrar primeiro shard >= hash_val
        idx = bisect.bisect_right(self.sorted_keys, hash_val)
        if idx == len(self.sorted_keys):
            idx = 0
        
        return self.ring[self.sorted_keys[idx]]
    
    def add_shard(self, new_shard):
        """Adicionar novo shard"""
        # Com consistent hashing:
        # - Apenas 1/N dos keys precisam mover
        # - N = número total de shards
        # - Muito melhor que rehash completo
        
        for i in range(150):
            key = self._hash(f"{new_shard}:{i}")
            self.ring[key] = new_shard
        
        self.sorted_keys = sorted(self.ring.keys())
        self.shards.append(new_shard)

# Exemplo:
ring = ConsistentHashRing(['shard0', 'shard1', 'shard2', 'shard3'])

# Adicionar 5º shard: Apenas 20% dos dados movem (vs 100% com modulo hashing)
ring.add_shard('shard4')
```

#### 4. Database Clustering

**Amazon Aurora - Shared Storage Architecture:**

```
                [Writer Instance]
                (Primary - All writes)
                        ↓
                [Aurora Storage]
                (Shared by all instances)
                (6 copies across 3 AZs)
                    ↓   ↓   ↓
        [Reader 1] [Reader 2] [Reader 3]
        (Read-only instances)

# Vantagens do Aurora:
# - Storage compartilhado (não replicação de dados)
# - Replication lag: Tipicamente <10ms (vs segundos no RDS)
# - Auto-scaling de storage: 10GB → 128TB automático
# - Até 15 read replicas (vs 5 no RDS)
# - Failover rápido: <30 segundos (vs 1-2 minutos)
```

**Amazon Aurora Serverless:**

```python
# Aurora Serverless: Auto-scaling completo
aurora_serverless_config = {
    'MinCapacity': 2,   # 2 ACUs = 4 GB RAM
    'MaxCapacity': 64,  # 64 ACUs = 128 GB RAM
    'Auto Pause': True,  # Pause após inatividade (economia)
    'Seconds Until Auto Pause': 300,  # 5 minutos
    'Cost': 'Pay per second of usage'
}

# Use cases:
# - Workloads intermitentes (dev/test)
# - Aplicações com tráfego imprevisível
# - Startup com budget limitado

# Custo comparison:
# Aurora Provisioned (db.r5.large): $330/mês sempre ligado
# Aurora Serverless: $0/mês quando pausado, ~$50-150/mês uso típico
# Economia: 50-100%
```

---

### Open Source Relational Database Systems

#### 1. PostgreSQL

**Características:**
- ACID completo, MVCC, transações robustas
- Extensível (custom types, functions, operators)
- JSON/JSONB nativo (NoSQL-like em RDBMS)
- Full-text search, GIS (PostGIS)
- Comunidade ativa, excelente documentação

**Use Cases:**
- Aplicações complexas com queries avançadas
- Data warehousing (com extensões como Citus)
- GIS applications
- Time-series data (com TimescaleDB)

**AWS: Amazon RDS for PostgreSQL e Amazon Aurora PostgreSQL**

```python
# Criar RDS PostgreSQL
rds.create_db_instance(
    DBInstanceIdentifier='my-postgres',
    DBInstanceClass='db.t3.medium',
    Engine='postgres',
    EngineVersion='15.3',
    MasterUsername='admin',
    MasterUserPassword=get_password_from_secrets_manager(),  # Não hardcode senhas; use um secrets manager
    AllocatedStorage=100,
    StorageType='gp3',
    StorageEncrypted=True,
    BackupRetentionPeriod=7,
    MultiAZ=True  # Alta disponibilidade
)

# Features do PostgreSQL:
# - JSONB: Store e query JSON eficientemente
SELECT * FROM products WHERE metadata @> '{"color": "red"}';

# - Full-text search:
SELECT * FROM articles WHERE search_vector @@ to_tsquery('postgresql & database');

# - Window functions:
SELECT 
    product_name,
    price,
    AVG(price) OVER (PARTITION BY category) as avg_category_price
FROM products;

# - Common Table Expressions (CTEs) recursivos:
WITH RECURSIVE employee_tree AS (
    SELECT id, name, manager_id, 1 as level
    FROM employees
    WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, et.level + 1
    FROM employees e
    JOIN employee_tree et ON e.manager_id = et.id
)
SELECT * FROM employee_tree;
```

#### 2. MySQL

**Características:**
- Performance excelente para workloads read-heavy
- Replicação master-slave simples e efetiva
- Storage engines pluggáveis (InnoDB, MyISAM)
- Amplamente utilizado (WordPress, Drupal)
- Fácil de aprender e usar

**Use Cases:**
- Aplicações web (LAMP stack)
- E-commerce
- Content Management Systems
- Aplicações read-heavy

**AWS: Amazon RDS for MySQL e Amazon Aurora MySQL**

```python
# Criar RDS MySQL
rds.create_db_instance(
    DBInstanceIdentifier='my-mysql',
    DBInstanceClass='db.t3.medium',
    Engine='mysql',
    EngineVersion='8.0.33',
    MasterUsername='admin',
    MasterUserPassword='SecurePassword123!',
    AllocatedStorage=100,
    StorageType='gp3'
)

# MySQL-specific features:
# - Storage engines:
CREATE TABLE users_innodb (id INT PRIMARY KEY) ENGINE=InnoDB;  # Transações
CREATE TABLE cache_myisam (id INT PRIMARY KEY) ENGINE=MyISAM;  # Read-only fast

# - Performance Schema:
SELECT * FROM performance_schema.events_statements_summary_by_digest
ORDER BY SUM_TIMER_WAIT DESC LIMIT 10;
# Identifica queries lentas automaticamente
```

#### 3. MariaDB

**Características:**
- Fork do MySQL (criado por criador original do MySQL)
- Compatível com MySQL (drop-in replacement)
- Features adicionais (window functions, JSON)
- Governance open-source mais transparente

**AWS: Amazon RDS for MariaDB**

```python
rds.create_db_instance(
    DBInstanceIdentifier='my-mariadb',
    Engine='mariadb',
    EngineVersion='10.11',
    # Resto igual ao MySQL
)

# MariaDB extras:
# - Galera Cluster: Multi-master replication
# - ColumnStore: Column-oriented storage engine
# - Spider: Sharding engine nativo
```

---

## Conclusion (Conclusão)

### Resumo dos Conceitos Chave

Este capítulo abordou de forma abrangente os tipos de armazenamento de dados e bancos de dados relacionais, fundamentos essenciais para design de sistemas distribuídos.

**Data Storage Formats:**
- **Row-Oriented**: Ideal para OLTP (transações)
- **Column-Oriented**: Ideal para OLAP (analytics)
- **Log-Structured**: Ideal para writes sequenciais (logs, time-series)
- **Key-Value**: Ideal para latência ultra-baixa (cache, sessions)

**Storage Types:**
- **File-Based** (EFS, FSx): Compartilhamento entre instâncias, compatibilidade POSIX
- **Block-Based** (EBS): Performance máxima, low-latency, databases
- **Object-Based** (S3): Escalabilidade massiva, durabilidade, cost-effective

**Relational Databases:**
- **ACID Properties**: Garantias de transações confiáveis
- **Normalization**: Redução de redundância
- **Indexes**: Trade-off entre read performance e write overhead
- **Query Optimization**: Técnicas para melhorar performance

**Scaling Strategies:**
- **Vertical Scaling**: Simples mas limitado
- **Read Replicas**: Distribuir carga de leitura (até 15x capacity)
- **Sharding**: Escalabilidade horizontal ilimitada (complexidade alta)
- **Database Clustering**: Aurora para best of both worlds

### Princípios de Decisão

**Escolher Storage Type baseado em:**

1. **Pattern de Acesso:**
   - Random I/O: Block Storage (EBS)
   - Sequential I/O: File Storage (FSx)
   - HTTP API: Object Storage (S3)

2. **Requisitos de Performance:**
   - Ultra-low latency (<1ms): Block (io2 Block Express)
   - High throughput (GB/s): File (FSx for Lustre)
   - Scalability massiva: Object (S3)

3. **Modelo de Custo:**
   - Sempre acessado: Block/File
   - Acesso infrequente: S3 Intelligent-Tiering
   - Archive: S3 Glacier

**Escolher Database Strategy baseado em:**

1. **Workload Characteristics:**
   - Read-heavy (90%+ reads): Use read replicas
   - Write-heavy: Use sharding ou Aurora
   - Analytical: Use column-store (Redshift) ou materialized views

2. **Consistency Requirements:**
   - Strong consistency: Reads do primary
   - Eventual consistency OK: Reads de replicas

3. **Scale Requirements:**
   - < 100K QPS: Vertical scaling suficiente
   - 100K-500K QPS: Read replicas (5-15 replicas)
   - > 500K QPS: Sharding necessário

### Perguntas para Entrevistas Técnicas

#### Pergunta 1: Design de Sistema de E-commerce

**Pergunta:**
"Design o backend de um e-commerce que processa 100K orders/dia, tem 10M de produtos, e precisa garantir que inventory nunca sobrevenda. Escolha e justifique storage types e database strategy."

**Resposta Esperada:**

```
Arquitetura proposta:

1. Product Catalog:
   - Storage: S3 + CloudFront
   - Database: Aurora MySQL com read replicas
   - Justificativa:
     * Imagens em S3 (cheaper, scalable)
     * Metadata em Aurora (relational, ACID)
     * 5 read replicas para distribuir carga (read-heavy)

2. Inventory Management:
   - Database: Aurora PostgreSQL (primary only)
   - Justificativa:
     * ACID crítico (zero overselling)
     * Serializable isolation level
     * Pessimistic locking em inventory checks

3. Orders:
   - Database: Aurora MySQL sharded por customer_id
   - Justificativa:
     * 100K orders/dia = 1.2 writes/segundo (light)
     * Mas reads de order history são frequentes
     * Shard por customer_id para scale futuro

4. Session/Cart:
   - Storage: ElastiCache (Redis)
   - Justificativa:
     * Temporary data (TTL)
     * Sub-millisecond latency
     * High throughput

Implementation:

```python
class EcommerceBackend:
    def place_order(self, customer_id, items):
        # 1. Reserve inventory (ACID transaction)
        with aurora_primary.transaction(isolation='SERIALIZABLE'):
            for item in items:
                inventory = db.query(
                    "SELECT quantity FROM inventory WHERE product_id = %s FOR UPDATE",
                    item.product_id
                )
                
                if inventory.quantity < item.quantity:
                    raise OutOfStockError()
                
                db.execute(
                    "UPDATE inventory SET quantity = quantity - %s WHERE product_id = %s",
                    (item.quantity, item.product_id)
                )
        
        # 2. Create order (sharded by customer_id)
        shard = get_shard(customer_id)
        order_id = shard.execute(
            "INSERT INTO orders (customer_id, ...) VALUES (%s, ...)",
            customer_id
        )
        
        # 3. Clear cart (Redis)
        redis.delete(f"cart:{customer_id}")
        
        return order_id
```

Métricas esperadas:
- Inventory check: <50ms (SERIALIZABLE overhead)
- Order creation: <10ms
- Total: <100ms P99
- Throughput: 1K orders/second capacity

Trade-offs discutidos:
- ACID em inventory sacrifica latência (50ms vs 10ms eventual)
- Mas zero overselling é requirement crítico
- Justified pela regra de negócio
```

#### Pergunta 2: Otimização de Database Lento

**Pergunta:**
"Um dashboard de analytics está levando 5 minutos para carregar. A query processa 100M de orders para calcular vendas por categoria nos últimos 30 dias. Como você otimizaria?"

**Resposta Esperada:**

```
Análise do problema:
- Query analítica em database transacional (row-oriented)
- Escaneia muitos dados desnecessariamente
- Provavelmente sem índices adequados
- Workload OLAP em OLTP database

Solução 1: Índices e Query Optimization (Quick win)

-- Adicionar índice composto
CREATE INDEX idx_orders_category_date 
ON orders(category, order_date) 
WHERE order_date >= CURRENT_DATE - INTERVAL '90 days';

-- Partial index: Apenas últimos 90 dias
-- Tamanho: 10% do índice completo

-- Query otimizada:
SELECT category, SUM(amount) as total_sales
FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY category;

-- Performance: 5 minutos → 10-30 segundos
-- Speedup: 10-30x
-- Custo: $0 (apenas criar índice)

Solução 2: Materialized Views (Medium term)

-- Pre-compute daily aggregations
CREATE MATERIALIZED VIEW daily_sales AS
SELECT 
    DATE(order_date) as date,
    category,
    SUM(amount) as total_sales,
    COUNT(*) as order_count
FROM orders
GROUP BY DATE(order_date), category;

-- Refresh diariamente
REFRESH MATERIALIZED VIEW daily_sales;

-- Dashboard query (muito rápido):
SELECT category, SUM(total_sales)
FROM daily_sales
WHERE date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY category;

-- Performance: 5 minutos → 100-500ms
-- Speedup: 600x
-- Trade-off: Dados com até 24h de lag
-- Custo: Storage adicional (~1GB)

Solução 3: Separate Analytics Database (Long term)

-- ETL para Redshift (column-oriented)
-- Nightly batch job:
```python
def etl_to_redshift():
    # Extract from Aurora
    orders = aurora_replica.query("""
        SELECT * FROM orders 
        WHERE order_date >= CURRENT_DATE - 1
    """)
    
    # Transform
    df = pandas.DataFrame(orders)
    # ... transformations ...
    
    # Load to Redshift
    df.to_parquet('s3://data-lake/orders/date=2024-01-15/')
    
    redshift.execute("""
        COPY orders_redshift
        FROM 's3://data-lake/orders/date=2024-01-15/'
        FORMAT AS PARQUET
    """)
```

-- Dashboard query (Redshift):
SELECT category, SUM(amount)
FROM orders_redshift
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY category;

-- Performance: 5 minutos → 2-5 segundos
-- Speedup: 60-150x
-- Benefits:
--   - Offload analytics de production DB
--   - Column-oriented = 10-100x faster para analytics
--   - Escalabilidade independente
-- Custo: ~$200/mês (dc2.large cluster)

Recomendação:
- Short term (hoje): Solução 1 (índices)
- Medium term (2 semanas): Solução 2 (materialized views)
- Long term (2 meses): Solução 3 (Redshift)

Arquitetura final (Lambda architecture):
- Aurora: OLTP (transactions)
- Materialized Views: Serving layer (queries rápidas)
- Redshift: OLAP (analytics complexas, ad-hoc queries)
```

### Recursos Adicionais

**Livros:**
- *Designing Data-Intensive Applications* - Martin Kleppmann (2017)
- *Database Internals* - Alex Petrov (2019)
- *High Performance MySQL* - Baron Schwartz et al. (2012)
- *PostgreSQL: Up and Running* - Regina Obe & Leo Hsu (2017)

**Papers Acadêmicos:**
- "ARIES: A Transaction Recovery Method" - Mohan et al. (1992)
- "The Google File System" - Ghemawat et al. (2003)
- "Dynamo: Amazon's Highly Available Key-value Store" - DeCandia et al. (2007)
- "Amazon Aurora: Design Considerations" - Verbitski et al. (2017)

**Documentação AWS:**
- AWS Well-Architected Framework - Database Lens
- Amazon RDS Best Practices
- Amazon Aurora Best Practices
- Amazon S3 Performance Guidelines

---

**Fonte:** System Design on AWS - Jayanth Kumar, Mandeep Singh

**Autor do Capítulo:** AWS Learning Repository Contributors

**Última Atualização:** 2024

---

*Este capítulo fornece uma base sólida para entender storage types e relational databases em sistemas distribuídos. A combinação de fundamentos teóricos, exemplos práticos na AWS, benchmarks reais e questões de entrevista prepara você para projetar sistemas escaláveis e eficientes.*
