# Storage Services

## Visão Geral
Os serviços de armazenamento da AWS oferecem soluções escaláveis, duráveis e seguras para diversos tipos de dados e casos de uso.

## Serviços

### S3 (Simple Storage Service)
Armazenamento de objetos escalável e durável para qualquer tipo de dados.

**Recursos principais:**
- Armazenamento ilimitado
- 11 noves de durabilidade (99.999999999%)
- Classes de armazenamento variadas
- Versionamento e lifecycle policies
- Encryption e segurança

**Classes de Armazenamento:**
- S3 Standard
- S3 Intelligent-Tiering
- S3 Standard-IA (Infrequent Access)
- S3 One Zone-IA
- S3 Glacier Instant Retrieval
- S3 Glacier Flexible Retrieval
- S3 Glacier Deep Archive

### EBS (Elastic Block Store)
Volumes de armazenamento em bloco persistente para instâncias EC2.

**Recursos principais:**
- Alto desempenho
- Snapshots incrementais
- Encryption
- Diferentes tipos de volume (SSD, HDD)
- Multi-Attach (io2)

**Tipos de Volume:**
- gp3/gp2: SSD de propósito geral
- io2/io1: SSD de alto desempenho (IOPS provisionado)
- st1: HDD otimizado para throughput
- sc1: HDD cold storage

### EFS (Elastic File System)
Sistema de arquivos NFS totalmente gerenciado e elástico.

**Recursos principais:**
- Escalabilidade automática
- Acesso concorrente de múltiplas instâncias
- Alta disponibilidade e durabilidade
- Classes de armazenamento (Standard, IA)
- Performance modes (General Purpose, Max I/O)

### Glacier
Armazenamento de longo prazo de baixo custo para arquivamento e backup.

**Recursos principais:**
- Custo extremamente baixo
- Durabilidade de 11 noves
- Diferentes opções de recuperação
- Vault Lock para compliance

**Opções de Recuperação:**
- Expedited: 1-5 minutos
- Standard: 3-5 horas
- Bulk: 5-12 horas

### Storage Gateway
Híbrido entre on-premises e nuvem, conecta infraestrutura local com armazenamento AWS.

**Tipos:**
- **File Gateway**: NFS/SMB para S3
- **Volume Gateway**: iSCSI para EBS snapshots
- **Tape Gateway**: Backup virtual para Glacier

### AWS Backup
Serviço centralizado para backup e restauração de recursos AWS.

**Recursos principais:**
- Backup automático
- Políticas centralizadas
- Cross-region e cross-account backup
- Compliance reporting
- Suporte a múltiplos serviços AWS

### FSx
Sistemas de arquivos totalmente gerenciados.

**Tipos:**
- **FSx for Windows File Server**: SMB protocol
- **FSx for Lustre**: Alto desempenho para HPC
- **FSx for NetApp ONTAP**: Multi-protocol
- **FSx for OpenZFS**: Linux workloads

## Casos de Uso

1. **Backup e Arquivamento**: Glacier, AWS Backup
2. **Data Lakes**: S3
3. **Aplicações Web**: S3, CloudFront
4. **Databases**: EBS
5. **File Sharing**: EFS, FSx
6. **Hybrid Storage**: Storage Gateway
7. **Content Distribution**: S3, CloudFront
8. **Big Data Analytics**: S3, FSx for Lustre

## Melhores Práticas

### S3
- Use lifecycle policies para otimizar custos
- Habilite versionamento para dados críticos
- Implemente encryption (SSE-S3, SSE-KMS, SSE-C)
- Use S3 Intelligent-Tiering para dados com padrões de acesso desconhecidos
- Configure replicação cross-region para DR

### EBS
- Crie snapshots regulares
- Use encryption
- Escolha o tipo de volume adequado para workload
- Monitore IOPS e throughput
- Delete volumes não utilizados

### EFS
- Use IA storage class para dados acessados com pouca frequência
- Configure backup automático
- Implemente access points para controle de acesso
- Use encryption em trânsito e at rest

### Geral
- Implemente políticas de retenção
- Use tags para organização e cost tracking
- Monitore com CloudWatch
- Implemente least privilege access
- Teste procedimentos de restauração

## Comparação de Serviços

| Serviço | Tipo | Uso | Latência | Custo |
|---------|------|-----|----------|-------|
| S3 | Object | Arquivos, backup | Milissegundos | $ |
| EBS | Block | EC2 volumes | Submilissegundos | $$ |
| EFS | File | Shared storage | Milissegundos | $$$ |
| Glacier | Object | Archive | Minutos/Horas | ¢ |

## Recursos de Aprendizado

- [AWS Storage Services Overview](https://aws.amazon.com/products/storage/)
- [S3 Best Practices](https://docs.aws.amazon.com/AmazonS3/latest/userguide/best-practices.html)
- [EBS Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ebs-best-practices.html)
- [Storage Gateway User Guide](https://docs.aws.amazon.com/storagegateway/)
