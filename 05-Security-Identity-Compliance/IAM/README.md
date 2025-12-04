# AWS IAM (Identity and Access Management)

## O que é IAM?

AWS IAM é um serviço que permite gerenciar com segurança o acesso aos serviços e recursos da AWS. Com IAM, você pode criar e gerenciar usuários, grupos e permissões.

## Conceitos Principais

### Users
Identidades permanentes que representam pessoas ou aplicações.

**Características:**
- Credenciais permanentes
- Console access (username/password)
- Programmatic access (access keys)
- MFA support

### Groups
Coleções de usuários com permissões compartilhadas.

**Exemplo:**
```
Group: Developers
├── User: Alice
├── User: Bob
└── Policy: DeveloperAccess
```

### Roles
Identidades assumíveis temporariamente.

**Casos de Uso:**
- EC2 instances
- Lambda functions
- Cross-account access
- Federation
- Service-to-service

**Características:**
- Credenciais temporárias
- Não tem senha ou access keys
- Assumidas via STS (Security Token Service)

### Policies
Documentos JSON que definem permissões.

**Tipos:**
1. **AWS Managed Policies**: Criadas e gerenciadas pela AWS
2. **Customer Managed Policies**: Criadas por você
3. **Inline Policies**: Anexadas diretamente a user/group/role

### Trust Policies
Definem quem pode assumir uma role.

## Estrutura de Policy

### Policy Document

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowS3Read",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::my-bucket",
        "arn:aws:s3:::my-bucket/*"
      ],
      "Condition": {
        "IpAddress": {
          "aws:SourceIp": "203.0.113.0/24"
        }
      }
    }
  ]
}
```

### Elementos

**Version**: Sempre "2012-10-17"

**Statement**: Array de permissões
- **Sid**: Identificador (opcional)
- **Effect**: "Allow" ou "Deny"
- **Principal**: Quem (em resource-based policies)
- **Action**: O que pode fazer
- **Resource**: Em quais recursos
- **Condition**: Quando aplicar (opcional)

## Actions

### Wildcards
```json
"Action": [
  "s3:*",              // Todas as ações S3
  "s3:Get*",           // Todas as ações Get
  "ec2:Describe*"      // Todas as ações Describe
]
```

### Exemplos Comuns
```json
// S3
"s3:GetObject"
"s3:PutObject"
"s3:DeleteObject"
"s3:ListBucket"

// EC2
"ec2:RunInstances"
"ec2:TerminateInstances"
"ec2:DescribeInstances"

// Lambda
"lambda:InvokeFunction"
"lambda:CreateFunction"
"lambda:UpdateFunctionCode"

// DynamoDB
"dynamodb:GetItem"
"dynamodb:PutItem"
"dynamodb:Query"
"dynamodb:Scan"
```

## Resources

### ARN Format
```
arn:partition:service:region:account-id:resource-type/resource-id
```

### Exemplos
```json
"Resource": [
  "arn:aws:s3:::my-bucket",                              // Bucket
  "arn:aws:s3:::my-bucket/*",                           // Objects
  "arn:aws:dynamodb:us-east-1:123456789012:table/MyTable",  // Table
  "arn:aws:ec2:us-east-1:123456789012:instance/*",      // Any instance
  "*"                                                    // All resources
]
```

## Conditions

### Operadores
- **StringEquals**: String exata
- **StringLike**: Pattern matching
- **NumericLessThan**: Comparação numérica
- **DateGreaterThan**: Comparação de data
- **Bool**: Booleano
- **IpAddress**: CIDR block
- **ArnLike**: ARN pattern

### Exemplos

**Restringir por IP:**
```json
"Condition": {
  "IpAddress": {
    "aws:SourceIp": ["203.0.113.0/24", "198.51.100.0/24"]
  }
}
```

**Exigir MFA:**
```json
"Condition": {
  "Bool": {
    "aws:MultiFactorAuthPresent": "true"
  }
}
```

**Restringir por região:**
```json
"Condition": {
  "StringEquals": {
    "aws:RequestedRegion": ["us-east-1", "us-west-2"]
  }
}
```

**Horário comercial:**
```json
"Condition": {
  "DateGreaterThan": {"aws:CurrentTime": "2024-01-01T09:00:00Z"},
  "DateLessThan": {"aws:CurrentTime": "2024-01-01T17:00:00Z"}
}
```

## Policy Evaluation Logic

### Ordem de Avaliação
1. **Explicit Deny**: Sempre tem precedência
2. **Organizations SCPs**: Service Control Policies
3. **Resource-based Policies**: S3 bucket policies, etc.
4. **IAM Permissions Boundaries**: Limite máximo
5. **Identity-based Policies**: User/Group/Role policies

### Regra de Ouro
```
Deny by default
→ Explicit Allow grants access
→ Explicit Deny overrides everything
```

## MFA (Multi-Factor Authentication)

### Tipos Suportados
- Virtual MFA (Google Authenticator, Authy)
- U2F Security Key (YubiKey)
- Hardware MFA

### Enforcement

**Exigir MFA para actions sensíveis:**
```json
{
  "Effect": "Allow",
  "Action": "ec2:TerminateInstances",
  "Resource": "*",
  "Condition": {
    "Bool": {"aws:MultiFactorAuthPresent": "true"}
  }
}
```

## IAM Roles para Serviços

### EC2 Instance Role

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:GetObject",
      "s3:PutObject"
    ],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

**Attach to EC2:**
```bash
aws iam create-role --role-name MyEC2Role --assume-role-policy-document file://trust-policy.json
aws iam put-role-policy --role-name MyEC2Role --policy-name S3Access --policy-document file://policy.json
aws ec2 run-instances --iam-instance-profile Name=MyEC2Role ...
```

### Lambda Execution Role

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "logs:CreateLogGroup",
      "logs:CreateLogStream",
      "logs:PutLogEvents"
    ],
    "Resource": "arn:aws:logs:*:*:*"
  }]
}
```

## Cross-Account Access

### Scenario: Account A acessa recursos em Account B

**Account B (Resource Account) - Trust Policy:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {
      "AWS": "arn:aws:iam::ACCOUNT-A-ID:role/CrossAccountRole"
    },
    "Action": "sts:AssumeRole"
  }]
}
```

**Account A - Assume Role:**
```bash
aws sts assume-role \
  --role-arn arn:aws:iam::ACCOUNT-B-ID:role/ResourceAccessRole \
  --role-session-name my-session
```

## Permission Boundaries

Limite máximo de permissões que uma entidade pode ter.

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "s3:*",
      "cloudwatch:*",
      "logs:*"
    ],
    "Resource": "*"
  }]
}
```

**Use Cases:**
- Delegação segura de criação de IAM users
- Prevenir privilege escalation
- Compliance

## Service Control Policies (SCPs)

Políticas da AWS Organizations que limitam permissões em contas.

**Deny uso de regiões específicas:**
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Deny",
    "Action": "*",
    "Resource": "*",
    "Condition": {
      "StringNotEquals": {
        "aws:RequestedRegion": ["us-east-1", "us-west-2"]
      }
    }
  }]
}
```

## IAM Access Analyzer

Identifica recursos compartilhados externamente.

**Findings:**
- S3 buckets públicos
- IAM roles assumíveis externamente
- KMS keys compartilhadas
- Lambda functions com permissions externas

## Credential Report

CSV com todos os usuários e status de credenciais.

```bash
aws iam generate-credential-report
aws iam get-credential-report
```

**Informações:**
- User creation date
- Password last used
- Password last changed
- MFA active
- Access key last used

## Melhores Práticas

### Security

1. **Enable MFA** para root e usuários privilegiados
2. **Use Roles** ao invés de access keys
3. **Least Privilege**: Dê apenas permissões necessárias
4. **Rotate Credentials** regularmente
5. **Use CloudTrail** para audit logging
6. **Password Policy** forte
7. **No Root User** para day-to-day tasks
8. **IAM Access Analyzer** habilitado

### Organization

1. **Groups**: Use para gerenciar permissões
2. **Naming Convention**: Consistente e descritiva
3. **Tagging**: Tag resources para tracking
4. **Documentation**: Documente políticas customizadas
5. **Regular Reviews**: Audit permissões periodicamente

### Development

1. **Use Roles** para EC2/Lambda/ECS
2. **Temporary Credentials** via STS
3. **Environment Variables** para credentials
4. **Never Hardcode** credentials
5. **Use SDK** credential chain

## Casos de Uso

### 1. Developer Access
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:*",
      "s3:*",
      "lambda:*",
      "logs:*"
    ],
    "Resource": "*",
    "Condition": {
      "StringEquals": {
        "aws:RequestedRegion": "us-east-1"
      }
    }
  }]
}
```

### 2. Read-Only Access
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": [
      "ec2:Describe*",
      "s3:Get*",
      "s3:List*",
      "cloudwatch:Get*",
      "cloudwatch:List*"
    ],
    "Resource": "*"
  }]
}
```

### 3. S3 Bucket Admin
```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::my-bucket",
      "arn:aws:s3:::my-bucket/*"
    ]
  }]
}
```

## Troubleshooting

### Access Denied

**Verificações:**
1. Policy tem Allow explícito?
2. Há Deny explícito em algum lugar?
3. SCP permitindo?
4. Permission boundary limitando?
5. Resource-based policy bloqueando?

### Policy Simulator
Teste policies antes de aplicar.

```bash
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:user/Alice \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::my-bucket/file.txt
```

### CloudTrail
Veja quem fez o quê e quando.

```bash
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances
```

## Comandos Úteis

```bash
# Usuários
aws iam list-users
aws iam create-user --user-name alice
aws iam delete-user --user-name alice

# Groups
aws iam list-groups
aws iam create-group --group-name developers
aws iam add-user-to-group --user-name alice --group-name developers

# Policies
aws iam list-policies --scope Local
aws iam create-policy --policy-name MyPolicy --policy-document file://policy.json
aws iam attach-user-policy --user-name alice --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# Roles
aws iam list-roles
aws iam create-role --role-name MyRole --assume-role-policy-document file://trust-policy.json
aws iam attach-role-policy --role-name MyRole --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Access Keys
aws iam create-access-key --user-name alice
aws iam list-access-keys --user-name alice
aws iam delete-access-key --user-name alice --access-key-id AKIAIOSFODNN7EXAMPLE

# MFA
aws iam enable-mfa-device --user-name alice --serial-number arn:aws:iam::123456789012:mfa/alice --authentication-code-1 123456 --authentication-code-2 789012

# Credential Report
aws iam generate-credential-report
aws iam get-credential-report --output text > report.csv
```

## Exemplo: Python Boto3

```python
import boto3

iam = boto3.client('iam')

# Create user
iam.create_user(UserName='alice')

# Create access key
response = iam.create_access_key(UserName='alice')
access_key = response['AccessKey']['AccessKeyId']
secret_key = response['AccessKey']['SecretAccessKey']

# Attach policy
iam.attach_user_policy(
    UserName='alice',
    PolicyArn='arn:aws:iam::aws:policy/ReadOnlyAccess'
)

# List user policies
response = iam.list_attached_user_policies(UserName='alice')
for policy in response['AttachedPolicies']:
    print(policy['PolicyName'])

# Assume role
sts = boto3.client('sts')
response = sts.assume_role(
    RoleArn='arn:aws:iam::123456789012:role/MyRole',
    RoleSessionName='session1'
)
credentials = response['Credentials']
```

## Recursos de Aprendizado

- [IAM User Guide](https://docs.aws.amazon.com/IAM/latest/UserGuide/)
- [IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [IAM Policy Simulator](https://policysim.aws.amazon.com/)
- [IAM Policy Reference](https://docs.aws.amazon.com/service-authorization/latest/reference/)
- [AWS Security Blog](https://aws.amazon.com/blogs/security/)
