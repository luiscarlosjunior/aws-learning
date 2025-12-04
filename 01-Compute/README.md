# Compute Services

## Visão Geral
Os serviços de computação da AWS fornecem capacidade de processamento escalável e segura na nuvem. Estes serviços permitem executar aplicações, processar dados e hospedar workloads de diversos tipos.

## Serviços

### EC2 (Elastic Compute Cloud)
Servidores virtuais escaláveis na nuvem. Permite executar aplicações em instâncias virtuais com controle total sobre o sistema operacional.

**Recursos principais:**
- Instâncias sob demanda, reservadas e spot
- Diversos tipos de instância (compute, memory, storage optimized)
- Auto Scaling
- Load Balancing
- Elastic IP

### Lambda
Computação serverless que executa código em resposta a eventos sem necessidade de gerenciar servidores.

**Recursos principais:**
- Execução baseada em eventos
- Pagamento por uso (por tempo de execução)
- Escalabilidade automática
- Suporte a múltiplas linguagens (Python, Node.js, Java, Go, etc.)
- Integração com outros serviços AWS

### ECS (Elastic Container Service)
Serviço de orquestração de containers Docker totalmente gerenciado.

**Recursos principais:**
- Gerenciamento de clusters
- Integração com Docker
- Suporte a Fargate (serverless) e EC2
- Service discovery
- Load balancing

### EKS (Elastic Kubernetes Service)
Kubernetes gerenciado pela AWS para executar aplicações em containers.

**Recursos principais:**
- Kubernetes certificado
- Gerenciamento automatizado
- Integração com serviços AWS
- Multi-AZ para alta disponibilidade

### Elastic Beanstalk
Plataforma como serviço (PaaS) para deploy e gerenciamento de aplicações.

**Recursos principais:**
- Deploy simplificado
- Gerenciamento automático de infraestrutura
- Suporte a múltiplas linguagens e frameworks
- Monitoramento integrado

### Lightsail
Servidores virtuais privados (VPS) simples e de baixo custo para projetos pequenos.

**Recursos principais:**
- Planos previsíveis e de baixo custo
- Interface simplificada
- Templates pré-configurados
- Armazenamento SSD

### Batch
Execução de jobs em batch em qualquer escala.

**Recursos principais:**
- Processamento batch gerenciado
- Escalabilidade automática
- Otimização de custos
- Integração com EC2 e Spot Instances

### Fargate
Motor de computação serverless para containers.

**Recursos principais:**
- Sem gerenciamento de servidores
- Isolamento de segurança
- Compatível com ECS e EKS
- Pagamento por recursos utilizados

## Casos de Uso

1. **Hospedagem de Aplicações Web**: EC2, Elastic Beanstalk, Lightsail
2. **Processamento de Eventos**: Lambda
3. **Microservices**: ECS, EKS, Fargate
4. **Processamento em Lote**: Batch
5. **APIs Serverless**: Lambda + API Gateway

## Melhores Práticas

- Utilize Auto Scaling para otimizar custos e performance
- Implemente health checks e monitoramento
- Use IAM roles para segurança
- Distribua workloads em múltiplas AZs
- Considere Spot Instances para workloads flexíveis
- Implemente backups e disaster recovery

## Recursos de Aprendizado

- [AWS Compute Documentation](https://docs.aws.amazon.com/compute/)
- [AWS Training - Compute](https://aws.amazon.com/training/)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
