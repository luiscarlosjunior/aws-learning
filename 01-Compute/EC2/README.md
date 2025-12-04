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

## Recursos de Aprendizado

- [EC2 User Guide](https://docs.aws.amazon.com/ec2/)
- [EC2 Instance Types](https://aws.amazon.com/ec2/instance-types/)
- [EC2 Pricing](https://aws.amazon.com/ec2/pricing/)
- [AWS Hands-on Tutorials](https://aws.amazon.com/getting-started/hands-on/)

## Comandos Úteis

```bash
# Listar instâncias
aws ec2 describe-instances

# Iniciar instância
aws ec2 start-instances --instance-ids i-1234567890abcdef0

# Parar instância
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# Criar snapshot
aws ec2 create-snapshot --volume-id vol-1234567890abcdef0

# Criar AMI
aws ec2 create-image --instance-id i-1234567890abcdef0 --name "My AMI"
```
