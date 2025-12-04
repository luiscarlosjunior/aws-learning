# AWS Database Migration Service (DMS)

## O que é DMS?

AWS Database Migration Service ajuda a migrar bancos de dados para a AWS de forma rápida e segura. O banco de dados de origem permanece totalmente operacional durante a migração, minimizando o downtime. Suporta migrações homogêneas (mesmo engine) e heterogêneas (engines diferentes).

## Características

- **Minimal downtime**: Replicação contínua (CDC)
- **Homogeneous migrations**: MySQL → RDS MySQL, Oracle → RDS Oracle
- **Heterogeneous migrations**: Oracle → Aurora PostgreSQL, SQL Server → RDS MySQL
- **Consolidation**: Múltiplas sources → single target
- **Continuous replication**: Ongoing data sync
- **Schema Conversion Tool (SCT)**: Converte schemas automaticamente

## Conceitos Principais

### Componentes

```
┌─────────────────┐        ┌─────────────────┐        ┌─────────────────┐
│  Source         │        │  Replication    │        │  Target         │
│  Endpoint       │───────▶│   Instance      │───────▶│  Endpoint       │
│  (On-prem/AWS)  │        │  (Managed EC2)  │        │  (AWS)          │
└─────────────────┘        └─────────────────┘        └─────────────────┘
         │                          │                          │
         │                          │                          │
    Source DB              Runs Migration Tasks           Target DB
```

### Replication Instance

- Managed EC2 instance que executa migration tasks
- Instance classes: dms.t3.micro até dms.r5.24xlarge
- Multi-AZ support para HA
- Storage: gp2, io1

### Endpoints

**Source Endpoint:**
- On-premises database
- EC2 database
- RDS, Aurora
- S3, DynamoDB

**Target Endpoint:**
- RDS, Aurora
- Redshift
- S3
- DynamoDB
- DocumentDB, Neptune
- Kinesis Data Streams
- Kafka

### Migration Types

**Full Load:**
- One-time migration
- Copy all existing data
- Downtime required

**Full Load + CDC:**
- Initial full load
- Then continuous replication
- Minimal downtime

**CDC Only:**
- Ongoing replication
- Assume data já sincronizado
- Zero downtime after initial setup

## Engines Suportados

### Source

- Oracle (10.2+)
- SQL Server (2005+)
- MySQL (5.5+)
- PostgreSQL (9.4+)
- MongoDB (3.x, 4.x)
- MariaDB (10.0.24+)
- SAP ASE (12.5+)
- IBM Db2 (9.7+)
- Amazon S3
- Azure SQL Database

### Target

- Aurora (MySQL/PostgreSQL)
- RDS (MySQL, PostgreSQL, Oracle, SQL Server, MariaDB)
- Redshift
- DynamoDB
- DocumentDB
- Neptune
- S3
- Elasticsearch
- Kinesis Data Streams
- Apache Kafka

## Exemplos Práticos

### 1. MySQL to Aurora MySQL (Homogeneous)

```bash
#!/bin/bash

# 1. Criar replication instance
aws dms create-replication-instance \
    --replication-instance-identifier dms-instance-1 \
    --replication-instance-class dms.r5.large \
    --allocated-storage 100 \
    --vpc-security-group-ids sg-12345678 \
    --replication-subnet-group-identifier dms-subnet-group \
    --multi-az \
    --engine-version 3.4.7 \
    --publicly-accessible false

# 2. Criar source endpoint (MySQL on-prem)
aws dms create-endpoint \
    --endpoint-identifier mysql-source \
    --endpoint-type source \
    --engine-name mysql \
    --username admin \
    --password SourcePassword123! \
    --server-name mysql.example.com \
    --port 3306 \
    --database-name myapp

# 3. Criar target endpoint (Aurora MySQL)
aws dms create-endpoint \
    --endpoint-identifier aurora-target \
    --endpoint-type target \
    --engine-name aurora \
    --username admin \
    --password TargetPassword123! \
    --server-name myaurora.cluster-xxx.us-east-1.rds.amazonaws.com \
    --port 3306 \
    --database-name myapp

# 4. Testar conexões
aws dms test-connection \
    --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:DMS-INSTANCE-1 \
    --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:MYSQL-SOURCE

aws dms test-connection \
    --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:DMS-INSTANCE-1 \
    --endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:AURORA-TARGET

# 5. Criar migration task
aws dms create-replication-task \
    --replication-task-identifier mysql-to-aurora \
    --source-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:MYSQL-SOURCE \
    --target-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:AURORA-TARGET \
    --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:DMS-INSTANCE-1 \
    --migration-type full-load-and-cdc \
    --table-mappings file://table-mappings.json \
    --replication-task-settings file://task-settings.json

# 6. Iniciar migration
aws dms start-replication-task \
    --replication-task-arn arn:aws:dms:us-east-1:123456789012:task:MYSQL-TO-AURORA \
    --start-replication-task-type start-replication
```

**table-mappings.json:**
```json
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all-tables",
      "object-locator": {
        "schema-name": "myapp",
        "table-name": "%"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "transformation",
      "rule-id": "2",
      "rule-name": "rename-schema",
      "rule-target": "schema",
      "object-locator": {
        "schema-name": "myapp"
      },
      "rule-action": "rename",
      "value": "myapp_prod"
    }
  ]
}
```

**task-settings.json:**
```json
{
  "TargetMetadata": {
    "TargetSchema": "",
    "SupportLobs": true,
    "FullLobMode": false,
    "LobChunkSize": 64,
    "LimitedSizeLobMode": true,
    "LobMaxSize": 32
  },
  "FullLoadSettings": {
    "TargetTablePrepMode": "DROP_AND_CREATE",
    "CreatePkAfterFullLoad": false,
    "StopTaskCachedChangesApplied": false,
    "StopTaskCachedChangesNotApplied": false,
    "MaxFullLoadSubTasks": 8,
    "TransactionConsistencyTimeout": 600,
    "CommitRate": 10000
  },
  "Logging": {
    "EnableLogging": true,
    "LogComponents": [
      {
        "Id": "SOURCE_UNLOAD",
        "Severity": "LOGGER_SEVERITY_DEFAULT"
      },
      {
        "Id": "TARGET_LOAD",
        "Severity": "LOGGER_SEVERITY_DEFAULT"
      },
      {
        "Id": "TARGET_APPLY",
        "Severity": "LOGGER_SEVERITY_INFO"
      }
    ]
  },
  "ChangeProcessingTuning": {
    "BatchApplyPreserveTransaction": true,
    "BatchApplyTimeoutMin": 1,
    "BatchApplyTimeoutMax": 30,
    "BatchApplyMemoryLimit": 500,
    "BatchSplitSize": 0,
    "MinTransactionSize": 1000,
    "CommitTimeout": 1,
    "MemoryLimitTotal": 1024,
    "MemoryKeepTime": 60,
    "StatementCacheSize": 50
  }
}
```

### 2. Oracle to Aurora PostgreSQL (Heterogeneous)

```bash
# 1. Usar Schema Conversion Tool (SCT) primeiro
# Download e instale AWS SCT
# Conecte ao Oracle source e Aurora PostgreSQL target
# Converta schema automaticamente
# Gere relatório de conversão

# 2. Aplicar schema convertido manualmente no target

# 3. Criar DMS task para migração de dados
aws dms create-replication-task \
    --replication-task-identifier oracle-to-postgres \
    --source-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:ORACLE-SOURCE \
    --target-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:POSTGRES-TARGET \
    --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:DMS-INSTANCE \
    --migration-type full-load-and-cdc \
    --table-mappings file://oracle-mappings.json
```

### 3. Python - Monitorar Migration

```python
import boto3
import time
from datetime import datetime

dms_client = boto3.client('dms')

def get_task_status(task_arn):
    """Get status de replication task"""
    response = dms_client.describe_replication_tasks(
        Filters=[
            {
                'Name': 'replication-task-arn',
                'Values': [task_arn]
            }
        ]
    )
    
    if response['ReplicationTasks']:
        task = response['ReplicationTasks'][0]
        return {
            'status': task['Status'],
            'progress': task.get('ReplicationTaskStats', {}),
            'message': task.get('StopReason', '')
        }
    return None

def monitor_migration(task_arn, interval=30):
    """Monitor migration progress"""
    print(f"Monitoring task: {task_arn}")
    print("=" * 80)
    
    while True:
        status = get_task_status(task_arn)
        
        if not status:
            print("Task not found")
            break
        
        timestamp = datetime.now().strftime('%Y-%m-%d %H:%M:%S')
        print(f"\n[{timestamp}] Status: {status['status']}")
        
        if 'progress' in status:
            stats = status['progress']
            print(f"  Full Load Progress: {stats.get('FullLoadProgressPercent', 0)}%")
            print(f"  Tables Loaded: {stats.get('TablesLoaded', 0)}")
            print(f"  Tables Loading: {stats.get('TablesLoading', 0)}")
            print(f"  Tables Queued: {stats.get('TablesQueued', 0)}")
            print(f"  Tables Errored: {stats.get('TablesErrored', 0)}")
            
            if stats.get('FullLoadProgressPercent') == 100:
                print(f"  CDC Latency: {stats.get('CDCLatencyTarget', 'N/A')} seconds")
        
        if status['status'] in ['stopped', 'failed']:
            print(f"\nTask {status['status']}: {status['message']}")
            break
        
        if status['status'] == 'running' and stats.get('FullLoadProgressPercent') == 100:
            print("\n✓ Full load complete. CDC replication ongoing...")
            print("Task will continue running. Press Ctrl+C to stop monitoring.")
        
        time.sleep(interval)

# Uso
task_arn = 'arn:aws:dms:us-east-1:123456789012:task:MYSQL-TO-AURORA'
monitor_migration(task_arn, interval=30)
```

### 4. MongoDB to DocumentDB

```bash
# Criar source endpoint (MongoDB)
aws dms create-endpoint \
    --endpoint-identifier mongodb-source \
    --endpoint-type source \
    --engine-name mongodb \
    --username admin \
    --password MongoPassword123! \
    --server-name mongodb.example.com \
    --port 27017 \
    --database-name mydb \
    --mongodb-settings '{
        "AuthType": "password",
        "AuthSource": "admin",
        "AuthMechanism": "SCRAM-SHA-1",
        "NestingLevel": "one",
        "ExtractDocId": "true",
        "DocsToInvestigate": "1000"
    }'

# Criar target endpoint (DocumentDB)
aws dms create-endpoint \
    --endpoint-identifier documentdb-target \
    --endpoint-type target \
    --engine-name docdb \
    --username admin \
    --password DocDBPassword123! \
    --server-name my-docdb.cluster-xxx.us-east-1.docdb.amazonaws.com \
    --port 27017 \
    --database-name mydb \
    --ssl-mode require

# Criar migration task
aws dms create-replication-task \
    --replication-task-identifier mongodb-to-documentdb \
    --source-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:MONGODB-SOURCE \
    --target-endpoint-arn arn:aws:dms:us-east-1:123456789012:endpoint:DOCUMENTDB-TARGET \
    --replication-instance-arn arn:aws:dms:us-east-1:123456789012:rep:DMS-INSTANCE \
    --migration-type full-load-and-cdc
```

### 5. Database to S3 (Data Lake)

```bash
# Criar target endpoint (S3)
aws dms create-endpoint \
    --endpoint-identifier s3-target \
    --endpoint-type target \
    --engine-name s3 \
    --s3-settings '{
        "BucketName": "my-data-lake",
        "BucketFolder": "database-exports",
        "CompressionType": "gzip",
        "DataFormat": "parquet",
        "DatePartitionEnabled": true,
        "DatePartitionSequence": "YYYYMMDD",
        "ParquetVersion": "parquet-2-0",
        "EnableStatistics": true,
        "CdcPath": "cdc-data",
        "ServiceAccessRoleArn": "arn:aws:iam::123456789012:role/dms-s3-role"
    }'

# Table mappings para partitioning
cat > s3-mappings.json <<EOF
{
  "rules": [
    {
      "rule-type": "selection",
      "rule-id": "1",
      "rule-name": "include-all",
      "object-locator": {
        "schema-name": "%",
        "table-name": "%"
      },
      "rule-action": "include"
    },
    {
      "rule-type": "object-mapping",
      "rule-id": "2",
      "rule-name": "partition-by-date",
      "rule-action": "map-record-to-record",
      "object-locator": {
        "schema-name": "myapp",
        "table-name": "orders"
      },
      "target-table-name": "orders",
      "mapping-parameters": {
        "partition-enabled": true,
        "partition-type": "date-based",
        "partition-date-column": "created_at"
      }
    }
  ]
}
EOF
```

### 6. Validation e Testing

```python
import boto3
from collections import defaultdict

dms_client = boto3.client('dms')

def validate_migration(task_arn):
    """Validar migration results"""
    # Get table statistics
    response = dms_client.describe_table_statistics(
        ReplicationTaskArn=task_arn
    )
    
    stats = defaultdict(dict)
    issues = []
    
    for table in response['TableStatistics']:
        table_name = f"{table['SchemaName']}.{table['TableName']}"
        
        stats[table_name] = {
            'full_load_rows': table.get('FullLoadRows', 0),
            'full_load_condtnl_chk_failed_rows': table.get('FullLoadCondtnlChkFailedRows', 0),
            'full_load_error_rows': table.get('FullLoadErrorRows', 0),
            'inserts': table.get('Inserts', 0),
            'updates': table.get('Updates', 0),
            'deletes': table.get('Deletes', 0),
            'ddls': table.get('Ddls', 0),
            'validation_state': table.get('ValidationState', 'N/A'),
            'validation_pending': table.get('ValidationPendingRecords', 0),
            'validation_failed': table.get('ValidationFailedRecords', 0)
        }
        
        # Check for issues
        if table.get('FullLoadErrorRows', 0) > 0:
            issues.append(f"❌ {table_name}: {table['FullLoadErrorRows']} error rows")
        
        if table.get('ValidationFailedRecords', 0) > 0:
            issues.append(f"⚠️  {table_name}: {table['ValidationFailedRecords']} validation failures")
    
    # Print summary
    print("\n" + "=" * 80)
    print("MIGRATION VALIDATION REPORT")
    print("=" * 80)
    
    for table_name, table_stats in stats.items():
        print(f"\n📊 {table_name}")
        print(f"  Full Load: {table_stats['full_load_rows']:,} rows")
        print(f"  CDC - Inserts: {table_stats['inserts']:,}, Updates: {table_stats['updates']:,}, Deletes: {table_stats['deletes']:,}")
        print(f"  Validation: {table_stats['validation_state']}")
        if table_stats['validation_failed'] > 0:
            print(f"  ⚠️  Failed validations: {table_stats['validation_failed']}")
    
    if issues:
        print("\n" + "=" * 80)
        print("ISSUES FOUND:")
        print("=" * 80)
        for issue in issues:
            print(issue)
    else:
        print("\n✅ All tables migrated successfully!")
    
    return stats, issues

# Uso
task_arn = 'arn:aws:dms:us-east-1:123456789012:task:MYSQL-TO-AURORA'
stats, issues = validate_migration(task_arn)
```

## Schema Conversion Tool (SCT)

### Heterogeneous Migration Workflow

```
1. Download AWS SCT
   ↓
2. Connect to source DB (Oracle, SQL Server, etc.)
   ↓
3. Connect to target DB (Aurora PostgreSQL, MySQL, etc.)
   ↓
4. Create database migration assessment report
   ↓
5. Convert schema automatically
   ↓
6. Review conversion issues
   ↓
7. Apply converted schema to target
   ↓
8. Use DMS for data migration
```

### Conversion Examples

**Oracle PL/SQL → PostgreSQL PL/pgSQL:**
- Procedures → Functions
- Packages → Schemas
- Sequences → Serial columns
- ROWNUM → ROW_NUMBER()

**SQL Server T-SQL → MySQL:**
- IDENTITY → AUTO_INCREMENT
- TOP → LIMIT
- GETDATE() → NOW()
- Stored procedures conversion

## Melhores Práticas

### Planejamento
1. **Assessment first** - Use SCT para heterogeneous
2. **Test migration** em non-prod primeiro
3. **Document dependencies** (apps, ETL, reporting)
4. **Plan cutover** carefully
5. **Rollback strategy** pronta

### Performance
1. **Right-size replication instance**
2. **Parallel full load** (MaxFullLoadSubTasks)
3. **LOB handling** appropriately
4. **Batch apply** settings
5. **Network bandwidth** adequado
6. **Monitor metrics** (latency, throughput)

### Reliability
1. **Enable CloudWatch logs**
2. **Multi-AZ replication instance** para prod
3. **Regular monitoring** de task status
4. **Validation** enabled
5. **Backup source** antes de cutover

### Segurança
1. **VPC deployment** para replication instance
2. **Encrypt connections** (SSL/TLS)
3. **IAM roles** para AWS resources
4. **Secrets Manager** para credentials
5. **Least privilege** access

### Custo
1. **Stop tasks** quando não necessários
2. **Delete replication instances** após migration
3. **Appropriate instance sizing**
4. **Monitor data transfer** costs
5. **Multi-AZ** apenas se necessário

## Limitações

- Max task size: Depends on instance
- LOB support: Limited (32 MB default)
- Some DDL not supported in CDC
- Transformation rules: Limited
- No triggers/stored procedures migration (use SCT)

## Troubleshooting

### Common Issues

**Connection failures:**
```bash
# Check security groups
# Verify network connectivity
# Test endpoint connections
aws dms test-connection \
    --replication-instance-arn <arn> \
    --endpoint-arn <endpoint-arn>
```

**Slow performance:**
- Increase replication instance size
- Adjust MaxFullLoadSubTasks
- Check network latency
- Review source database load

**CDC lag:**
- Check source transaction log retention
- Increase instance size
- Review BatchApply settings
- Check target write capacity

## Perguntas e Respostas

### P: DMS vs native replication?
**R:** DMS: managed, heterogeneous support, minimal setup. Native: engine-specific, mais control, pode ser mais rápido. Use DMS para heterogeneous ou simplicity, native para homogeneous performance-critical.

### P: Downtime durante migration?
**R:** Full-load-and-cdc: minimal (seconds para cutover). Full-load only: downtime durante load. CDC-only: zero se já sincronizado. Plan cutover window adequadamente.

### P: Como lidar com large databases?
**R:** Use Snowball for >10TB initial load, então DMS para CDC. Ou full-load-and-cdc com large replication instance. Parallel load (MaxFullLoadSubTasks). Consider partitioning strategy.

### P: DMS suporta stored procedures?
**R:** Não, DMS migra dados apenas. Use SCT para converter code. Ou rewrite manualmente. Triggers não executados durante migration.

### P: Como validar data accuracy?
**R:** Enable DMS validation. Compare row counts. Sample data validation. Run application tests. Consider third-party tools para complex validation.

### P: Posso pausar migration?
**R:** Sim, stop task e resume later. CDC picks up from last checkpoint. Be careful com source log retention. Monitor storage durante pause.

### P: Como migrar incrementally?
**R:** Use table selection rules para migrar subset. Full-load-and-cdc allows ongoing sync. Consider phased migration (tables by priority).

### P: Performance tuning?
**R:** Instance size critical. LOB handling strategy. Parallel load settings. Batch apply tuning. Monitor CloudWatch metrics. Network optimization.

### P: Cost of DMS?
**R:** Replication instance hours (on-demand ou RI). Data transfer costs. Storage. No charge para target writes. Stop instances quando não em uso.

### P: Multi-region migration?
**R:** Use DMS em target region. Cross-region endpoints supported. Consider data transfer costs e latency. VPN/Direct Connect recommended.

## Recursos de Aprendizado

- [DMS Documentation](https://docs.aws.amazon.com/dms/)
- [DMS Best Practices](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_BestPractices.html)
- [Schema Conversion Tool](https://docs.aws.amazon.com/SchemaConversionTool/latest/userguide/)
- [DMS Migration Playbooks](https://docs.aws.amazon.com/dms/latest/userguide/CHAP_DMSStudio_Playbooks.html)
- [AWS re:Invent DMS Sessions](https://www.youtube.com/results?search_query=reinvent+dms)
- [DMS Pricing](https://aws.amazon.com/dms/pricing/)
