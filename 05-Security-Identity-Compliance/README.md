# Security, Identity & Compliance Services

## Visão Geral
Os serviços de segurança da AWS fornecem recursos para proteger dados, gerenciar identidades e manter compliance em sua infraestrutura na nuvem.

## Serviços

### IAM (Identity and Access Management)
Gerenciamento de acesso a recursos AWS.

**Componentes:**
- **Users**: Identidades permanentes
- **Groups**: Coleção de usuários
- **Roles**: Identidades assumíveis
- **Policies**: Permissões em JSON

**Recursos principais:**
- Multi-Factor Authentication (MFA)
- Identity Federation (SAML, OIDC)
- Fine-grained permissions
- Service Control Policies (SCPs) via Organizations
- Access Analyzer

**Melhores Práticas:**
- Use roles ao invés de credenciais hardcoded
- Implemente least privilege
- Habilite MFA para root e usuários privilegiados
- Rotate credentials regularmente
- Use policy conditions para controle adicional

### Cognito
Autenticação e autorização para aplicações web e mobile.

**Componentes:**
- **User Pools**: Diretório de usuários
- **Identity Pools**: Credenciais AWS temporárias

**Recursos principais:**
- Sign-up e sign-in
- Social identity providers (Google, Facebook, Amazon)
- SAML e OIDC integration
- MFA support
- User migration
- Lambda triggers para customização

**Casos de Uso:**
- Mobile apps
- Web applications
- API authentication
- User management

### KMS (Key Management Service)
Criação e gerenciamento de chaves criptográficas.

**Recursos principais:**
- Hardware Security Modules (HSMs) gerenciados
- Integração com serviços AWS
- Audit via CloudTrail
- Key rotation automática
- Suporte a multi-region keys

**Tipos de Chaves:**
- AWS Managed Keys
- Customer Managed Keys
- AWS Owned Keys
- Custom Key Stores (CloudHSM)

### Secrets Manager
Gerenciamento de segredos (senhas, API keys, tokens).

**Recursos principais:**
- Rotação automática de segredos
- Fine-grained access control
- Audit logging
- Integração com RDS, DocumentDB, Redshift
- Cross-account access

**Casos de Uso:**
- Database credentials
- API keys
- OAuth tokens
- Certificados privados

### Certificate Manager (ACM)
Provisionamento e gerenciamento de certificados SSL/TLS.

**Recursos principais:**
- Certificados gratuitos
- Renovação automática
- Integração com ELB, CloudFront, API Gateway
- Importação de certificados terceiros
- Wildcard certificates

**Ver também:**
- [Algoritmos de Criptografia](./Cryptographic-Algorithms/README.md) - Guia detalhado sobre RSA, ECC, PBES#1, PBES#2 e compatibilidade com serviços AWS

### WAF (Web Application Firewall)
Proteção de aplicações web contra exploits comuns.

**Recursos principais:**
- Rules customizáveis
- Managed rule groups
- Rate limiting
- IP blocking
- Geo-blocking
- SQL injection e XSS protection
- Integration com CloudFront, ALB, API Gateway

**Managed Rules:**
- Core Rule Set (CRS)
- Known Bad Inputs
- SQL Database
- Linux/Windows OS
- PHP/WordPress specific

### Shield
Proteção contra DDoS (Distributed Denial of Service).

**Níveis:**
- **Shield Standard**: Gratuito, proteção automática
- **Shield Advanced**: Proteção avançada, response team, cost protection

**Recursos Shield Advanced:**
- 24/7 DDoS Response Team (DRT)
- Advanced attack analytics
- Cost protection
- Layer 7 DDoS protection

### Security Hub
Dashboard centralizado de segurança e compliance.

**Recursos principais:**
- Agregação de findings de múltiplos serviços
- Compliance checks (CIS, PCI-DSS, AWS best practices)
- Integration com GuardDuty, Inspector, Macie
- Automated remediation
- Cross-account/cross-region views

### GuardDuty
Detecção de ameaças inteligente e contínua.

**Recursos principais:**
- Machine learning
- Threat intelligence feeds
- Análise de VPC Flow Logs, CloudTrail, DNS logs
- Detecção de comportamento anômalo
- Integration com EventBridge para resposta automática

**Tipos de Findings:**
- Reconnaissance (port scanning, etc.)
- Instance compromise
- Account compromise
- Bucket compromise
- Malware

### Inspector
Avaliação automatizada de segurança para aplicações.

**Recursos principais:**
- Vulnerability scanning
- Network accessibility assessment
- Package vulnerabilities (CVEs)
- Lambda function scanning
- Integration com Systems Manager

**Assessment Types:**
- EC2 instances
- Container images (ECR)
- Lambda functions

### Macie
Descoberta e proteção de dados sensíveis usando machine learning.

**Recursos principais:**
- PII detection
- S3 data classification
- Sensitive data discovery
- Compliance reporting
- Automated remediation

**Detecta:**
- Credit card numbers
- Social security numbers
- API keys
- Private keys
- PII (Personally Identifiable Information)

### AWS Config
Avaliação, auditoria e compliance de recursos AWS.

**Recursos principais:**
- Resource inventory
- Configuration history
- Change tracking
- Compliance rules
- Remediation automática
- Multi-account/multi-region

**Config Rules:**
- AWS Managed Rules
- Custom Rules (Lambda)
- Conformance packs

## Frameworks de Compliance

### Suportados pela AWS:
- **SOC 1/2/3**: Service Organization Controls
- **PCI-DSS**: Payment Card Industry Data Security Standard
- **HIPAA**: Health Insurance Portability and Accountability Act
- **GDPR**: General Data Protection Regulation
- **ISO 27001**: Information Security Management
- **FedRAMP**: Federal Risk and Authorization Management Program

## Modelo de Responsabilidade Compartilhada

### AWS Responsável por:
- Segurança **DA** nuvem
- Hardware, software, networking, facilities
- Managed services

### Cliente Responsável por:
- Segurança **NA** nuvem
- Dados, encryption
- Network configuration
- IAM
- OS patching (EC2)
- Application security

## Melhores Práticas de Segurança

### Identity & Access
1. **Use IAM roles** ao invés de access keys
2. **Implemente MFA** para usuários privilegiados
3. **Least privilege principle** em policies
4. **Regular access reviews**
5. **Separate accounts** para diferentes ambientes

### Data Protection
1. **Encrypt data at rest** (S3, EBS, RDS, etc.)
2. **Encrypt data in transit** (TLS/SSL)
3. **Use KMS** para gerenciamento de chaves
4. **Backup regular** com AWS Backup
5. **Classify data** por sensibilidade

### Infrastructure Security
1. **Use Security Groups** restritivos
2. **Private subnets** para recursos internos
3. **VPC Flow Logs** habilitados
4. **Patch management** regular
5. **Network segmentation**

### Monitoring & Detection
1. **Enable CloudTrail** em todas as regiões
2. **Use GuardDuty** para threat detection
3. **Security Hub** para visão centralizada
4. **CloudWatch Alarms** para eventos críticos
5. **Config Rules** para compliance

### Incident Response
1. **Playbooks** documentados
2. **Automated responses** via EventBridge + Lambda
3. **Regular drills**
4. **Forensics capabilities** (EC2 snapshots, memory dumps)
5. **Contact information** atualizada

## Arquitetura de Referência - Defense in Depth

```
Edge Protection:
├── WAF (Application layer)
├── Shield (DDoS protection)
└── CloudFront (with geo-restrictions)

Network Layer:
├── VPC (Network isolation)
├── Security Groups (Stateful firewall)
├── NACLs (Stateless firewall)
└── VPC Flow Logs (Network monitoring)

Identity Layer:
├── IAM (Access management)
├── Cognito (User authentication)
└── MFA (Multi-factor authentication)

Data Layer:
├── KMS (Encryption keys)
├── Secrets Manager (Credentials)
└── Encryption at rest & in transit

Detection & Response:
├── GuardDuty (Threat detection)
├── Security Hub (Central dashboard)
├── Inspector (Vulnerability scanning)
├── Macie (Data discovery)
└── Config (Compliance monitoring)
```

## Comandos Úteis

```bash
# IAM
aws iam list-users
aws iam get-user
aws iam create-role --role-name MyRole --assume-role-policy-document file://trust.json

# KMS
aws kms list-keys
aws kms create-key --description "My key"
aws kms encrypt --key-id alias/my-key --plaintext "secret"

# Secrets Manager
aws secretsmanager create-secret --name MySecret --secret-string "mysecret"
aws secretsmanager get-secret-value --secret-id MySecret

# Security Hub
aws securityhub get-findings
aws securityhub enable-security-hub

# GuardDuty
aws guardduty list-detectors
aws guardduty get-findings --detector-id <detector-id>
```

## Recursos de Aprendizado

- [AWS Security Best Practices](https://aws.amazon.com/architecture/security-identity-compliance/)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [AWS Security Hub User Guide](https://docs.aws.amazon.com/securityhub/)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)
- [AWS Security Workshops](https://workshops.aws/categories/Security)
