# Containers

## Visão Geral
Os serviços de containers da AWS permitem executar aplicações containerizadas com diferentes níveis de gerenciamento e controle.

## Serviços

### ECS (Elastic Container Service)
Orquestrador de containers Docker totalmente gerenciado pela AWS.

**Recursos principais:**
- Gerenciamento simplificado
- Integração profunda com AWS services
- Suporte a Fargate e EC2
- Service discovery
- Load balancing (ALB/NLB)
- Auto scaling
- Task definitions

**Launch Types:**
- **Fargate**: Serverless, sem gerenciamento de instâncias
- **EC2**: Controle total sobre instâncias

**Componentes:**
- **Cluster**: Agrupamento lógico de recursos
- **Task Definition**: Blueprint da aplicação
- **Service**: Mantém número desejado de tasks rodando
- **Task**: Instância de um container em execução

### EKS (Elastic Kubernetes Service)
Kubernetes gerenciado pela AWS.

**Recursos principais:**
- Kubernetes certificado
- Control plane gerenciado
- Multi-AZ para alta disponibilidade
- Suporte a Fargate e EC2
- Add-ons gerenciados
- Integração com AWS services

**Componentes:**
- **Control Plane**: Gerenciado pela AWS
- **Worker Nodes**: EC2 ou Fargate
- **Node Groups**: Grupos de EC2 instances
- **Fargate Profiles**: Configuração serverless

**Add-ons:**
- VPC CNI
- CoreDNS
- kube-proxy
- AWS Load Balancer Controller
- EBS CSI Driver

### ECR (Elastic Container Registry)
Registro de imagens Docker totalmente gerenciado.

**Recursos principais:**
- Private e public registries
- Image scanning (vulnerabilidades)
- Lifecycle policies
- Cross-region replication
- OCI-compliant
- Integration com ECS/EKS

**Recursos de Segurança:**
- Encryption at rest
- IAM-based access control
- Image scanning (Clair)
- Immutable tags
- Resource-based policies

### Fargate
Motor de computação serverless para containers.

**Recursos principais:**
- Sem gerenciamento de servidores
- Pagamento por recursos usados (vCPU e memória)
- Isolamento de segurança por task
- Compatível com ECS e EKS
- Automatic scaling
- Spot capacity support

**Casos de Uso:**
- Microservices
- Batch jobs
- CI/CD workloads
- Web applications

### App Runner
Forma mais simples de executar containers em produção.

**Recursos principais:**
- Fully managed
- Deploy from source ou container
- Automatic scaling
- Load balancing
- HTTPS out-of-the-box
- Custom domains

**Casos de Uso:**
- Web applications
- APIs
- Microservices simples
- Prototipagem rápida

## Comparação ECS vs EKS

| Aspecto | ECS | EKS |
|---------|-----|-----|
| **Orquestrador** | Proprietário AWS | Kubernetes |
| **Complexidade** | Mais simples | Mais complexo |
| **Portabilidade** | AWS-only | Multi-cloud |
| **Integração AWS** | Nativa e profunda | Via add-ons |
| **Comunidade** | Menor | Enorme (K8s) |
| **Custo** | Sem custo adicional | $0.10/hora por cluster |
| **Use Case** | AWS-native apps | K8s workloads, portability |

## Comparação Launch Types

| Aspecto | Fargate | EC2 |
|---------|---------|-----|
| **Gerenciamento** | Zero | Manual |
| **Controle** | Limitado | Total |
| **Pricing** | vCPU + Memory | Instance hours |
| **Startup** | Mais lento | Mais rápido |
| **Use Cases** | Microservices, batch | Otimização custo, controle |

## Arquitetura de Referência - Microservices ECS

```
Internet → ALB
           ↓
    ECS Service (Fargate)
    ├── Task 1 (Container)
    ├── Task 2 (Container)
    └── Task 3 (Container)
           ↓
    Service Discovery (Cloud Map)
           ↓
    Backend Services
    ├── Database (RDS)
    ├── Cache (ElastiCache)
    └── Queue (SQS)
```

## Arquitetura de Referência - EKS

```
kubectl → EKS Control Plane (Managed)
                ↓
         Worker Nodes
         ├── EC2 Node Group
         │   └── Pods
         └── Fargate Profile
             └── Pods
                ↓
         AWS Services
         ├── ALB (via LB Controller)
         ├── EBS (via CSI Driver)
         └── ECR (images)
```

## Casos de Uso

1. **Microservices**: ECS/EKS + ALB + Service Discovery
2. **Batch Processing**: ECS/EKS com Fargate
3. **CI/CD**: CodePipeline + ECS/EKS
4. **Web Applications**: App Runner ou ECS com ALB
5. **Legacy Modernization**: Containerize + ECS/EKS
6. **ML Inference**: ECS/EKS + GPU instances

## Melhores Práticas

### ECS
- Use Fargate para simplificar operações
- Implement health checks
- Use task roles para permissions
- Configure auto scaling
- Use ECR para imagens
- Implement blue/green deployments
- Monitor com CloudWatch Container Insights

### EKS
- Use managed node groups
- Implement RBAC properly
- Use Fargate para critical workloads
- Configure cluster autoscaler
- Use Helm para package management
- Implement GitOps (Flux/ArgoCD)
- Monitor com Container Insights

### ECR
- Enable image scanning
- Use lifecycle policies
- Implement immutable tags
- Configure replication se necessário
- Tag images adequadamente
- Clean up unused images

### Geral
- Use multi-stage Docker builds
- Keep images small
- Don't run as root
- Scan for vulnerabilities
- Use secrets management (Secrets Manager)
- Implement logging strategy
- Resource limits e requests

## Exemplo: Task Definition ECS

```json
{
  "family": "my-app",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "ENV",
          "value": "production"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/my-app",
          "awslogs-region": "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

## Exemplo: Kubernetes Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: default
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
      - name: app
        image: 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        env:
        - name: ENV
          value: "production"
---
apiVersion: v1
kind: Service
metadata:
  name: my-app-service
spec:
  type: LoadBalancer
  selector:
    app: my-app
  ports:
  - port: 80
    targetPort: 80
```

## Exemplo: Dockerfile Multi-Stage

```dockerfile
# Build stage
FROM node:16 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Runtime stage
FROM node:16-alpine
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
EXPOSE 3000
CMD ["node", "dist/index.js"]
```

## CI/CD para Containers

### Pipeline Típico:
```
Code Commit
    ↓
CodeBuild
├── Build Docker image
├── Run tests
├── Scan for vulnerabilities
└── Push to ECR
    ↓
CodeDeploy
└── Deploy to ECS/EKS
    ├── Blue/Green deployment
    └── Rollback if needed
```

## Comandos Úteis

### ECS
```bash
# List clusters
aws ecs list-clusters

# Describe service
aws ecs describe-services --cluster my-cluster --services my-service

# Update service
aws ecs update-service --cluster my-cluster --service my-service --force-new-deployment
```

### EKS
```bash
# Configure kubectl
aws eks update-kubeconfig --name my-cluster --region us-east-1

# Get nodes
kubectl get nodes

# Get pods
kubectl get pods -A

# Apply manifest
kubectl apply -f deployment.yaml
```

### ECR
```bash
# Login
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 123456789012.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t my-app .
docker tag my-app:latest 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

## Recursos de Aprendizado

- [ECS Documentation](https://docs.aws.amazon.com/ecs/)
- [EKS Documentation](https://docs.aws.amazon.com/eks/)
- [EKS Best Practices Guide](https://aws.github.io/aws-eks-best-practices/)
- [ECS Workshop](https://ecsworkshop.com/)
- [EKS Workshop](https://www.eksworkshop.com/)
- [Containers on AWS](https://aws.amazon.com/containers/)
