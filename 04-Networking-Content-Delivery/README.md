# Networking & Content Delivery Services

## Visão Geral
Os serviços de rede e entrega de conteúdo da AWS fornecem conectividade segura, escalável e de alto desempenho para suas aplicações.

## Serviços

### VPC (Virtual Private Cloud)
Rede virtual isolada logicamente na AWS Cloud.

**Componentes principais:**
- **Subnets**: Segmentação de rede (públicas e privadas)
- **Route Tables**: Controle de roteamento
- **Internet Gateway**: Acesso à internet
- **NAT Gateway/Instance**: Internet para subnets privadas
- **Security Groups**: Firewall stateful em nível de instância
- **Network ACLs**: Firewall stateless em nível de subnet
- **VPC Peering**: Conexão entre VPCs
- **VPC Endpoints**: Acesso privado a serviços AWS

**Recursos:**
- CIDR blocks customizáveis
- IPv4 e IPv6 support
- DNS resolution
- DHCP options
- Flow Logs para monitoramento

### CloudFront
Content Delivery Network (CDN) global da AWS.

**Recursos principais:**
- Edge locations em todo o mundo
- Integração com S3, EC2, ELB
- Lambda@Edge para customização
- SSL/TLS support
- Real-time metrics
- DDoS protection (AWS Shield)
- Cache customizável

**Casos de Uso:**
- Distribuição de conteúdo estático
- Streaming de vídeo
- Aceleração de APIs
- Aplicações web dinâmicas

### Route 53
Serviço de DNS escalável e altamente disponível.

**Recursos principais:**
- Domain registration
- DNS routing
- Health checking
- Traffic management

**Routing Policies:**
- **Simple**: Roteamento básico
- **Weighted**: Distribuição por peso
- **Latency**: Menor latência
- **Failover**: Alta disponibilidade
- **Geolocation**: Baseado em localização
- **Geoproximity**: Proximidade geográfica
- **Multi-value**: Múltiplos valores com health checks

### API Gateway
Criação, publicação e gerenciamento de APIs.

**Tipos:**
- **REST APIs**: RESTful APIs completas
- **HTTP APIs**: APIs mais simples e baratas
- **WebSocket APIs**: Comunicação bidirecional em tempo real

**Recursos principais:**
- Throttling e rate limiting
- Autenticação e autorização
- Request/response transformation
- API versioning
- Caching
- Monitoring e logging
- Integration com Lambda, HTTP endpoints, AWS services

### Direct Connect
Conexão de rede dedicada entre on-premises e AWS.

**Recursos principais:**
- Bandwidth dedicado (1 Gbps, 10 Gbps, 100 Gbps)
- Redução de custos de transferência
- Latência consistente
- Private connectivity (não passa pela internet)
- VPN over Direct Connect

**Casos de Uso:**
- Hybrid cloud
- Transferência de grandes volumes de dados
- Aplicações de baixa latência
- Compliance e segurança

### Elastic Load Balancing (ELB)
Distribuição automática de tráfego entre múltiplos targets.

**Tipos:**
- **Application Load Balancer (ALB)**: Layer 7 (HTTP/HTTPS)
  - Path-based routing
  - Host-based routing
  - Lambda targets
  
- **Network Load Balancer (NLB)**: Layer 4 (TCP/UDP)
  - Ultra baixa latência
  - Milhões de requests por segundo
  - Static IP support
  
- **Gateway Load Balancer (GWLB)**: Layer 3
  - Appliances virtuais (firewalls, IDS/IPS)
  
- **Classic Load Balancer**: Legacy (deprecated)

### Transit Gateway
Hub central para conectar VPCs e redes on-premises.

**Recursos principais:**
- Simplifica arquiteturas de rede
- Suporta milhares de VPCs
- Roteamento inter-regional
- Direct Connect e VPN integration
- Multicast support

**Benefícios:**
- Reduz complexidade de peering
- Centraliza gerenciamento
- Escalabilidade

### Global Accelerator
Melhora disponibilidade e performance de aplicações globais.

**Recursos principais:**
- IPs anycast estáticos
- Otimização de rota via rede global AWS
- Health checking e failover automático
- Traffic dials para blue/green deployment

**Diferença do CloudFront:**
- Global Accelerator: Aceleração de TCP/UDP
- CloudFront: Cache de conteúdo HTTP/HTTPS

## Casos de Uso

1. **Aplicações Web Multi-Tier**: VPC, Subnets, ALB
2. **Distribuição Global de Conteúdo**: CloudFront, Route 53
3. **APIs Serverless**: API Gateway, Lambda
4. **Hybrid Cloud**: Direct Connect, VPN, Transit Gateway
5. **Alta Disponibilidade**: ELB, Route 53 health checks
6. **Microsserviços**: API Gateway, ALB, Service Mesh

## Melhores Práticas

### VPC
- Use múltiplas AZs para alta disponibilidade
- Segregue subnets públicas e privadas
- Implemente defense in depth (Security Groups + NACLs)
- Use VPC Flow Logs para troubleshooting
- Planeje CIDR blocks cuidadosamente

### CloudFront
- Use Origin Shield para reduzir carga no origin
- Configure TTLs apropriados
- Use signed URLs/cookies para conteúdo privado
- Habilite compression
- Monitore cache hit ratio

### Route 53
- Use health checks para failover automático
- Implemente alias records quando possível
- Configure TTLs apropriados
- Use Traffic Flow para roteamento complexo

### API Gateway
- Implemente throttling para proteção
- Use API keys e usage plans
- Cache responses quando apropriado
- Habilite CloudWatch Logs
- Version APIs adequadamente

### ELB
- Use target groups para flexibilidade
- Configure health checks adequadamente
- Enable access logs para auditoria
- Use SSL/TLS offloading
- Distribua em múltiplas AZs

### Segurança
- Use private subnets para recursos sensíveis
- Implemente least privilege em Security Groups
- Use VPC Endpoints para serviços AWS
- Habilite AWS Shield e WAF
- Encrypt data in transit (TLS)

## Arquitetura de Referência

### Three-Tier Web Application
```
Internet → CloudFront → ALB → Web Tier (Public Subnet)
                         ↓
                    App Tier (Private Subnet)
                         ↓
                    Database (Private Subnet)
                         ↓
                    NAT Gateway (para updates)
```

### Hybrid Architecture
```
On-Premises ← Direct Connect → Transit Gateway → VPCs
                                      ↓
                                Route 53 (DNS)
                                      ↓
                                  Internet
```

## Recursos de Aprendizado

- [AWS Networking Fundamentals](https://aws.amazon.com/products/networking/)
- [VPC User Guide](https://docs.aws.amazon.com/vpc/)
- [CloudFront Developer Guide](https://docs.aws.amazon.com/cloudfront/)
- [Route 53 Developer Guide](https://docs.aws.amazon.com/route53/)
- [AWS Networking Workshops](https://networking.workshop.aws/)
