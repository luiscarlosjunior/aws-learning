# Amazon VPC (Virtual Private Cloud)

## O que é VPC?

Amazon VPC permite provisionar uma seção logicamente isolada da AWS Cloud onde você pode lançar recursos AWS em uma rede virtual que você define.

## Conceitos Principais

### VPC
Rede virtual dedicada à sua conta AWS.

**Características:**
- CIDR block (ex: 10.0.0.0/16)
- Isolamento lógico
- Controle completo sobre networking
- Suporte IPv4 e IPv6

### Subnets
Divisão do CIDR block da VPC.

**Tipos:**
- **Public Subnet**: Tem rota para Internet Gateway
- **Private Subnet**: Sem acesso direto à internet

**Boas Práticas:**
- Distribua subnets em múltiplas AZs
- Use subnets públicas para recursos expostos (ALB, NAT Gateway)
- Use subnets privadas para aplicações e databases

### Internet Gateway (IGW)
Permite comunicação entre VPC e internet.

**Características:**
- Um por VPC
- Horizontally scaled, redundante, alta disponibilidade
- Sem bandwidth constraints

### NAT Gateway/Instance
Permite instâncias em subnets privadas acessarem internet (outbound).

**NAT Gateway (recomendado):**
- Managed service
- Alta disponibilidade em uma AZ
- Bandwidth até 45 Gbps
- Deploy um por AZ para HA

**NAT Instance:**
- EC2 instance
- Gerenciamento manual
- Menor custo
- Pode ser bottleneck

### Route Tables
Controla roteamento de tráfego de/para subnets.

**Exemplo:**
```
Destination      Target
10.0.0.0/16     local
0.0.0.0/0       igw-xxxxx  (Internet Gateway)
```

### Security Groups
Firewall stateful em nível de instância/ENI.

**Características:**
- Allow rules apenas (deny é implícito)
- Stateful (retorno é automático)
- Avalia todas as rules antes de decidir
- Pode referenciar outros security groups

**Exemplo:**
```
Inbound:
- Type: HTTP, Protocol: TCP, Port: 80, Source: 0.0.0.0/0
- Type: SSH, Protocol: TCP, Port: 22, Source: My IP

Outbound:
- Type: All traffic, Protocol: All, Port: All, Destination: 0.0.0.0/0
```

### Network ACLs (NACLs)
Firewall stateless em nível de subnet.

**Características:**
- Allow e deny rules
- Stateless (retorno deve ser explícito)
- Rules processadas em ordem numérica
- Default NACL permite todo tráfego

**Quando Usar:**
- Block IPs específicos
- Defense in depth
- Subnet-level control

### VPC Peering
Conexão entre duas VPCs.

**Características:**
- Não transitivo
- Sem overlapping CIDR blocks
- Cross-account e cross-region suportado
- Traffic stays on AWS network

### VPC Endpoints
Acesso privado a serviços AWS sem internet.

**Tipos:**
- **Interface Endpoints**: ENI com IP privado (powered by PrivateLink)
  - Suporta muitos serviços (CloudWatch, SNS, etc.)
  - Custo: por hora + per GB
  
- **Gateway Endpoints**: Target em route table
  - S3 e DynamoDB apenas
  - Sem custo adicional

### VPC Flow Logs
Capture informações sobre tráfego IP.

**Níveis:**
- VPC
- Subnet
- ENI (Elastic Network Interface)

**Destinos:**
- CloudWatch Logs
- S3
- Kinesis Data Firehose

**Formato:**
```
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

## Arquitetura Típica

### Multi-Tier Application

```
VPC (10.0.0.0/16)
│
├── Public Subnet 1 (10.0.1.0/24) - AZ1
│   ├── Internet Gateway
│   ├── NAT Gateway
│   └── Bastion Host
│
├── Public Subnet 2 (10.0.2.0/24) - AZ2
│   └── NAT Gateway
│
├── Private Subnet 1 (10.0.11.0/24) - AZ1
│   └── Application Servers
│
├── Private Subnet 2 (10.0.12.0/24) - AZ2
│   └── Application Servers
│
├── Private Subnet 3 (10.0.21.0/24) - AZ1
│   └── Database
│
└── Private Subnet 4 (10.0.22.0/24) - AZ2
    └── Database
```

### Fluxo de Tráfego

**Internet → Application:**
```
Internet → IGW → ALB (Public Subnet) → EC2 (Private Subnet)
```

**Application → Internet:**
```
EC2 (Private Subnet) → NAT Gateway (Public Subnet) → IGW → Internet
```

## CIDR Planning

### RFC 1918 (Private IP Ranges)
- 10.0.0.0/8 (10.0.0.0 - 10.255.255.255)
- 172.16.0.0/12 (172.16.0.0 - 172.31.255.255)
- 192.168.0.0/16 (192.168.0.0 - 192.168.255.255)

### Exemplo de Planejamento

**VPC:** 10.0.0.0/16 (65,536 IPs)
- **Public Subnets:** 10.0.0.0/20 (4,096 IPs)
  - AZ1: 10.0.0.0/24 (256 IPs)
  - AZ2: 10.0.1.0/24 (256 IPs)
  - AZ3: 10.0.2.0/24 (256 IPs)
  
- **Private App Subnets:** 10.0.16.0/20 (4,096 IPs)
  - AZ1: 10.0.16.0/24
  - AZ2: 10.0.17.0/24
  - AZ3: 10.0.18.0/24
  
- **Private DB Subnets:** 10.0.32.0/20 (4,096 IPs)
  - AZ1: 10.0.32.0/24
  - AZ2: 10.0.33.0/24
  - AZ3: 10.0.34.0/24

**AWS reserva 5 IPs por subnet:**
- .0: Network address
- .1: VPC router
- .2: DNS server
- .3: Future use
- .255: Broadcast (não usado mas reservado)

## Conectividade

### Site-to-Site VPN
Conexão IPsec entre on-premises e VPC.

**Componentes:**
- Customer Gateway (on-premises)
- Virtual Private Gateway (AWS)
- VPN Connection (dois túneis IPsec)

### Direct Connect
Conexão dedicada entre on-premises e AWS.

**Benefícios:**
- Bandwidth consistente
- Menor latência
- Redução de custos de transferência
- Conexão privada

### Transit Gateway
Hub central para conectar VPCs e redes on-premises.

**Benefícios:**
- Simplifica arquitetura
- Scaling (milhares de VPCs)
- Roteamento centralizado
- Multi-account support

## Segurança

### Melhores Práticas

1. **Least Privilege**: Security Groups restritivos
2. **Defense in Depth**: Security Groups + NACLs
3. **Private Subnets**: Recursos internos em subnets privadas
4. **Bastion Hosts**: Acesso controlado a recursos privados
5. **VPC Flow Logs**: Monitoramento de tráfego
6. **Encryption**: VPN e Direct Connect encrypted
7. **VPC Endpoints**: Acesso privado a serviços AWS

### Bastion Host / Jump Box

```
Internet → Bastion (Public Subnet) → Private Resources
```

**Alternativa Moderna:**
- AWS Systems Manager Session Manager (sem bastion necessário)

## Casos de Uso

1. **Three-Tier Web Application**: Web, App, Database tiers
2. **Hybrid Cloud**: Direct Connect ou VPN
3. **Multi-Account**: VPC Peering ou Transit Gateway
4. **Microservices**: Múltiplas subnets, service discovery
5. **DMZ**: Public subnet com stricter security

## Melhores Práticas

### Planejamento
- Plan CIDR carefully (avoid overlaps)
- Leave room for growth
- Use consistent naming
- Document architecture

### Alta Disponibilidade
- Deploy em múltiplas AZs
- Use NAT Gateway por AZ
- Multi-AZ databases
- ALB com targets em múltiplas AZs

### Segurança
- Use private subnets quando possível
- Implement least privilege
- Enable Flow Logs
- Use VPC Endpoints
- Regular security audits

### Custo
- Use Gateway Endpoints (free) para S3/DynamoDB
- Consider NAT Instance para low-traffic
- Delete unused resources
- Monitor data transfer costs

## Exemplo: Terraform VPC

```hcl
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true
  
  tags = {
    Name = "main-vpc"
  }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]
  
  map_public_ip_on_launch = true
  
  tags = {
    Name = "public-subnet-${count.index + 1}"
  }
}

resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id
  
  tags = {
    Name = "main-igw"
  }
}

resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id
  
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }
  
  tags = {
    Name = "public-rt"
  }
}
```

## Comandos Úteis

```bash
# List VPCs
aws ec2 describe-vpcs

# Create VPC
aws ec2 create-vpc --cidr-block 10.0.0.0/16

# Create Subnet
aws ec2 create-subnet --vpc-id vpc-xxx --cidr-block 10.0.1.0/24 --availability-zone us-east-1a

# Create Internet Gateway
aws ec2 create-internet-gateway

# Attach IGW to VPC
aws ec2 attach-internet-gateway --vpc-id vpc-xxx --internet-gateway-id igw-xxx

# Enable Flow Logs
aws ec2 create-flow-logs \
  --resource-type VPC \
  --resource-ids vpc-xxx \
  --traffic-type ALL \
  --log-destination-type cloud-watch-logs \
  --log-group-name /aws/vpc/flowlogs
```

## Troubleshooting

### Instância não acessa internet
1. Check se está em public subnet
2. Verify Internet Gateway exists e está attached
3. Check route table tem rota para IGW
4. Verify Security Group permite outbound
5. Check NACL permite outbound e inbound return
6. Verify instance tem public IP ou Elastic IP
7. Check se não há blackhole routes

### Não consegue SSH
1. Verify Security Group permite port 22
2. Check NACL permite porta 22 inbound e ephemeral ports (1024-65535) outbound
3. Verify key pair correto
4. Check se instance tem public IP (se em public subnet)
5. Verify instance está running
6. Check route table
7. Considere usar Session Manager como alternativa

### VPC Peering não funciona
1. Check CIDRs não sobrepõem
2. Verify route tables em ambas VPCs
3. Check Security Groups referencing peered VPC
4. Verify peering connection accepted
5. Check NACLs em ambos os lados
6. Verify DNS resolution habilitado se necessário

### Problemas com DNS
1. Verify DNS resolution e DNS hostnames habilitados
2. Check Route 53 Private Hosted Zone associations
3. Verify DHCP Options Set correto
4. Check se VPC Endpoints precisam de private DNS

### NAT Gateway não funciona
1. Check NAT Gateway está em public subnet
2. Verify Elastic IP associado ao NAT Gateway
3. Check route table da private subnet aponta para NAT Gateway
4. Verify Security Groups e NACLs
5. Check limites de conexões concorrentes (55.000 por NAT Gateway)

## Perguntas e Respostas (Q&A)

### Conceitos Fundamentais

**P: Qual CIDR devo usar para minha VPC?**
R: Recomendações:
- **Pequena empresa/dev**: 10.0.0.0/16 (65.536 IPs)
- **Média empresa**: 172.16.0.0/12 (1.048.576 IPs)
- **Grande empresa**: 10.0.0.0/8 (16.777.216 IPs)

Evite conflitos com:
- Redes on-premises
- Outras VPCs que precisará fazer peering
- Redes parceiras

**P: Diferença entre Security Group e NACL?**
R:
| Característica | Security Group | NACL |
|----------------|----------------|------|
| **Nível** | Instance (ENI) | Subnet |
| **Estado** | Stateful | Stateless |
| **Regras** | Apenas Allow | Allow e Deny |
| **Retorno** | Automático | Manual |
| **Ordem** | Todas aplicadas | Por número |
| **Default** | Deny all inbound | Allow all |

Use ambos para defense in depth!

**P: Quantas subnets devo criar?**
R: Arquitetura típica:
- **2 Public subnets**: Uma em cada AZ para ALB, NAT Gateway, Bastion
- **2 Private subnets**: Uma em cada AZ para aplicações
- **2 Database subnets**: Uma em cada AZ para databases isolados

Mínimo: 2 AZs para alta disponibilidade
Ideal: 3 AZs para máxima resiliência

**P: Como calcular tamanhos de subnet?**
R: AWS reserva 5 IPs por subnet:
- .0: Network address
- .1: VPC router
- .2: DNS server
- .3: Reservado para uso futuro
- .255: Network broadcast

Exemplo /24 (256 IPs):
- 256 - 5 = 251 IPs utilizáveis
- /28 (16 IPs) = 11 utilizáveis
- /20 (4.096 IPs) = 4.091 utilizáveis

**P: Posso mudar o CIDR da VPC depois de criada?**
R: Não pode mudar o CIDR primário, mas pode:
- Adicionar CIDRs secundários (até 4 adicionais)
- CIDRs secundários devem ser do mesmo tamanho ou maiores
- Útil quando precisa expandir VPC sem recriar

### Internet Gateway vs NAT

**P: Quando usar Internet Gateway vs NAT Gateway?**
R:
- **Internet Gateway**: Para recursos que PRECISAM ser acessados da internet (public subnets). Comunicação bidirecional.
- **NAT Gateway**: Para recursos que só precisam ACESSAR a internet, mas não ser acessados (private subnets). Comunicação unidirecional (outbound only).

**P: NAT Gateway vs NAT Instance - qual usar?**
R:
**NAT Gateway** (recomendado):
- ✅ Managed, sem manutenção
- ✅ Alta disponibilidade em AZ
- ✅ Até 45 Gbps bandwidth
- ✅ Scaling automático
- ❌ Custo: $0.045/hora + $0.045/GB processado

**NAT Instance**:
- ✅ Mais barato (~$0.01/hora t3.nano)
- ✅ Pode usar como bastion também
- ✅ Pode customizar
- ❌ Você gerencia
- ❌ Single point of failure
- ❌ Bandwidth limitado ao tipo de instância

**P: Preciso de um NAT Gateway por AZ?**
R: Para alta disponibilidade, SIM. Se NAT Gateway falhar ou AZ ficar indisponível, instances na outra AZ ficarão sem internet. Custo extra vale a pena para produção.

### VPC Peering

**P: VPC Peering é transitivo?**
R: Não! Se VPC-A peer com VPC-B, e VPC-B peer com VPC-C, VPC-A NÃO pode comunicar com VPC-C. Você precisa criar peering direto entre A e C.

**P: Quantos VPC Peerings posso ter?**
R: Limite padrão: 125 peerings por VPC (pode aumentar até 125). Para muitas VPCs, considere AWS Transit Gateway.

**P: Posso fazer peering entre regiões?**
R: Sim! Inter-Region VPC Peering suporta:
- Peering entre qualquer região comercial AWS
- Tráfego criptografado
- Sem bandwidth bottleneck
- Latência consistente

Limitações:
- Não suporta IPv6
- Não suporta security group referencing cross-region

**P: VPC Peering vs Transit Gateway vs AWS PrivateLink?**
R:
- **VPC Peering**: Para poucos VPCs, comunicação full mesh
- **Transit Gateway**: Hub central para muitos VPCs (>10), simplifica topologia
- **PrivateLink**: Para expor serviços específicos, sem expor toda VPC

### Endpoints e PrivateLink

**P: Interface Endpoint vs Gateway Endpoint?**
R:
**Gateway Endpoint** (free):
- Apenas S3 e DynamoDB
- Usa route table
- Sem custo adicional
- Não usa IP da subnet

**Interface Endpoint** (~$0.01/hora):
- Qualquer serviço AWS suportado
- Usa ENI na subnet
- Usa PrivateLink
- Cobra por endpoint e por GB

**P: Quando usar VPC Endpoints?**
R: Use quando:
- Quer acesso privado a serviços AWS (sem internet)
- Compliance requer tráfego não sair da AWS network
- Quer economizar em NAT Gateway data transfer
- Precisa de baixa latência

Exemplo: Lambda em VPC acessando S3 via Gateway Endpoint economiza em NAT Gateway.

### Segurança

**P: Como implementar bastion host seguro?**
R: Melhores práticas:
1. Use em public subnet
2. Security Group restrito (apenas seu IP)
3. Use Systems Manager Session Manager (sem SSH)
4. Se usar SSH, use key-based auth + MFA
5. Desabilite password auth
6. Configure audit logging
7. Use Auto Scaling Group (min=0, max=1) para on-demand
8. Considere AWS Client VPN como alternativa

**P: Como proteger databases em VPC?**
R:
1. Sempre em private subnet
2. Security Group permite apenas application tier
3. Use separate database subnet group
4. Encryption at rest (KMS)
5. Encryption in transit (SSL/TLS)
6. VPC Flow Logs habilitado
7. Use RDS Proxy para connection pooling
8. Backups automáticos para subnet diferente

**P: O que são VPC Flow Logs?**
R: Logs de tráfego de rede capturando:
- Source/destination IP e port
- Protocol
- Action (ACCEPT/REJECT)
- Bytes e packets

Útil para:
- Security analysis
- Troubleshooting connectivity
- Compliance
- Traffic analytics

Custo: Storage (S3 ou CloudWatch Logs) + ingest

### Híbrido e Conectividade

**P: VPN vs Direct Connect - qual usar?**
R:
**VPN** (~$0.05/hora):
- ✅ Setup rápido (minutos)
- ✅ Baixo custo
- ✅ Criptografia IPsec
- ❌ Internet-dependent
- ❌ Variável latência/bandwidth
- Use para: Dev/test, disaster recovery, temporary

**Direct Connect** (~$0.30/hora + port fees):
- ✅ Dedicado, consistente performance
- ✅ Baixa latência
- ✅ Até 100 Gbps
- ❌ Setup lento (semanas/meses)
- ❌ Caro
- Use para: Produção, high bandwidth, low latency

**Melhor**: Usar ambos - Direct Connect para produção, VPN como backup.

**P: O que é AWS Transit Gateway?**
R: Hub central que simplifica conectividade:
- Conecta milhares de VPCs
- Conecta on-premises via VPN ou Direct Connect
- Uma rota table central
- Suporta multicast
- Peering entre regions

Use quando:
- >10 VPCs
- Arquitetura hub-and-spoke desejada
- Simplificar gerenciamento de rotas

### Custos

**P: O que é cobrado em VPC?**
R: VPC em si é grátis! Cobranças:
- **NAT Gateway**: $0.045/hora + $0.045/GB
- **VPC Endpoints**: $0.01/hora + $0.01/GB (Interface)
- **VPN**: $0.05/hora por connection
- **Transit Gateway**: $0.05/hora + $0.02/GB
- **Data Transfer**: OUT para internet
- **Elastic IPs**: Se não associado

**P: Como reduzir custos de VPC?**
R:
1. Use Gateway Endpoints (S3, DynamoDB) - free
2. Delete NAT Gateways em dev/test quando não usar
3. Use single NAT Gateway (não HA) para dev/test
4. Use VPC Endpoints ao invés de NAT para serviços AWS
5. Compress data antes de transfer
6. Delete unused Elastic IPs
7. Right-size VPN e Direct Connect

## Exemplos Avançados

### Exemplo 1: VPC Completa com Terraform

```hcl
# Main VPC
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_hostnames = true
  enable_dns_support   = true

  tags = {
    Name = "production-vpc"
  }
}

# Public Subnets
resource "aws_subnet" "public" {
  count                   = 2
  vpc_id                  = aws_vpc.main.id
  cidr_block              = "10.0.${count.index}.0/24"
  availability_zone       = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true

  tags = {
    Name = "public-subnet-${count.index + 1}"
    Tier = "Public"
  }
}

# Private Subnets (Application)
resource "aws_subnet" "private_app" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 10}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-app-subnet-${count.index + 1}"
    Tier = "Private-App"
  }
}

# Private Subnets (Database)
resource "aws_subnet" "private_db" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = "10.0.${count.index + 20}.0/24"
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "private-db-subnet-${count.index + 1}"
    Tier = "Private-DB"
  }
}

# Internet Gateway
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "main-igw"
  }
}

# Elastic IPs for NAT Gateways
resource "aws_eip" "nat" {
  count  = 2
  domain = "vpc"

  tags = {
    Name = "nat-eip-${count.index + 1}"
  }
}

# NAT Gateways (one per AZ for HA)
resource "aws_nat_gateway" "main" {
  count         = 2
  allocation_id = aws_eip.nat[count.index].id
  subnet_id     = aws_subnet.public[count.index].id

  tags = {
    Name = "nat-gateway-${count.index + 1}"
  }

  depends_on = [aws_internet_gateway.main]
}

# Route Table for Public Subnets
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "public-route-table"
  }
}

# Route Tables for Private Subnets (one per AZ)
resource "aws_route_table" "private" {
  count  = 2
  vpc_id = aws_vpc.main.id

  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.main[count.index].id
  }

  tags = {
    Name = "private-route-table-${count.index + 1}"
  }
}

# Route Table Associations
resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

resource "aws_route_table_association" "private_app" {
  count          = 2
  subnet_id      = aws_subnet.private_app[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

resource "aws_route_table_association" "private_db" {
  count          = 2
  subnet_id      = aws_subnet.private_db[count.index].id
  route_table_id = aws_route_table.private[count.index].id
}

# VPC Endpoints
resource "aws_vpc_endpoint" "s3" {
  vpc_id       = aws_vpc.main.id
  service_name = "com.amazonaws.${var.region}.s3"
  
  route_table_ids = concat(
    [aws_route_table.public.id],
    aws_route_table.private[*].id
  )

  tags = {
    Name = "s3-gateway-endpoint"
  }
}

# Security Groups
resource "aws_security_group" "alb" {
  name        = "alb-security-group"
  description = "Security group for Application Load Balancer"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "alb-sg"
  }
}

resource "aws_security_group" "app" {
  name        = "app-security-group"
  description = "Security group for application tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 80
    to_port         = 80
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "app-sg"
  }
}

resource "aws_security_group" "db" {
  name        = "db-security-group"
  description = "Security group for database tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.app.id]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "db-sg"
  }
}

# VPC Flow Logs
resource "aws_flow_log" "main" {
  iam_role_arn    = aws_iam_role.flow_logs.arn
  log_destination = aws_cloudwatch_log_group.flow_logs.arn
  traffic_type    = "ALL"
  vpc_id          = aws_vpc.main.id

  tags = {
    Name = "vpc-flow-logs"
  }
}

resource "aws_cloudwatch_log_group" "flow_logs" {
  name              = "/aws/vpc/flow-logs"
  retention_in_days = 30
}

# Network ACLs (additional security layer)
resource "aws_network_acl" "public" {
  vpc_id     = aws_vpc.main.id
  subnet_ids = aws_subnet.public[*].id

  ingress {
    protocol   = "tcp"
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 80
    to_port    = 80
  }

  ingress {
    protocol   = "tcp"
    rule_no    = 110
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 443
    to_port    = 443
  }

  ingress {
    protocol   = "tcp"
    rule_no    = 120
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 1024
    to_port    = 65535
  }

  egress {
    protocol   = -1
    rule_no    = 100
    action     = "allow"
    cidr_block = "0.0.0.0/0"
    from_port  = 0
    to_port    = 0
  }

  tags = {
    Name = "public-nacl"
  }
}
```

## Recursos de Aprendizado

- [VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [VPC Networking Components](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Networking.html)
- [VPC Security Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
- [CIDR Calculator](https://www.ipaddressguide.com/cidr)
- [AWS VPC Workshop](https://networking.workshop.aws/)
- [VPC Design Patterns](https://aws.amazon.com/blogs/networking-and-content-delivery/)
- [AWS re:Invent VPC Sessions](https://www.youtube.com/results?search_query=reinvent+vpc)
