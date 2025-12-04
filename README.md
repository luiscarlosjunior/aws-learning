# AWS Learning Repository 🚀

## 📚 Sobre este Repositório

Bem-vindo ao repositório de estudos AWS! Este projeto foi criado para organizar o aprendizado dos principais serviços da Amazon Web Services (AWS), separados por categorias conforme a organização oficial da AWS.

Cada categoria contém pastas para os serviços principais, com README.md detalhados explicando:
- O que é o serviço
- Conceitos principais
- Recursos e funcionalidades
- Casos de uso
- Melhores práticas
- Exemplos de código
- Comandos úteis
- Links para documentação oficial

## 📋 Estrutura do Repositório

### [01-Compute](./01-Compute/)
Serviços de computação para executar aplicações e processar workloads.
- **[EC2](./01-Compute/EC2/)** - Elastic Compute Cloud (Servidores virtuais)
- **[Lambda](./01-Compute/Lambda/)** - Computação serverless
- **[ECS](./01-Compute/ECS/)** - Elastic Container Service
- **[EKS](./01-Compute/EKS/)** - Elastic Kubernetes Service
- **[Elastic Beanstalk](./01-Compute/Elastic-Beanstalk/)** - PaaS para deploy de aplicações
- **[Lightsail](./01-Compute/Lightsail/)** - VPS simples e de baixo custo
- **[Batch](./01-Compute/Batch/)** - Processamento batch
- **[Fargate](./01-Compute/Fargate/)** - Containers serverless

### [02-Storage](./02-Storage/)
Soluções de armazenamento escaláveis e duráveis.
- **[S3](./02-Storage/S3/)** - Simple Storage Service (Object storage)
- **[EBS](./02-Storage/EBS/)** - Elastic Block Store (Block storage)
- **[EFS](./02-Storage/EFS/)** - Elastic File System (File storage)
- **[Glacier](./02-Storage/Glacier/)** - Arquivamento de longo prazo
- **[Storage Gateway](./02-Storage/Storage-Gateway/)** - Hybrid storage
- **[AWS Backup](./02-Storage/AWS-Backup/)** - Backup centralizado
- **[FSx](./02-Storage/FSx/)** - Sistemas de arquivos gerenciados

### [03-Database](./03-Database/)
Bancos de dados gerenciados para diversos tipos de workloads.
- **[RDS](./03-Database/RDS/)** - Relational Database Service
- **[DynamoDB](./03-Database/DynamoDB/)** - NoSQL database
- **[Aurora](./03-Database/Aurora/)** - MySQL/PostgreSQL compatível
- **[Redshift](./03-Database/Redshift/)** - Data warehouse
- **[ElastiCache](./03-Database/ElastiCache/)** - In-memory cache (Redis/Memcached)
- **[Neptune](./03-Database/Neptune/)** - Graph database
- **[DocumentDB](./03-Database/DocumentDB/)** - MongoDB compatível
- **[Timestream](./03-Database/Timestream/)** - Time series database
- **[DMS](./03-Database/Database-Migration-Service/)** - Database Migration Service

### [04-Networking-Content-Delivery](./04-Networking-Content-Delivery/)
Serviços de rede e entrega de conteúdo.
- **[VPC](./04-Networking-Content-Delivery/VPC/)** - Virtual Private Cloud
- **[CloudFront](./04-Networking-Content-Delivery/CloudFront/)** - Content Delivery Network
- **[Route 53](./04-Networking-Content-Delivery/Route-53/)** - DNS service
- **[API Gateway](./04-Networking-Content-Delivery/API-Gateway/)** - API management
- **[Direct Connect](./04-Networking-Content-Delivery/Direct-Connect/)** - Conexão dedicada
- **[ELB](./04-Networking-Content-Delivery/Elastic-Load-Balancing/)** - Load balancing
- **[Transit Gateway](./04-Networking-Content-Delivery/Transit-Gateway/)** - Network hub
- **[Global Accelerator](./04-Networking-Content-Delivery/Global-Accelerator/)** - Network optimization

### [05-Security-Identity-Compliance](./05-Security-Identity-Compliance/)
Serviços de segurança, identidade e compliance.
- **[IAM](./05-Security-Identity-Compliance/IAM/)** - Identity and Access Management
- **[Cognito](./05-Security-Identity-Compliance/Cognito/)** - User authentication
- **[KMS](./05-Security-Identity-Compliance/KMS/)** - Key Management Service
- **[Secrets Manager](./05-Security-Identity-Compliance/Secrets-Manager/)** - Secrets management
- **[Certificate Manager](./05-Security-Identity-Compliance/Certificate-Manager/)** - SSL/TLS certificates
- **[WAF](./05-Security-Identity-Compliance/WAF/)** - Web Application Firewall
- **[Shield](./05-Security-Identity-Compliance/Shield/)** - DDoS protection
- **[Security Hub](./05-Security-Identity-Compliance/Security-Hub/)** - Security dashboard
- **[GuardDuty](./05-Security-Identity-Compliance/GuardDuty/)** - Threat detection
- **[Inspector](./05-Security-Identity-Compliance/Inspector/)** - Security assessment
- **[Macie](./05-Security-Identity-Compliance/Macie/)** - Data discovery
- **[Config](./05-Security-Identity-Compliance/AWS-Config/)** - Configuration management

### [06-Developer-Tools](./06-Developer-Tools/)
Ferramentas para desenvolvimento e CI/CD.
- **[CodeCommit](./06-Developer-Tools/CodeCommit/)** - Git repositories
- **[CodeBuild](./06-Developer-Tools/CodeBuild/)** - Build service
- **[CodeDeploy](./06-Developer-Tools/CodeDeploy/)** - Deployment automation
- **[CodePipeline](./06-Developer-Tools/CodePipeline/)** - CI/CD orchestration
- **[Cloud9](./06-Developer-Tools/Cloud9/)** - Cloud IDE
- **[X-Ray](./06-Developer-Tools/X-Ray/)** - Distributed tracing
- **[CodeArtifact](./06-Developer-Tools/CodeArtifact/)** - Artifact repository
- **[CodeGuru](./06-Developer-Tools/CodeGuru/)** - Code review e profiling

### [07-Management-Governance](./07-Management-Governance/)
Gerenciamento e governança de recursos.
- **[CloudWatch](./07-Management-Governance/CloudWatch/)** - Monitoring e observability
- **[CloudFormation](./07-Management-Governance/CloudFormation/)** - Infrastructure as Code
- **[Systems Manager](./07-Management-Governance/Systems-Manager/)** - Operations management
- **[CloudTrail](./07-Management-Governance/CloudTrail/)** - Audit logging
- **[Organizations](./07-Management-Governance/AWS-Organizations/)** - Multi-account management
- **[Control Tower](./07-Management-Governance/Control-Tower/)** - Account setup automation
- **[Service Catalog](./07-Management-Governance/Service-Catalog/)** - Product catalog
- **[Trusted Advisor](./07-Management-Governance/Trusted-Advisor/)** - Best practices recommendations
- **[Cost Explorer](./07-Management-Governance/Cost-Explorer/)** - Cost analysis

### [08-Application-Integration](./08-Application-Integration/)
Integração de aplicações e mensageria.
- **[SQS](./08-Application-Integration/SQS/)** - Message queuing
- **[SNS](./08-Application-Integration/SNS/)** - Pub/Sub messaging
- **[EventBridge](./08-Application-Integration/EventBridge/)** - Event bus
- **[Step Functions](./08-Application-Integration/Step-Functions/)** - Workflow orchestration
- **[AppSync](./08-Application-Integration/App-Sync/)** - GraphQL APIs
- **[MQ](./08-Application-Integration/MQ/)** - Managed message broker
- **[SWF](./08-Application-Integration/Simple-Workflow/)** - Workflow service (legacy)

### [09-Analytics](./09-Analytics/)
Processamento e análise de dados.
- **[Athena](./09-Analytics/Athena/)** - Interactive query service
- **[EMR](./09-Analytics/EMR/)** - Big data platform
- **[Kinesis](./09-Analytics/Kinesis/)** - Real-time streaming
- **[Glue](./09-Analytics/Glue/)** - ETL service
- **[QuickSight](./09-Analytics/QuickSight/)** - Business Intelligence
- **[Data Pipeline](./09-Analytics/Data-Pipeline/)** - Data workflow (legacy)
- **[MSK](./09-Analytics/Managed-Streaming-Kafka/)** - Managed Kafka

### [10-Machine-Learning](./10-Machine-Learning/)
Serviços de inteligência artificial e machine learning.
- **[SageMaker](./10-Machine-Learning/SageMaker/)** - ML platform completo
- **[Rekognition](./10-Machine-Learning/Rekognition/)** - Image e video analysis
- **[Comprehend](./10-Machine-Learning/Comprehend/)** - Natural Language Processing
- **[Polly](./10-Machine-Learning/Polly/)** - Text-to-Speech
- **[Lex](./10-Machine-Learning/Lex/)** - Conversational AI
- **[Transcribe](./10-Machine-Learning/Transcribe/)** - Speech-to-Text
- **[Translate](./10-Machine-Learning/Translate/)** - Language translation
- **[Fraud Detector](./10-Machine-Learning/Fraud-Detector/)** - Fraud detection
- **[Personalize](./10-Machine-Learning/Personalize/)** - Recommendation engine

### [11-Containers](./11-Containers/)
Serviços para executar aplicações containerizadas.
- **[ECS](./11-Containers/ECS/)** - Elastic Container Service
- **[EKS](./11-Containers/EKS/)** - Elastic Kubernetes Service
- **[ECR](./11-Containers/ECR/)** - Container registry
- **[Fargate](./11-Containers/Fargate/)** - Serverless containers
- **[App Runner](./11-Containers/App-Runner/)** - Fully managed container service

### [12-Serverless](./12-Serverless/)
Arquiteturas e padrões serverless.
- **[Lambda](./12-Serverless/Lambda/)** - Serverless compute
- **[API Gateway](./12-Serverless/API-Gateway/)** - API management
- **[DynamoDB](./12-Serverless/DynamoDB/)** - Serverless database
- **[Step Functions](./12-Serverless/Step-Functions/)** - Workflow orchestration
- **[EventBridge](./12-Serverless/EventBridge/)** - Event-driven architecture
- **[AppSync](./12-Serverless/AppSync/)** - GraphQL serverless

## 🎯 Como Usar Este Repositório

1. **Navegue pelas Categorias**: Escolha a categoria de serviços que deseja estudar
2. **Explore os Serviços**: Entre nas pastas individuais de cada serviço
3. **Leia os READMEs**: Cada serviço tem documentação detalhada em português
4. **Pratique**: Use os exemplos de código fornecidos
5. **Consulte Links**: Acesse a documentação oficial para aprofundamento

## 📖 Recursos Adicionais

### Certificações AWS
- **AWS Certified Cloud Practitioner** - Fundamentos básicos
- **AWS Certified Solutions Architect Associate** - Design de arquiteturas
- **AWS Certified Developer Associate** - Desenvolvimento de aplicações
- **AWS Certified SysOps Administrator Associate** - Operações
- **Certificações Professional** - Arquiteto e DevOps avançado
- **Certificações Specialty** - Especializações (Security, ML, Database, etc.)

### Links Úteis
- [AWS Documentation](https://docs.aws.amazon.com/) - Documentação oficial
- [AWS Training](https://aws.amazon.com/training/) - Treinamentos gratuitos e pagos
- [AWS Free Tier](https://aws.amazon.com/free/) - Tier gratuito para prática
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/) - Melhores práticas
- [AWS Workshops](https://workshops.aws/) - Workshops práticos
- [AWS Samples](https://github.com/aws-samples) - Exemplos de código
- [AWS Blog](https://aws.amazon.com/blogs/) - Blog oficial
- [AWS re:Invent](https://reinvent.awsevents.com/) - Conferência anual

### Comunidade
- [AWS Community Builders](https://aws.amazon.com/developer/community/community-builders/)
- [AWS User Groups](https://aws.amazon.com/developer/community/usergroups/)
- [Stack Overflow - AWS](https://stackoverflow.com/questions/tagged/amazon-web-services)
- [Reddit - r/aws](https://www.reddit.com/r/aws/)

## 🚀 Começando com AWS

### Primeiros Passos
1. Crie uma conta AWS (Free Tier disponível)
2. Configure MFA na conta root
3. Crie um usuário IAM para uso diário
4. Instale AWS CLI
5. Configure credenciais locais
6. Explore os serviços básicos (S3, EC2, Lambda)

### Práticas Essenciais
- **Segurança**: Sempre use IAM roles, habilite MFA, criptografe dados
- **Custo**: Use billing alerts, monitore gastos, aproveite Free Tier
- **Confiabilidade**: Multi-AZ, backups, disaster recovery
- **Performance**: Choose right service, monitor metrics
- **Operação**: Automate, use IaC, implement CI/CD

## 📝 Contribuindo

Este é um repositório de estudos pessoal, mas sugestões e melhorias são bem-vindas! Sinta-se à vontade para:
- Reportar erros ou informações desatualizadas
- Sugerir novos conteúdos
- Compartilhar recursos úteis
- Adicionar exemplos práticos

## 📄 Licença

Este repositório é para fins educacionais. O conteúdo é baseado na documentação oficial da AWS e em melhores práticas da comunidade.

## ⚠️ Aviso

Sempre consulte a documentação oficial da AWS para informações mais atualizadas e detalhadas. Os preços e recursos podem mudar ao longo do tempo.

---

**Happy Learning! 🎓**

*Última atualização: Dezembro 2024*