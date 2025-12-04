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

### Não consegue SSH
1. Verify Security Group permite port 22
2. Check NACL
3. Verify key pair correto
4. Check se instance tem public IP (se em public subnet)
5. Verify instance está running

### VPC Peering não funciona
1. Check CIDRs não sobrepõem
2. Verify route tables em ambas VPCs
3. Check Security Groups
4. Verify peering connection accepted

## Recursos de Aprendizado

- [VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [VPC Networking Components](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Networking.html)
- [VPC Security Best Practices](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-best-practices.html)
- [CIDR Calculator](https://www.ipaddressguide.com/cidr)
