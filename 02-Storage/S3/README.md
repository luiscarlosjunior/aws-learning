# Amazon S3 (Simple Storage Service)

## O que é S3?

Amazon S3 é um serviço de armazenamento de objetos que oferece escalabilidade, disponibilidade de dados, segurança e performance líderes do setor.

## Conceitos Principais

### Buckets
Containers para armazenar objetos no S3. Cada bucket tem um nome globalmente único.

### Objects
Arquivos armazenados no S3. Cada objeto pode ter até 5 TB.

**Componentes de um Objeto:**
- **Key**: Nome do arquivo (path completo)
- **Value**: Conteúdo do arquivo
- **Version ID**: Identificador de versão
- **Metadata**: Pares key-value sobre o objeto
- **Access Control**: Permissões de acesso

## Classes de Armazenamento

### S3 Standard
- **Uso**: Dados acessados frequentemente
- **Durabilidade**: 11 noves (99.999999999%)
- **Disponibilidade**: 99.99%
- **Latência**: Milissegundos
- **Custo**: $$$$

### S3 Intelligent-Tiering
- **Uso**: Dados com padrões de acesso desconhecidos ou variáveis
- **Automação**: Move automaticamente objetos entre tiers
- **Custo**: Otimizado automaticamente
- **Tiers**:
  - Frequent Access
  - Infrequent Access (30 dias)
  - Archive Instant Access (90 dias)
  - Archive Access (90-270 dias)
  - Deep Archive Access (180-730 dias)

### S3 Standard-IA (Infrequent Access)
- **Uso**: Dados acessados raramente mas precisam de acesso rápido
- **Durabilidade**: 11 noves
- **Disponibilidade**: 99.9%
- **Custo**: $$$
- **Retrieval Fee**: Por GB recuperado

### S3 One Zone-IA
- **Uso**: Dados secundários/reproduzíveis
- **Armazenamento**: Uma única AZ
- **Disponibilidade**: 99.5%
- **Custo**: $$

### S3 Glacier Instant Retrieval
- **Uso**: Archive data com acesso instantâneo raro
- **Retrieval**: Milissegundos
- **Mínimo**: 90 dias de armazenamento
- **Custo**: $$

### S3 Glacier Flexible Retrieval
- **Uso**: Archive data de longo prazo
- **Retrieval**:
  - Expedited: 1-5 minutos
  - Standard: 3-5 horas
  - Bulk: 5-12 horas
- **Mínimo**: 90 dias
- **Custo**: $

### S3 Glacier Deep Archive
- **Uso**: Archive data raramente acessado
- **Retrieval**:
  - Standard: 12 horas
  - Bulk: 48 horas
- **Mínimo**: 180 dias
- **Custo**: ¢ (mais barato)

## Recursos

### Versioning
Mantém múltiplas versões de objetos no mesmo bucket.

**Benefícios:**
- Proteção contra exclusão acidental
- Recuperação de versões antigas
- Compliance

### Lifecycle Policies
Automatiza transição de objetos entre classes de armazenamento.

```json
{
  "Rules": [{
    "Id": "Archive old logs",
    "Status": "Enabled",
    "Transitions": [
      {
        "Days": 30,
        "StorageClass": "STANDARD_IA"
      },
      {
        "Days": 90,
        "StorageClass": "GLACIER"
      }
    ],
    "Expiration": {
      "Days": 365
    }
  }]
}
```

### Replication
Copia objetos automaticamente entre buckets.

**Tipos:**
- **CRR (Cross-Region Replication)**: Entre regiões
- **SRR (Same-Region Replication)**: Na mesma região

**Casos de Uso:**
- Disaster recovery
- Compliance (dados em múltiplas regiões)
- Latência reduzida
- Agregação de logs

### Encryption

**At Rest:**
- **SSE-S3**: AWS-managed keys
- **SSE-KMS**: KMS-managed keys
- **SSE-C**: Customer-provided keys
- **Client-side**: Cliente faz encryption

**In Transit:**
- HTTPS/TLS

### S3 Object Lock
WORM (Write Once Read Many) model para compliance.

**Modos:**
- **Governance**: Usuários especiais podem modificar
- **Compliance**: Ninguém pode modificar (nem root)

### S3 Access Points
Simplifica gerenciamento de acesso a shared datasets.

### S3 Event Notifications
Trigger eventos quando objetos são criados/modificados/deletados.

**Destinos:**
- SNS
- SQS
- Lambda
- EventBridge

## Segurança

### Bucket Policies
Políticas JSON para controlar acesso ao bucket.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicRead",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

### Access Control Lists (ACLs)
Controle de acesso granular (legacy, use bucket policies).

### Block Public Access
Proteção adicional contra exposição acidental.

### S3 Access Analyzer
Identifica buckets acessíveis externamente.

## Performance

### Transfer Acceleration
Usa CloudFront edge locations para uploads mais rápidos.

### Multipart Upload
- Obrigatório para objetos > 5 GB
- Recomendado para objetos > 100 MB
- Upload paralelo de partes
- Retomada de upload

### S3 Select
Query dados diretamente no S3 usando SQL.

```python
response = s3.select_object_content(
    Bucket='my-bucket',
    Key='data.csv',
    Expression="SELECT * FROM S3Object WHERE age > 30",
    ExpressionType='SQL',
    InputSerialization={'CSV': {"FileHeaderInfo": "Use"}},
    OutputSerialization={'CSV': {}}
)
```

### Byte-Range Fetches
Download parcial de objetos.

## Casos de Uso

1. **Data Lakes**: Armazenamento centralizado para analytics
2. **Backup e Archive**: DR e compliance
3. **Static Website Hosting**: Sites estáticos
4. **Media Storage**: Imagens, vídeos
5. **Application Data**: User uploads, logs
6. **Big Data**: Input para EMR, Athena, Redshift

## Melhores Práticas

### Performance
- Use multipart upload para arquivos grandes
- Enable Transfer Acceleration para uploads globais
- Use CloudFront para distribuição
- Implement request rate optimization

### Custo
- Use lifecycle policies
- Enable Intelligent-Tiering
- Delete incomplete multipart uploads
- Use S3 Analytics para insights
- Compress data

### Segurança
- Enable encryption
- Block public access por padrão
- Use least privilege IAM policies
- Enable versioning para dados críticos
- Enable logging (S3 Server Access Logs)
- Use VPC Endpoints para acesso privado

### Confiabilidade
- Enable versioning
- Implement lifecycle policies
- Use replication para DR
- Regular backup testing
- Monitor com CloudWatch

## Pricing

**Componentes:**
1. **Storage**: Por GB/mês (varia por classe)
2. **Requests**: PUT, GET, LIST, etc.
3. **Data Transfer**: OUT para internet
4. **Management**: Analytics, Inventory, etc.

**Exemplo (us-east-1):**
- Standard: ~$0.023/GB/mês
- Standard-IA: ~$0.0125/GB/mês
- Glacier: ~$0.004/GB/mês
- Deep Archive: ~$0.00099/GB/mês

## Static Website Hosting

```bash
# Configure bucket for hosting
aws s3 website s3://my-bucket \
    --index-document index.html \
    --error-document error.html

# Upload website
aws s3 sync ./dist s3://my-bucket

# URL: http://my-bucket.s3-website-us-east-1.amazonaws.com
```

## Comandos Úteis

```bash
# Create bucket
aws s3 mb s3://my-bucket

# List buckets
aws s3 ls

# List objects
aws s3 ls s3://my-bucket

# Upload file
aws s3 cp file.txt s3://my-bucket/

# Upload directory
aws s3 sync ./local-dir s3://my-bucket/

# Download file
aws s3 cp s3://my-bucket/file.txt ./

# Delete object
aws s3 rm s3://my-bucket/file.txt

# Delete bucket (must be empty)
aws s3 rb s3://my-bucket

# Presigned URL (temporary access)
aws s3 presign s3://my-bucket/file.txt --expires-in 3600
```

## Exemplo: Python SDK (boto3)

```python
import boto3

s3 = boto3.client('s3')

# Upload file
s3.upload_file('local.txt', 'my-bucket', 'remote.txt')

# Download file
s3.download_file('my-bucket', 'remote.txt', 'local.txt')

# List objects
response = s3.list_objects_v2(Bucket='my-bucket')
for obj in response.get('Contents', []):
    print(obj['Key'])

# Generate presigned URL
url = s3.generate_presigned_url(
    'get_object',
    Params={'Bucket': 'my-bucket', 'Key': 'file.txt'},
    ExpiresIn=3600
)
```

## Integração com Outros Serviços

- **CloudFront**: CDN para distribuição
- **Lambda**: Processamento de eventos
- **Athena**: Query data com SQL
- **Glue**: ETL e data catalog
- **EMR**: Big data processing
- **Redshift**: Data warehouse load
- **EC2**: Instance storage
- **EBS**: Snapshots storage
- **RDS**: Backups storage

## Monitoramento

### CloudWatch Metrics
- BucketSizeBytes
- NumberOfObjects
- AllRequests
- 4xxErrors, 5xxErrors

### S3 Access Logs
Logs detalhados de todas as requisições.

### CloudTrail
Logs de API calls (management events).

## Troubleshooting Comum

### Erro 403 Forbidden
**Problema**: Acesso negado ao objeto
**Soluções**:
- Verificar bucket policy
- Verificar IAM permissions
- Verificar Block Public Access settings
- Verificar ACLs do objeto
- Verificar S3 Object Lock

### Erro 404 Not Found
**Problema**: Objeto não encontrado
**Soluções**:
- Verificar se key está correto (case-sensitive)
- Verificar se bucket existe
- Verificar se versioning está habilitado
- Verificar eventual consistency (improvável agora)

### Slow Upload/Download
**Problema**: Performance lenta
**Soluções**:
- Use Transfer Acceleration
- Implementar multipart upload
- Usar CloudFront para distribuição
- Verificar network bandwidth
- Usar endpoint correto (VPC endpoint)

### High Costs
**Problema**: Custos elevados
**Soluções**:
- Implementar lifecycle policies
- Usar Intelligent-Tiering
- Limpar incomplete multipart uploads
- Comprimir dados
- Revisar storage class de cada objeto
- Configurar S3 Analytics

## Perguntas e Respostas (Q&A)

### Conceitos Básicos

**P: Qual a diferença entre S3 e EBS?**
R: S3 é armazenamento de objetos (files), acessível via HTTP/HTTPS de qualquer lugar. EBS é armazenamento em bloco (volumes) que precisa ser anexado a uma instância EC2. S3 é melhor para arquivos estáticos e backups, EBS para databases e sistemas operacionais.

**P: Como funciona a durabilidade de 11 noves?**
R: Significa 99.999999999% de durabilidade, ou seja, se você armazenar 10 milhões de objetos, pode esperar perder apenas 1 objeto a cada 10.000 anos. A AWS replica automaticamente seus dados em pelo menos 3 availability zones.

**P: S3 é um sistema de arquivos?**
R: Não, S3 é armazenamento de objetos, não um file system hierárquico. As "pastas" no console são apenas prefixos nos nomes dos objetos. Não há verdadeira estrutura de diretórios.

**P: Posso hospedar um site dinâmico no S3?**
R: Não diretamente. S3 Static Website Hosting serve apenas conteúdo estático (HTML, CSS, JS). Para sites dinâmicos, você precisa usar EC2, Lambda, ou Amplify Hosting.

### Performance

**P: Qual o limite de requests por segundo no S3?**
R: 3.500 PUT/COPY/POST/DELETE e 5.500 GET/HEAD requests por segundo por prefixo. Para mais performance, distribua objetos em múltiplos prefixos.

**P: Quando devo usar multipart upload?**
R: É obrigatório para objetos maiores que 5 GB. Recomendado para objetos maiores que 100 MB. Permite upload paralelo e recuperação de falhas.

**P: Transfer Acceleration vale a pena?**
R: Sim, se você faz uploads de locais distantes da região do seu bucket. Pode acelerar uploads em até 50-500%, especialmente para arquivos grandes. Teste usando a ferramenta de comparação da AWS.

**P: Como melhorar performance de GET requests?**
R: Use CloudFront CDN para cache, habilite HTTP/2, use byte-range fetches para downloads parciais, e considere usar prefixos aleatórios para distribuir carga.

### Segurança

**P: Como tornar um bucket completamente privado?**
R: Habilite "Block Public Access" em todas as configurações, não adicione bucket policies públicas, não use ACLs públicas, e use IAM roles para acesso.

**P: Qual encryption devo usar: SSE-S3, SSE-KMS ou SSE-C?**
R: 
- SSE-S3: Simples, gerenciado pela AWS, sem custo extra, ideal para maioria dos casos
- SSE-KMS: Quando precisa de controle de chaves, audit trail, ou compliance específico. Custo adicional por requests
- SSE-C: Quando deve gerenciar suas próprias chaves (você é responsável)

**P: Como compartilhar acesso temporário a um objeto privado?**
R: Use presigned URLs. Você pode gerar URLs que concedem acesso temporário (ex: 1 hora) sem tornar o objeto público.

**P: Bucket policies vs IAM policies - qual usar?**
R: 
- Bucket policies: Para acesso cross-account, IP restrictions, ou regras aplicadas a todos os objetos no bucket
- IAM policies: Para controlar o que usuários/roles específicos podem fazer
- Melhor: Usar ambos em conjunto (defense in depth)

### Custos

**P: Como reduzir custos de armazenamento S3?**
R: 
1. Use lifecycle policies para mover dados para classes mais baratas
2. Habilite Intelligent-Tiering para automação
3. Delete objetos não utilizados
4. Limpe incomplete multipart uploads
5. Comprima arquivos antes de upload
6. Use S3 Storage Class Analysis para insights

**P: Sou cobrado por requests mesmo com Free Tier?**
R: Free Tier inclui 20.000 GET requests e 2.000 PUT requests por mês. Depois disso, há cobrança (~$0.0004 por 1.000 GET requests).

**P: Data transfer dentro da mesma região é cobrado?**
R: Não, transferência entre S3 e outros serviços AWS na mesma região é geralmente gratuita. Cobranças aplicam-se para data transfer OUT para internet.

**P: Qual a diferença de custo entre storage classes?**
R: (us-east-1 aproximado)
- Standard: $0.023/GB
- Standard-IA: $0.0125/GB (+ retrieval fee)
- Glacier Instant: $0.004/GB
- Glacier Flexible: $0.0036/GB
- Deep Archive: $0.00099/GB (mais barato)

### Versionamento e Lifecycle

**P: Se eu habilitar versioning, os custos dobram?**
R: Potencialmente sim, porque cada versão é cobrada. Use lifecycle policies para deletar versões antigas ou movê-las para classes mais baratas.

**P: Como deletar um objeto versionado permanentemente?**
R: Com versioning habilitado, DELETE cria um delete marker. Para deletar permanentemente, você deve deletar o version ID específico.

**P: Lifecycle policies afetam todas as versões?**
R: Você pode configurar regras específicas para current version vs non-current versions. Por exemplo, manter current version no Standard e mover non-current para Glacier.

### Replicação

**P: Replicação é em tempo real?**
R: É near real-time. Objetos pequenos replicam em minutos, objetos grandes podem levar mais tempo. Não é síncrona.

**P: Replicação funciona para objetos existentes?**
R: Por padrão não, apenas para novos objetos. Use S3 Batch Replication para replicar objetos existentes.

**P: Posso replicar entre contas AWS?**
R: Sim, CRR (Cross-Region Replication) e SRR (Same-Region Replication) suportam cross-account replication. Configure IAM roles apropriadas.

### Casos Práticos

**P: Como implementar um data lake no S3?**
R: 
1. Organize dados por domain/team usando prefixos
2. Use partitioning (ex: year/month/day)
3. Armazene em formatos otimizados (Parquet, ORC)
4. Use AWS Glue para catalogar
5. Query com Athena ou Redshift Spectrum
6. Implemente lifecycle policies

**P: Como fazer backup de terabytes de dados para S3?**
R: 
1. Use AWS DataSync para transferências grandes
2. Ou AWS Snowball para volumes muito grandes (>10TB)
3. Configure multipart upload
4. Use Transfer Acceleration
5. Implemente lifecycle para Glacier Deep Archive
6. Teste procedimentos de restore

**P: Como servir imagens otimizadas de um site?**
R: 
1. Armazene originais no S3
2. Use CloudFront na frente
3. Implemente Lambda@Edge para redimensionamento dinâmico
4. Ou use AWS Amplify Image Optimization
5. Configure cache headers apropriados
6. Use WebP/AVIF quando suportado pelo browser

## Exemplos Avançados

### Exemplo 1: Lambda para Processamento de Imagem ao Upload

```python
import boto3
from PIL import Image
import io

s3 = boto3.client('s3')

def lambda_handler(event, context):
    """Cria thumbnail quando imagem é uploaded"""
    
    # Pegar informações do evento S3
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Baixar imagem original
    response = s3.get_object(Bucket=bucket, Key=key)
    image_content = response['Body'].read()
    
    # Criar thumbnail
    image = Image.open(io.BytesIO(image_content))
    image.thumbnail((200, 200))
    
    # Salvar thumbnail
    buffer = io.BytesIO()
    image.save(buffer, format='JPEG')
    buffer.seek(0)
    
    # Upload thumbnail
    thumbnail_key = f"thumbnails/{key}"
    s3.put_object(
        Bucket=bucket,
        Key=thumbnail_key,
        Body=buffer,
        ContentType='image/jpeg'
    )
    
    return {
        'statusCode': 200,
        'body': f'Thumbnail criado: {thumbnail_key}'
    }
```

### Exemplo 2: CloudFormation para Bucket Seguro

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: S3 Bucket seguro com encryption, versioning e lifecycle

Resources:
  SecureBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'secure-bucket-${AWS::AccountId}'
      BucketEncryption:
        ServerSideEncryptionConfiguration:
          - ServerSideEncryptionByDefault:
              SSEAlgorithm: 'AES256'
      VersioningConfiguration:
        Status: Enabled
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true
      LifecycleConfiguration:
        Rules:
          - Id: TransitionToIA
            Status: Enabled
            Transitions:
              - TransitionInDays: 30
                StorageClass: STANDARD_IA
              - TransitionInDays: 90
                StorageClass: GLACIER
            NoncurrentVersionTransitions:
              - TransitionInDays: 30
                StorageClass: GLACIER
            NoncurrentVersionExpiration:
              NoncurrentDays: 90
      LoggingConfiguration:
        DestinationBucketName: !Ref LoggingBucket
        LogFilePrefix: access-logs/
      Tags:
        - Key: Environment
          Value: Production
        
  LoggingBucket:
    Type: AWS::S3::Bucket
    Properties:
      BucketName: !Sub 'secure-bucket-logs-${AWS::AccountId}'
      AccessControl: LogDeliveryWrite
      PublicAccessBlockConfiguration:
        BlockPublicAcls: true
        BlockPublicPolicy: true
        IgnorePublicAcls: true
        RestrictPublicBuckets: true

  BucketPolicy:
    Type: AWS::S3::BucketPolicy
    Properties:
      Bucket: !Ref SecureBucket
      PolicyDocument:
        Statement:
          - Sid: DenyUnencryptedObjectUploads
            Effect: Deny
            Principal: '*'
            Action: s3:PutObject
            Resource: !Sub '${SecureBucket.Arn}/*'
            Condition:
              StringNotEquals:
                s3:x-amz-server-side-encryption: 'AES256'
          - Sid: DenyInsecureTransport
            Effect: Deny
            Principal: '*'
            Action: 's3:*'
            Resource:
              - !GetAtt SecureBucket.Arn
              - !Sub '${SecureBucket.Arn}/*'
            Condition:
              Bool:
                aws:SecureTransport: false

Outputs:
  BucketName:
    Value: !Ref SecureBucket
    Description: Nome do bucket seguro
  BucketArn:
    Value: !GetAtt SecureBucket.Arn
    Description: ARN do bucket
```

### Exemplo 3: Script Python para Backup Incremental

```python
import boto3
import hashlib
import json
from datetime import datetime
from pathlib import Path

class S3BackupManager:
    def __init__(self, bucket_name):
        self.s3 = boto3.client('s3')
        self.bucket = bucket_name
        self.state_file = '.backup_state.json'
        
    def calculate_md5(self, file_path):
        """Calcula MD5 hash do arquivo"""
        hash_md5 = hashlib.md5()
        with open(file_path, 'rb') as f:
            for chunk in iter(lambda: f.read(4096), b''):
                hash_md5.update(chunk)
        return hash_md5.hexdigest()
    
    def load_state(self):
        """Carrega estado do último backup"""
        try:
            with open(self.state_file, 'r') as f:
                return json.load(f)
        except FileNotFoundError:
            return {}
    
    def save_state(self, state):
        """Salva estado atual"""
        with open(self.state_file, 'w') as f:
            json.dump(state, f, indent=2)
    
    def incremental_backup(self, local_dir, s3_prefix='backup'):
        """Faz backup incremental de diretório"""
        local_dir = Path(local_dir)
        state = self.load_state()
        new_state = {}
        uploaded = 0
        skipped = 0
        
        for file_path in local_dir.rglob('*'):
            if file_path.is_file():
                # Calcular hash
                current_hash = self.calculate_md5(file_path)
                relative_path = file_path.relative_to(local_dir)
                s3_key = f"{s3_prefix}/{relative_path}"
                
                # Verificar se arquivo mudou
                if str(relative_path) in state and state[str(relative_path)] == current_hash:
                    skipped += 1
                    new_state[str(relative_path)] = current_hash
                    continue
                
                # Upload arquivo
                print(f"Uploading {relative_path}...")
                self.s3.upload_file(
                    str(file_path),
                    self.bucket,
                    s3_key,
                    ExtraArgs={
                        'Metadata': {
                            'md5': current_hash,
                            'backup-date': datetime.now().isoformat()
                        }
                    }
                )
                
                new_state[str(relative_path)] = current_hash
                uploaded += 1
        
        self.save_state(new_state)
        print(f"\nBackup completo!")
        print(f"Arquivos uploaded: {uploaded}")
        print(f"Arquivos skipped: {skipped}")
        
    def restore(self, s3_prefix, local_dir):
        """Restaura backup do S3"""
        local_dir = Path(local_dir)
        local_dir.mkdir(parents=True, exist_ok=True)
        
        paginator = self.s3.get_paginator('list_objects_v2')
        pages = paginator.paginate(Bucket=self.bucket, Prefix=s3_prefix)
        
        for page in pages:
            for obj in page.get('Contents', []):
                key = obj['Key']
                relative_path = key.replace(f"{s3_prefix}/", '')
                local_path = local_dir / relative_path
                
                local_path.parent.mkdir(parents=True, exist_ok=True)
                
                print(f"Downloading {relative_path}...")
                self.s3.download_file(self.bucket, key, str(local_path))
        
        print("Restore completo!")

# Uso
if __name__ == '__main__':
    backup = S3BackupManager('my-backup-bucket')
    
    # Fazer backup incremental
    backup.incremental_backup('./documents', s3_prefix='docs-backup')
    
    # Restaurar backup
    # backup.restore('docs-backup', './restored-documents')
```

### Exemplo 4: Terraform para Data Lake

```hcl
# Bucket do Data Lake
resource "aws_s3_bucket" "data_lake" {
  bucket = "company-data-lake-${var.environment}"
  
  tags = {
    Name        = "Data Lake"
    Environment = var.environment
  }
}

# Versioning
resource "aws_s3_bucket_versioning" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  
  versioning_configuration {
    status = "Enabled"
  }
}

# Encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm = "AES256"
    }
  }
}

# Lifecycle policies
resource "aws_s3_bucket_lifecycle_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id

  rule {
    id     = "raw-data-transition"
    status = "Enabled"

    filter {
      prefix = "raw/"
    }

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }
  }

  rule {
    id     = "processed-data-transition"
    status = "Enabled"

    filter {
      prefix = "processed/"
    }

    transition {
      days          = 180
      storage_class = "GLACIER"
    }
  }

  rule {
    id     = "temp-data-expiration"
    status = "Enabled"

    filter {
      prefix = "temp/"
    }

    expiration {
      days = 7
    }
  }
}

# Replication para DR
resource "aws_s3_bucket_replication_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-all"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.data_lake_replica.arn
      storage_class = "STANDARD_IA"
    }
  }

  depends_on = [aws_s3_bucket_versioning.data_lake]
}

# Public access block
resource "aws_s3_bucket_public_access_block" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# Bucket para Analytics
resource "aws_s3_bucket_analytics_configuration" "data_lake" {
  bucket = aws_s3_bucket.data_lake.id
  name   = "EntireBucketAnalytics"

  storage_class_analysis {
    data_export {
      destination {
        s3_bucket_destination {
          bucket_arn = aws_s3_bucket.analytics.arn
          prefix     = "analytics"
        }
      }
    }
  }
}
```

## Recursos de Aprendizado

- [S3 User Guide](https://docs.aws.amazon.com/s3/)
- [S3 Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/best-practices.html)
- [S3 Performance Guidelines](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
- [AWS re:Invent S3 Sessions](https://www.youtube.com/results?search_query=reinvent+s3)
- [S3 FAQ](https://aws.amazon.com/s3/faqs/)
- [AWS Workshops - S3](https://workshops.aws/categories/S3)
