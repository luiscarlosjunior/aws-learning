# Database Services

## Visão Geral
Os serviços de banco de dados da AWS oferecem soluções gerenciadas para diversos tipos de workloads, desde bancos relacionais até NoSQL, in-memory, e especializados.

## Serviços

### RDS (Relational Database Service)
Bancos de dados relacionais gerenciados.

**Engines Suportados:**
- Amazon Aurora (MySQL e PostgreSQL compatível)
- MySQL
- PostgreSQL
- MariaDB
- Oracle
- SQL Server

**Recursos principais:**
- Backups automáticos
- Multi-AZ para alta disponibilidade
- Read replicas
- Automatic patching
- Monitoring com CloudWatch

### DynamoDB
Banco de dados NoSQL totalmente gerenciado, serverless e de alta performance.

**Recursos principais:**
- Performance consistente em qualquer escala
- Fully managed (sem servidores para gerenciar)
- Global Tables (multi-region)
- DynamoDB Streams
- Point-in-time recovery
- On-demand e provisioned capacity

**Casos de Uso:**
- Aplicações web de alta escala
- Gaming leaderboards
- IoT data storage
- Mobile backends
- Session management

### Aurora
Banco de dados relacional compatível com MySQL e PostgreSQL, até 5x mais rápido.

**Recursos principais:**
- Alta disponibilidade (6 cópias dos dados em 3 AZs)
- Até 15 read replicas
- Aurora Serverless para workloads variáveis
- Global Database (multi-region)
- Backtrack (point-in-time restore sem backups)
- Performance superior ao MySQL/PostgreSQL padrão

### Redshift
Data warehouse para análise de Big Data.

**Recursos principais:**
- Análise de petabytes de dados
- Columnar storage
- Massively Parallel Processing (MPP)
- Redshift Spectrum (query S3 data)
- Concurrency Scaling
- Machine Learning integration

### ElastiCache
Cache in-memory gerenciado.

**Engines:**
- **Redis**: Estruturas de dados avançadas, persistência, replicação
- **Memcached**: Cache simples e distribuído

**Recursos principais:**
- Microsegundos de latência
- Alta disponibilidade (Redis Multi-AZ)
- Backup e restore (Redis)
- Pub/Sub messaging (Redis)

### Neptune
Banco de dados de grafos totalmente gerenciado.

**Recursos principais:**
- Suporte a Gremlin e SPARQL
- Alta disponibilidade
- Read replicas
- Point-in-time recovery
- Ideal para redes sociais, fraud detection, knowledge graphs

### DocumentDB
Banco de dados de documentos compatível com MongoDB.

**Recursos principais:**
- Compatível com MongoDB 3.6, 4.0, 5.0
- Totalmente gerenciado
- Escalabilidade automática
- Alta disponibilidade
- Backup contínuo

### Timestream
Banco de dados de séries temporais serverless.

**Recursos principais:**
- Otimizado para IoT e aplicações operacionais
- Queries 1000x mais rápidas que bancos relacionais
- Cost-effective (1/10 do custo)
- Tiering automático de dados

### Database Migration Service (DMS)
Migração de bancos de dados para AWS.

**Recursos principais:**
- Migração homogênea e heterogênea
- Replicação contínua
- Schema Conversion Tool (SCT)
- Minimal downtime
- Suporte a diversos engines

## Comparação de Serviços

| Serviço | Tipo | Uso Principal | Escalabilidade |
|---------|------|---------------|----------------|
| RDS | Relational | OLTP, aplicações tradicionais | Vertical + Read Replicas |
| Aurora | Relational | Alto desempenho OLTP | Horizontal (reads) |
| DynamoDB | NoSQL | Web scale, serverless | Horizontal ilimitada |
| Redshift | Data Warehouse | Analytics, BI | MPP |
| ElastiCache | In-Memory | Cache, session store | Horizontal |
| Neptune | Graph | Relacionamentos complexos | Horizontal |
| DocumentDB | Document | Apps que usavam MongoDB | Horizontal |
| Timestream | Time Series | IoT, monitoring | Automática |

## Escolhendo o Banco de Dados Certo

### Use RDS quando:
- Precisa de ACID compliance
- Aplicações tradicionais
- Queries SQL complexas
- Structured data

### Use DynamoDB quando:
- Necessita de escala massiva
- Latência consistente de milissegundos
- Serverless architecture
- Key-value ou document data

### Use Aurora quando:
- RDS com melhor performance
- Alta disponibilidade crítica
- Workloads variáveis (Serverless)

### Use Redshift quando:
- Data warehousing
- Analytics e BI
- Queries complexas em grandes datasets
- OLAP workloads

### Use ElastiCache quando:
- Cache de aplicação
- Session store
- Real-time analytics
- Leaderboards/Counting

## Melhores Práticas

### Segurança
- Use encryption at rest e in transit
- Implemente network isolation (VPC)
- Use IAM database authentication quando possível
- Habilite audit logging
- Regular security patching

### Performance
- Escolha o tipo de instância adequado
- Use read replicas para distribuir carga de leitura
- Implemente caching strategies
- Monitore queries lentas
- Otimize índices

### Alta Disponibilidade
- Use Multi-AZ deployments
- Configure automated backups
- Teste disaster recovery procedures
- Implemente health checks
- Use connection pooling

### Custo
- Use Reserved Instances para workloads previsíveis
- Right-size instances
- Delete recursos não utilizados
- Use Aurora Serverless para workloads variáveis
- Implemente data lifecycle policies

## Recursos de Aprendizado

- [AWS Database Services](https://aws.amazon.com/products/databases/)
- [Which Database to Use](https://aws.amazon.com/startups/start-building/how-to-choose-a-database/)
- [Database Migration Guide](https://docs.aws.amazon.com/dms/)
- [RDS Best Practices](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/CHAP_BestPractices.html)
- [DynamoDB Best Practices](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/best-practices.html)
