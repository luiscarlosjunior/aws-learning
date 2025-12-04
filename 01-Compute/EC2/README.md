# Amazon EC2 (Elastic Compute Cloud)

## O que é EC2?

Amazon EC2 é um serviço web que fornece capacidade de computação redimensionável na nuvem. Permite executar servidores virtuais (instâncias) com diversos sistemas operacionais e configurações.

## Conceitos Principais

### Instâncias
Servidores virtuais na nuvem AWS.

**Tipos de Instância:**
- **General Purpose** (T3, T4g, M5, M6i): Balanceamento entre computação, memória e rede
- **Compute Optimized** (C5, C6i): Alto desempenho de processador
- **Memory Optimized** (R5, X1): Grandes quantidades de memória
- **Storage Optimized** (I3, D2): Alto IOPS e throughput
- **Accelerated Computing** (P3, G4): GPU para ML e gráficos

### Modelos de Compra

1. **On-Demand**: Pagamento por hora/segundo sem compromisso
2. **Reserved Instances**: Desconto de até 75% com compromisso de 1-3 anos
3. **Spot Instances**: Desconto de até 90% usando capacidade ociosa
4. **Savings Plans**: Flexibilidade com descontos significativos
5. **Dedicated Hosts**: Servidor físico dedicado

### AMI (Amazon Machine Image)
Template que contém configuração de software (SO, aplicações, etc.) para lançar instâncias.

### Security Groups
Firewall virtual que controla tráfego de entrada e saída das instâncias.

### EBS (Elastic Block Store)
Volumes de armazenamento persistente para instâncias EC2.

**Tipos de Volume:**
- **gp3/gp2**: SSD de propósito geral
- **io2/io1**: SSD de alto desempenho
- **st1**: HDD otimizado para throughput
- **sc1**: HDD cold storage

### Elastic IP
Endereço IPv4 estático para nuvem dinâmica.

## Recursos

### Auto Scaling
Ajusta automaticamente o número de instâncias baseado em demanda.

**Componentes:**
- Launch Configuration/Template
- Auto Scaling Group
- Scaling Policies
- Health Checks

### Load Balancing
Distribui tráfego entre múltiplas instâncias.

**Tipos:**
- Application Load Balancer (ALB) - Layer 7
- Network Load Balancer (NLB) - Layer 4
- Gateway Load Balancer (GWLB)
- Classic Load Balancer (deprecated)

### Placement Groups
Controla posicionamento de instâncias para otimizar comunicação.

**Tipos:**
- **Cluster**: Baixa latência, alta taxa de transferência
- **Spread**: Instâncias em hardware separado
- **Partition**: Grupos de instâncias isoladas

## Casos de Uso

1. **Hospedagem de Aplicações Web**
2. **Ambientes de Desenvolvimento/Teste**
3. **Processamento de Big Data**
4. **Gaming Servers**
5. **Aplicações Enterprise**
6. **Machine Learning Training**

## Melhores Práticas

### Segurança
- Use Security Groups restritivos
- Implemente IAM roles (não use credenciais hardcoded)
- Habilite encryption em volumes EBS
- Use Systems Manager Session Manager ao invés de SSH exposto
- Mantenha patches e atualizações em dia

### Performance
- Escolha o tipo de instância adequado
- Use Enhanced Networking
- Monitore com CloudWatch
- Otimize placement groups para aplicações distribuídas

### Custo
- Use Reserved Instances para workloads previsíveis
- Aproveite Spot Instances para workloads tolerantes a falhas
- Implemente Auto Scaling para otimizar recursos
- Utilize rightsizing recommendations
- Pare instâncias quando não estiverem em uso

### Disponibilidade
- Distribua instâncias em múltiplas AZs
- Use Auto Scaling Groups
- Implemente health checks
- Configure backups regulares (snapshots EBS)

## Exemplo Básico

### Lançar uma Instância via AWS CLI

```bash
aws ec2 run-instances \
    --image-id ami-0c55b159cbfafe1f0 \
    --instance-type t3.micro \
    --key-name MyKeyPair \
    --security-group-ids sg-0123456789abcdef0 \
    --subnet-id subnet-0123456789abcdef0 \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MyInstance}]'
```

### User Data Script

```bash
#!/bin/bash
yum update -y
yum install -y httpd
systemctl start httpd
systemctl enable httpd
echo "<h1>Hello from EC2</h1>" > /var/www/html/index.html
```

## Monitoramento

### CloudWatch Metrics (padrão)
- CPU Utilization
- Network In/Out
- Disk Read/Write Operations
- Status Checks

### CloudWatch Metrics (detalhado)
- Memory Utilization (requer CloudWatch Agent)
- Disk Space Utilization
- Swap Usage
- Process Count

## Troubleshooting Comum

### Instância não inicia (pending indefinidamente)
**Problema**: Instância fica em pending
**Soluções**:
- Verificar limites de conta (vCPU limits)
- Verificar se AMI está disponível na região
- Verificar se subnet tem capacidade
- Verificar se tipo de instância está disponível na AZ
- Conferir service health dashboard da AWS

### Não consigo conectar via SSH/RDP
**Problema**: Connection timeout ou refused
**Soluções**:
- Verificar Security Group permite porta 22 (SSH) ou 3389 (RDP)
- Verificar Network ACL
- Conferir se instância tem IP público ou Elastic IP
- Verificar se key pair está correto
- Verificar route table e internet gateway
- Usar Session Manager como alternativa

### Instância com CPU 100% constante
**Problema**: High CPU utilization
**Soluções**:
- Investigar processos com `top` ou Task Manager
- Verificar se há processos maliciosos
- Considerar tipo de instância maior (scale up)
- Implementar Auto Scaling (scale out)
- Otimizar aplicação
- Usar CloudWatch para análise detalhada

### Status Check Failed
**Problema**: System ou Instance status check falhou
**Soluções**:
- **System Check**: Problema com AWS infrastructure - stop/start para mover para novo host
- **Instance Check**: Problema com OS/configuração - debug via console logs, usar EC2 Rescue
- Verificar CloudWatch Logs
- Verificar EBS volume health

### EBS Volume não anexa
**Problema**: Não consegue anexar volume
**Soluções**:
- Verificar se volume e instância estão na mesma AZ
- Verificar limites de volumes por instância
- Verificar se volume já está attached
- Confirmar que device name está disponível
- Verificar permissões IAM

## Perguntas e Respostas (Q&A)

### Conceitos Básicos

**P: Qual a diferença entre Stop e Terminate?**
R: 
- **Stop**: Desliga a instância mas mantém EBS volumes e configuração. Pode reiniciar depois. Não é cobrado por compute, apenas por storage.
- **Terminate**: Deleta a instância permanentemente. EBS root volume é deletado (a menos que configurado para persist). Irreversível.

**P: Diferença entre Public IP, Private IP e Elastic IP?**
R:
- **Private IP**: IP interno da VPC, não muda, usado para comunicação interna
- **Public IP**: IP público atribuído automaticamente, muda quando instância é stopped/started
- **Elastic IP**: IP público estático, não muda, persiste após stop, cobrado se não associado

**P: O que é um Security Group?**
R: Firewall virtual stateful que controla tráfego. Regras permitem apenas (não há deny). Se você permite tráfego de entrada, resposta de saída é automaticamente permitida. Pode ter múltiplos SGs por instância.

**P: User Data vs EC2 Instance Connect vs Systems Manager?**
R:
- **User Data**: Script executado no boot inicial (bootstrap)
- **EC2 Instance Connect**: SSH temporário via console (sem key pair)
- **Systems Manager**: Gerenciamento remoto sem SSH (mais seguro)

### Tipos de Instância

**P: Como escolher o tipo de instância certo?**
R: Depende do workload:
- **General Purpose (T3, M5)**: Aplicações web, dev/test, balanceado
- **Compute Optimized (C5, C6i)**: Batch processing, high-performance web servers, gaming
- **Memory Optimized (R5, X1)**: Databases in-memory, big data, caching
- **Storage Optimized (I3, D2)**: NoSQL databases, data warehousing, analytics
- **GPU (P3, G4)**: Machine learning, rendering, video encoding

**P: T3 vs T3a - qual escolher?**
R: 
- **T3**: Processadores Intel, performance ligeiramente melhor para single-thread
- **T3a**: Processadores AMD, ~10% mais barato, performance comparável
- Para maioria dos casos, T3a é melhor custo-benefício

**P: O que é CPU Credits em instâncias T?**
R: Instâncias T usam burstable performance. Acumulam CPU credits quando abaixo do baseline. Gastam credits quando precisam de burst. Quando credits acabam, ficam no baseline. Use unlimited mode para pagar por burst extra.

**P: Graviton (ARM) vs Intel/AMD?**
R:
- **Graviton**: Até 40% melhor price/performance, melhor eficiência energética
- **Limitação**: Precisa de aplicações ARM-compatíveis
- **Uso**: Container workloads, APIs, microservices
- **Evite**: Legacy apps que só funcionam em x86

### Modelos de Compra

**P: Quando usar Spot Instances?**
R: Use para workloads:
- Tolerantes a interrupção
- Flexíveis em horário
- Stateless ou com checkpointing
- Batch processing, big data, CI/CD
- Economia de até 90%

Não use para:
- Databases críticos
- Aplicações stateful sem checkpoint
- Workloads que não toleram interrupção

**P: Reserved Instances vs Savings Plans?**
R:
- **Reserved Instances**: Desconto para instância específica, região, OS. Menos flexível, maior desconto
- **Savings Plans**: Mais flexível, comprometimento em $/hora, pode mudar tipo/região. Recomendado para maioria dos casos

**P: Como maximizar economia com Spot?**
R:
1. Use Spot Fleet com múltiplos tipos de instância
2. Implemente graceful shutdown (pega notificação 2 min antes)
3. Checkpointing para retomar trabalho
4. Use Spot Instance Advisor para escolher pools estáveis
5. Mix de On-Demand + Spot para baseline + burst

### Storage e EBS

**P: Diferença entre Instance Store e EBS?**
R:
- **Instance Store**: Temporário, muito rápido, perde dados em stop/terminate, free
- **EBS**: Persistente, pode anexar/desanexar, snapshots, cobrado por GB

Use Instance Store para: cache, buffers, temporary data
Use EBS para: databases, persistent storage, root volumes

**P: gp2 vs gp3 - qual escolher?**
R: **gp3** na maioria dos casos:
- 20% mais barato
- Performance baseline independente de tamanho (3000 IOPS, 125 MB/s)
- Pode aumentar IOPS/throughput independentemente
- gp2 só se tem volume muito pequeno com baixa necessidade de IOPS

**P: Quando usar io2 ao invés de gp3?**
R: Use io2/io2 Block Express quando:
- Precisa de >16.000 IOPS (gp3 máximo)
- Precisa de >1.000 MB/s throughput
- Latência sub-millisecond crítica
- Databases de alta performance (Oracle, SQL Server)
- Custo: ~5x mais caro que gp3

**P: EBS Snapshots são incrementais?**
R: Sim! Apenas blocos modificados são copiados. Primeiro snapshot é full, subsequentes são delta. Mas pode restaurar de qualquer snapshot (não precisa da chain). Custo apenas pelo storage único.

### Networking

**P: Quantos ENIs (Elastic Network Interfaces) posso ter?**
R: Depende do tipo de instância. Mínimo 2, máximo 15. Útil para:
- Múltiplos IPs privados
- Múltiplos Security Groups
- Management network separado
- Failover (desanexar e anexar em outra instância)

**P: Enhanced Networking - vale a pena?**
R: Sim! É grátis e disponível na maioria das instâncias modernas:
- SR-IOV para maior PPS (packets per second)
- Menor latência
- Menor jitter
- Até 100 Gbps em instâncias maiores
- Habilitado por padrão em instâncias novas

**P: Placement Groups - quando usar?**
R:
- **Cluster**: Para HPC, low-latency apps (mesma AZ, mesmo rack)
- **Spread**: Para alta disponibilidade (instâncias em hardware separado)
- **Partition**: Para distributed workloads (HDFS, Cassandra)

### Auto Scaling

**P: Diferença entre Launch Configuration e Launch Template?**
R: **Use Launch Template** (recomendado):
- Versionamento
- Suporta Spot + On-Demand mix
- Suporta placement groups
- Mais recente e com mais features
- Launch Configuration é legacy

**P: Como configurar Auto Scaling eficientemente?**
R:
1. Use target tracking scaling (ex: manter CPU em 70%)
2. Configure cooldown periods adequados
3. Use step scaling para mudanças rápidas
4. Monitore métricas personalizadas se necessário
5. Teste scale-in policies (não deletar instâncias importantes)
6. Use lifecycle hooks para graceful shutdown

**P: Scheduled Scaling vs Dynamic Scaling?**
R:
- **Scheduled**: Para padrões previsíveis (ex: horário comercial)
- **Dynamic**: Para demanda imprevisível, baseado em métricas
- **Melhor**: Usar ambos - scheduled para baseline, dynamic para ajustes

### Segurança

**P: Como proteger minhas instâncias EC2?**
R:
1. Use IAM roles (não credenciais hardcoded)
2. Security Groups restritivos (principle of least privilege)
3. Patch management regular (Systems Manager Patch Manager)
4. Encryption de EBS volumes
5. Use Systems Manager Session Manager ao invés de SSH público
6. Habilite CloudTrail e CloudWatch
7. Use AWS Inspector para vulnerability scanning
8. Implemente VPC Flow Logs
9. Não exponha RDP/SSH para 0.0.0.0/0

**P: Como fazer key rotation para EC2?**
R: Não há rotação automática de key pairs. Opções:
1. Use Systems Manager Session Manager (sem chaves)
2. Configure chaves adicionais no authorized_keys
3. Crie nova AMI com nova chave
4. Use ferramentas de config management (Ansible, Chef)

### Performance e Otimização

**P: Como otimizar custos EC2?**
R:
1. Use AWS Compute Optimizer para rightsizing
2. Reserved Instances/Savings Plans para workloads previsíveis
3. Spot Instances para workloads flexíveis
4. Stop instâncias fora do horário (dev/test)
5. Use Auto Scaling para escalar com demanda
6. Graviton instances quando possível
7. Monitore utilização com Cost Explorer

**P: Como melhorar performance de disco?**
R:
1. Use gp3 ou io2 com IOPS adequado
2. Use instance types com EBS-optimized
3. RAID 0 para aumentar throughput (múltiplos volumes)
4. Use EBS volume com tamanho adequado (gp2 IOPS cresce com tamanho)
5. Pre-warm volumes de snapshots antes de uso produtivo
6. Use Instance Store para temporary data
7. Monitore CloudWatch metrics (VolumeReadOps, etc)

## Exemplos Avançados

### Exemplo 1: Auto Scaling com Application Load Balancer

```yaml
# CloudFormation Template
AWSTemplateFormatVersion: '2010-09-09'
Description: EC2 Auto Scaling com ALB

Parameters:
  LatestAmiId:
    Type: AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  # Launch Template
  LaunchTemplate:
    Type: AWS::EC2::LaunchTemplate
    Properties:
      LaunchTemplateName: WebAppLaunchTemplate
      VersionDescription: v1
      LaunchTemplateData:
        ImageId: !Ref LatestAmiId
        InstanceType: t3.micro
        IamInstanceProfile:
          Name: !Ref InstanceProfile
        SecurityGroupIds:
          - !Ref InstanceSecurityGroup
        UserData:
          Fn::Base64: !Sub |
            #!/bin/bash
            yum update -y
            yum install -y httpd
            systemctl start httpd
            systemctl enable httpd
            
            # Simple web app
            cat > /var/www/html/index.html <<EOF
            <!DOCTYPE html>
            <html>
            <head><title>EC2 Auto Scaling Demo</title></head>
            <body>
              <h1>Instance: $(ec2-metadata --instance-id | cut -d " " -f 2)</h1>
              <h2>Availability Zone: $(ec2-metadata --availability-zone | cut -d " " -f 2)</h2>
            </body>
            </html>
            EOF
            
            # Health check endpoint
            cat > /var/www/html/health <<EOF
            OK
            EOF
        TagSpecifications:
          - ResourceType: instance
            Tags:
              - Key: Name
                Value: WebApp-AutoScaling
              - Key: Environment
                Value: Production

  # Auto Scaling Group
  AutoScalingGroup:
    Type: AWS::AutoScaling::AutoScalingGroup
    Properties:
      LaunchTemplate:
        LaunchTemplateId: !Ref LaunchTemplate
        Version: !GetAtt LaunchTemplate.LatestVersionNumber
      MinSize: 2
      MaxSize: 10
      DesiredCapacity: 2
      HealthCheckType: ELB
      HealthCheckGracePeriod: 300
      VPCZoneIdentifier:
        - !Ref PrivateSubnet1
        - !Ref PrivateSubnet2
      TargetGroupARNs:
        - !Ref TargetGroup
      Tags:
        - Key: Name
          Value: WebApp-ASG-Instance
          PropagateAtLaunch: true

  # Target Tracking Scaling Policy
  ScalingPolicy:
    Type: AWS::AutoScaling::ScalingPolicy
    Properties:
      AutoScalingGroupName: !Ref AutoScalingGroup
      PolicyType: TargetTrackingScaling
      TargetTrackingConfiguration:
        PredefinedMetricSpecification:
          PredefinedMetricType: ASGAverageCPUUtilization
        TargetValue: 70.0

  # Application Load Balancer
  LoadBalancer:
    Type: AWS::ElasticLoadBalancingV2::LoadBalancer
    Properties:
      Name: WebApp-ALB
      Type: application
      Scheme: internet-facing
      SecurityGroups:
        - !Ref ALBSecurityGroup
      Subnets:
        - !Ref PublicSubnet1
        - !Ref PublicSubnet2

  # Target Group
  TargetGroup:
    Type: AWS::ElasticLoadBalancingV2::TargetGroup
    Properties:
      Name: WebApp-TG
      Port: 80
      Protocol: HTTP
      VpcId: !Ref VPC
      HealthCheckEnabled: true
      HealthCheckPath: /health
      HealthCheckProtocol: HTTP
      HealthCheckIntervalSeconds: 30
      HealthCheckTimeoutSeconds: 5
      HealthyThresholdCount: 2
      UnhealthyThresholdCount: 3

  # Listener
  Listener:
    Type: AWS::ElasticLoadBalancingV2::Listener
    Properties:
      LoadBalancerArn: !Ref LoadBalancer
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref TargetGroup

  # Security Groups
  ALBSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP from internet
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0

  InstanceSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Allow HTTP from ALB
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !Ref ALBSecurityGroup

  # IAM Role for Instances
  InstanceRole:
    Type: AWS::IAM::Role
    Properties:
      AssumeRolePolicyDocument:
        Version: '2012-10-17'
        Statement:
          - Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action: sts:AssumeRole
      ManagedPolicyArns:
        - arn:aws:iam::aws:policy/CloudWatchAgentServerPolicy
        - arn:aws:iam::aws:policy/AmazonSSMManagedInstanceCore

  InstanceProfile:
    Type: AWS::IAM::InstanceProfile
    Properties:
      Roles:
        - !Ref InstanceRole

Outputs:
  LoadBalancerDNS:
    Description: DNS do Load Balancer
    Value: !GetAtt LoadBalancer.DNSName
```

### Exemplo 2: Script Python para Gerenciamento de Instâncias

```python
import boto3
from datetime import datetime, timedelta
import time

class EC2Manager:
    def __init__(self, region='us-east-1'):
        self.ec2 = boto3.client('ec2', region_name=region)
        self.resource = boto3.resource('ec2', region_name=region)
        
    def start_instances_by_tag(self, tag_key, tag_value):
        """Inicia instâncias baseado em tag"""
        instances = self.ec2.describe_instances(
            Filters=[
                {'Name': f'tag:{tag_key}', 'Values': [tag_value]},
                {'Name': 'instance-state-name', 'Values': ['stopped']}
            ]
        )
        
        instance_ids = []
        for reservation in instances['Reservations']:
            for instance in reservation['Instances']:
                instance_ids.append(instance['InstanceId'])
        
        if instance_ids:
            print(f"Iniciando {len(instance_ids)} instâncias...")
            self.ec2.start_instances(InstanceIds=instance_ids)
            return instance_ids
        return []
    
    def stop_instances_by_tag(self, tag_key, tag_value):
        """Para instâncias baseado em tag"""
        instances = self.ec2.describe_instances(
            Filters=[
                {'Name': f'tag:{tag_key}', 'Values': [tag_value]},
                {'Name': 'instance-state-name', 'Values': ['running']}
            ]
        )
        
        instance_ids = []
        for reservation in instances['Reservations']:
            for instance in reservation['Instances']:
                instance_ids.append(instance['InstanceId'])
        
        if instance_ids:
            print(f"Parando {len(instance_ids)} instâncias...")
            self.ec2.stop_instances(InstanceIds=instance_ids)
            return instance_ids
        return []
    
    def schedule_instances(self, tag_key, tag_value, start_hour=9, stop_hour=18):
        """Agenda start/stop de instâncias (horário comercial)"""
        current_hour = datetime.now().hour
        
        if current_hour == start_hour:
            self.start_instances_by_tag(tag_key, tag_value)
        elif current_hour == stop_hour:
            self.stop_instances_by_tag(tag_key, tag_value)
    
    def create_ami_backup(self, instance_id, retention_days=7):
        """Cria AMI backup de instância"""
        timestamp = datetime.now().strftime('%Y%m%d-%H%M%S')
        ami_name = f"backup-{instance_id}-{timestamp}"
        
        print(f"Criando AMI: {ami_name}")
        response = self.ec2.create_image(
            InstanceId=instance_id,
            Name=ami_name,
            Description=f'Automated backup from {instance_id}',
            NoReboot=True
        )
        
        ami_id = response['ImageId']
        
        # Tag AMI com data de expiração
        expiration_date = datetime.now() + timedelta(days=retention_days)
        self.ec2.create_tags(
            Resources=[ami_id],
            Tags=[
                {'Key': 'CreatedBy', 'Value': 'AutomatedBackup'},
                {'Key': 'ExpirationDate', 'Value': expiration_date.strftime('%Y-%m-%d')}
            ]
        )
        
        return ami_id
    
    def cleanup_old_amis(self):
        """Remove AMIs expirados"""
        images = self.ec2.describe_images(Owners=['self'])
        
        deleted = 0
        for image in images['Images']:
            tags = {tag['Key']: tag['Value'] for tag in image.get('Tags', [])}
            
            if 'ExpirationDate' in tags:
                expiration = datetime.strptime(tags['ExpirationDate'], '%Y-%m-%d')
                if expiration < datetime.now():
                    print(f"Deletando AMI expirado: {image['ImageId']}")
                    
                    # Deregister AMI
                    self.ec2.deregister_image(ImageId=image['ImageId'])
                    
                    # Delete associated snapshots
                    for device in image.get('BlockDeviceMappings', []):
                        if 'Ebs' in device:
                            snapshot_id = device['Ebs']['SnapshotId']
                            print(f"Deletando snapshot: {snapshot_id}")
                            self.ec2.delete_snapshot(SnapshotId=snapshot_id)
                    
                    deleted += 1
        
        print(f"Total de AMIs deletados: {deleted}")
        return deleted
    
    def get_instance_cost_estimate(self, instance_type, hours_per_month=730):
        """Estima custo mensal de instância (simplificado)"""
        # Nota: Usar AWS Price List API para valores reais
        pricing_map = {
            't3.micro': 0.0104,
            't3.small': 0.0208,
            't3.medium': 0.0416,
            't3.large': 0.0832,
            'm5.large': 0.096,
            'm5.xlarge': 0.192,
            'c5.large': 0.085,
            'r5.large': 0.126
        }
        
        hourly_cost = pricing_map.get(instance_type, 0)
        monthly_cost = hourly_cost * hours_per_month
        
        return {
            'instance_type': instance_type,
            'hourly_cost': hourly_cost,
            'monthly_cost_on_demand': monthly_cost,
            'monthly_cost_reserved_1yr': monthly_cost * 0.60,  # ~40% discount
            'monthly_cost_reserved_3yr': monthly_cost * 0.40   # ~60% discount
        }
    
    def rightsizing_recommendation(self, instance_id, days=14):
        """Analisa utilização e sugere rightsizing"""
        cloudwatch = boto3.client('cloudwatch')
        
        end_time = datetime.now()
        start_time = end_time - timedelta(days=days)
        
        # Get CPU utilization
        cpu_stats = cloudwatch.get_metric_statistics(
            Namespace='AWS/EC2',
            MetricName='CPUUtilization',
            Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}],
            StartTime=start_time,
            EndTime=end_time,
            Period=3600,  # 1 hora
            Statistics=['Average', 'Maximum']
        )
        
        if not cpu_stats['Datapoints']:
            return {'recommendation': 'Insufficient data'}
        
        avg_cpu = sum(p['Average'] for p in cpu_stats['Datapoints']) / len(cpu_stats['Datapoints'])
        max_cpu = max(p['Maximum'] for p in cpu_stats['Datapoints'])
        
        instance = self.resource.Instance(instance_id)
        current_type = instance.instance_type
        
        recommendation = {
            'instance_id': instance_id,
            'current_type': current_type,
            'avg_cpu': round(avg_cpu, 2),
            'max_cpu': round(max_cpu, 2),
            'days_analyzed': days
        }
        
        if avg_cpu < 20 and max_cpu < 40:
            recommendation['action'] = 'DOWNSIZE'
            recommendation['reason'] = 'CPU utilization muito baixo'
        elif avg_cpu > 70 or max_cpu > 90:
            recommendation['action'] = 'UPSIZE'
            recommendation['reason'] = 'CPU utilization muito alto'
        else:
            recommendation['action'] = 'NO_CHANGE'
            recommendation['reason'] = 'Utilization dentro do esperado'
        
        return recommendation

# Uso
manager = EC2Manager(region='us-east-1')

# Schedule instances (adicionar ao cron)
manager.schedule_instances('Environment', 'Development', start_hour=9, stop_hour=18)

# Backup automation
ami_id = manager.create_ami_backup('i-1234567890abcdef0', retention_days=7)
manager.cleanup_old_amis()

# Cost analysis
cost = manager.get_instance_cost_estimate('t3.medium')
print(f"Custo mensal On-Demand: ${cost['monthly_cost_on_demand']:.2f}")
print(f"Custo mensal Reserved 1yr: ${cost['monthly_cost_reserved_1yr']:.2f}")

# Rightsizing
recommendation = manager.rightsizing_recommendation('i-1234567890abcdef0', days=14)
print(f"Recomendação: {recommendation['action']}")
```

## Recursos de Aprendizado

- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [EC2 Pricing](https://aws.amazon.com/ec2/pricing/)
- [AWS Hands-on Tutorials](https://aws.amazon.com/getting-started/hands-on/)
- [EC2 Best Practices](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-best-practices.html)
- [AWS re:Invent EC2 Sessions](https://www.youtube.com/results?search_query=reinvent+ec2)
- [AWS Compute Blog](https://aws.amazon.com/blogs/compute/)

## Comandos Úteis

```bash
# Listar instâncias
aws ec2 describe-instances

# Listar instâncias running com query
aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query "Reservations[*].Instances[*].[InstanceId,InstanceType,State.Name,Tags[?Key=='Name'].Value|[0]]" \
  --output table

# Iniciar instância
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Parar instância
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Terminar instância
aws ec2 terminate-instances --instance-ids i-1234567890abcdef0

# Criar snapshot
aws ec2 create-snapshot \
  --volume-id vol-1234567890abcdef0 \
  --description "Backup $(date +%Y-%m-%d)"

# Criar AMI
aws ec2 create-image \
  --instance-id i-1234567890abcdef0 \
  --name "My AMI $(date +%Y%m%d)" \
  --no-reboot

# Modificar instance type (requires stop first)
aws ec2 modify-instance-attribute \
  --instance-id i-1234567890abcdef0 \
  --instance-type "{\"Value\": \"t3.large\"}"

# Get console output (troubleshooting boot issues)
aws ec2 get-console-output --instance-id i-1234567890abcdef0

# List AMIs owned by you
aws ec2 describe-images --owners self

# Get latest Amazon Linux 2 AMI
aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-gp2" \
  --query "sort_by(Images, &CreationDate)[-1].[ImageId,Name]" \
  --output table
```
