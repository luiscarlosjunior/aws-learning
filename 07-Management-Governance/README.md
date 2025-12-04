# Management & Governance

## Visão Geral
Os serviços de gerenciamento e governança da AWS ajudam a monitorar, gerenciar e governar recursos na nuvem.

## Serviços

### CloudWatch
Monitoramento e observabilidade de recursos e aplicações AWS.

**Componentes:**
- **Metrics**: Métricas de performance
- **Logs**: Agregação e análise de logs
- **Alarms**: Alertas baseados em thresholds
- **Events/EventBridge**: Automação baseada em eventos
- **Dashboards**: Visualização customizada
- **Insights**: Análise de logs e traces
- **Synthetics**: Monitoring de endpoints

**Métricas Padrão:**
- EC2: CPU, Network, Disk
- RDS: Connections, IOPS, Storage
- Lambda: Invocations, Duration, Errors
- ELB: Request count, Latency

### CloudFormation
Infrastructure as Code (IaC) para provisionar recursos AWS.

**Recursos principais:**
- Templates YAML/JSON
- Stacks para gerenciar recursos
- Change sets para preview
- Drift detection
- StackSets para multi-account/region
- Nested stacks
- Cross-stack references

**Componentes:**
- Parameters
- Resources
- Outputs
- Mappings
- Conditions

### Systems Manager
Gerenciamento operacional unificado.

**Recursos principais:**
- **Session Manager**: Acesso seguro a instâncias sem SSH
- **Run Command**: Executar comandos remotamente
- **Patch Manager**: Automação de patches
- **Parameter Store**: Armazenamento de configurações
- **Automation**: Workflows automatizados
- **Maintenance Windows**: Janelas de manutenção
- **State Manager**: Configuração consistente
- **OpsCenter**: Gerenciamento de OpsItems

### CloudTrail
Auditoria e logging de atividades AWS.

**Recursos principais:**
- Log de todas as API calls
- Compliance e governance
- Security analysis
- Operational troubleshooting
- Integration com CloudWatch Logs
- Multi-region trails
- Organization trails

**Event Types:**
- Management events (control plane)
- Data events (data plane - S3, Lambda)
- Insights events (anomalias)

### AWS Organizations
Gerenciamento centralizado de múltiplas contas AWS.

**Recursos principais:**
- Hierarchical organization
- Consolidated billing
- Service Control Policies (SCPs)
- Tag policies
- Backup policies
- Account creation automation

**Organizational Units (OUs):**
- Production
- Development
- Security
- Infrastructure

### Control Tower
Configuração e governança automatizada de multi-account environment.

**Recursos principais:**
- Landing zone setup
- Account vending
- Guardrails (preventive e detective)
- Dashboard centralizado
- Integration com Organizations
- Compliance monitoring

### Service Catalog
Catálogo de produtos aprovados para uso.

**Recursos principais:**
- Self-service provisioning
- Standardized templates
- Version control
- Access control
- Portfolio management
- Product constraints

**Casos de Uso:**
- IT governance
- Compliance
- Cost control
- Standardization

### Trusted Advisor
Recomendações para otimização de recursos AWS.

**Categorias:**
- **Cost Optimization**: Recursos subutilizados
- **Performance**: Gargalos de performance
- **Security**: Falhas de segurança
- **Fault Tolerance**: Melhorias de disponibilidade
- **Service Limits**: Limites de serviço

**Níveis:**
- Basic: Core checks (gratuito)
- Developer/Business: Conjunto expandido
- Enterprise: Todos os checks + API access

### Cost Explorer
Visualização e análise de custos AWS.

**Recursos principais:**
- Cost breakdown por serviço, região, tag
- Forecasting
- Reserved Instance recommendations
- Savings Plans recommendations
- Cost allocation tags
- Custom reports

## Casos de Uso

1. **Monitoramento**: CloudWatch para metrics e logs
2. **Automação de Infraestrutura**: CloudFormation
3. **Multi-account Governance**: Organizations + Control Tower
4. **Auditoria**: CloudTrail
5. **Otimização de Custos**: Cost Explorer + Trusted Advisor
6. **Patch Management**: Systems Manager
7. **Self-service Portal**: Service Catalog

## Melhores Práticas

### CloudWatch
- Configure alarms para métricas críticas
- Use log insights para análise
- Implement composite alarms
- Set appropriate retention periods
- Use metrics filters

### CloudFormation
- Use version control para templates
- Implement stack policies
- Use parameters para reutilização
- Test em non-production primeiro
- Use nested stacks para complexidade
- Implement rollback triggers

### Systems Manager
- Use Session Manager ao invés de SSH
- Automate patching
- Store secrets em Parameter Store (SecureString)
- Use Automation documents
- Implement change tracking

### CloudTrail
- Enable em todas as regiões
- Send logs para S3 com encryption
- Enable log file validation
- Integrate com CloudWatch Logs
- Set up SNS notifications

### Organizations
- Use SCPs para enforce policies
- Separate accounts por environment
- Implement tagging standards
- Use consolidated billing
- Automate account creation

### Cost Management
- Enable cost allocation tags
- Use budgets e alerts
- Regular review com Cost Explorer
- Implement Reserved Instances
- Use Savings Plans
- Right-size resources

## Arquitetura de Governança

```
AWS Organizations (Root)
├── Production OU
│   ├── Prod Account 1
│   └── Prod Account 2
├── Development OU
│   ├── Dev Account 1
│   └── Dev Account 2
└── Security OU
    └── Security Tooling Account
        ├── CloudTrail (Organization)
        ├── Security Hub
        ├── GuardDuty
        └── Config

Service Control Policies (SCPs)
├── Deny Root User
├── Require MFA
└── Restrict Regions

CloudFormation StackSets
└── Deploy baseline resources across accounts
```

## Exemplo: CloudFormation Template

```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Simple EC2 Instance'

Parameters:
  InstanceType:
    Type: String
    Default: t3.micro
    AllowedValues:
      - t3.micro
      - t3.small
      - t3.medium

Resources:
  MyEC2Instance:
    Type: AWS::EC2::Instance
    Properties:
      InstanceType: !Ref InstanceType
      ImageId: ami-0c55b159cbfafe1f0
      Tags:
        - Key: Name
          Value: MyInstance

Outputs:
  InstanceId:
    Description: Instance ID
    Value: !Ref MyEC2Instance
    Export:
      Name: !Sub '${AWS::StackName}-InstanceId'
```

## Comandos Úteis

```bash
# CloudFormation
aws cloudformation create-stack --stack-name mystack --template-body file://template.yaml
aws cloudformation describe-stacks --stack-name mystack
aws cloudformation delete-stack --stack-name mystack

# CloudWatch
aws cloudwatch put-metric-data --namespace "MyApp" --metric-name "RequestCount" --value 1
aws logs tail /aws/lambda/my-function --follow

# Systems Manager
aws ssm start-session --target i-1234567890abcdef0
aws ssm send-command --document-name "AWS-RunShellScript" --targets "Key=instanceids,Values=i-123456"

# CloudTrail
aws cloudtrail lookup-events --lookup-attributes AttributeKey=EventName,AttributeValue=RunInstances

# Organizations
aws organizations list-accounts
aws organizations create-account --email newaccount@example.com --account-name "New Account"
```

## Recursos de Aprendizado

- [AWS Management & Governance](https://aws.amazon.com/products/management-and-governance/)
- [CloudFormation Best Practices](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/best-practices.html)
- [CloudWatch User Guide](https://docs.aws.amazon.com/cloudwatch/)
- [Well-Architected Framework - Operational Excellence](https://docs.aws.amazon.com/wellarchitected/latest/operational-excellence-pillar/)
