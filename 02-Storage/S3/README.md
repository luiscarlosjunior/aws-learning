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

## Recursos de Aprendizado

- [S3 User Guide](https://docs.aws.amazon.com/s3/)
- [S3 Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/best-practices.html)
- [S3 Performance Guidelines](https://docs.aws.amazon.com/AmazonS3/latest/userguide/optimizing-performance.html)
- [S3 Security Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html)
