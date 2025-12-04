# Developer Tools

## Visão Geral
Os serviços de ferramentas de desenvolvimento da AWS ajudam a automatizar processos de desenvolvimento, build, teste e deploy de aplicações.

## Serviços

### CodeCommit
Repositório Git privado totalmente gerenciado.

**Recursos principais:**
- Repositórios Git privados ilimitados
- Alta disponibilidade e durabilidade
- Integração com IAM
- Code reviews e pull requests
- Triggers e notificações
- Encryption at rest e in transit

### CodeBuild
Serviço de build totalmente gerenciado.

**Recursos principais:**
- Build serverless (sem servidores para gerenciar)
- Ambientes de build pré-configurados
- Custom build environments (Docker)
- Build paralelo
- Integração com CodePipeline
- Pay per minute

**Linguagens Suportadas:**
- Java, Python, Node.js, Ruby, Go, .NET, PHP, Android

### CodeDeploy
Automação de deployment de código.

**Recursos principais:**
- Deploy em EC2, Lambda, ECS, on-premises
- Rolling deployments
- Blue/green deployments
- Rollback automático
- Hooks para customização
- Integration com CI/CD tools

**Deployment Types:**
- In-place
- Blue/Green
- Canary
- Linear

### CodePipeline
Orquestração de CI/CD workflows.

**Recursos principais:**
- Automação completa de release
- Integração com múltiplas ferramentas
- Aprovações manuais
- Parallel execution
- Artifacts management
- Cross-region deployments

**Integrations:**
- Source: CodeCommit, GitHub, S3
- Build: CodeBuild, Jenkins
- Deploy: CodeDeploy, ECS, S3, CloudFormation
- Test: CodeBuild, third-party tools

### Cloud9
IDE baseado em nuvem.

**Recursos principais:**
- Editor de código online
- Terminal integrado
- Debugging
- Pair programming
- AWS CLI pré-instalado
- Acesso direto a serviços AWS

### X-Ray
Análise e debugging de aplicações distribuídas.

**Recursos principais:**
- Distributed tracing
- Service map visualization
- Performance analysis
- Error detection
- Request tracking
- Integration com Lambda, API Gateway, EC2, ECS

**Componentes:**
- Segments: Trabalho de um serviço
- Subsegments: Detalhes granulares
- Traces: Requisição end-to-end
- Service map: Visualização de dependências

### CodeArtifact
Repositório de artefatos totalmente gerenciado.

**Recursos principais:**
- Armazenamento de pacotes
- Proxy para repositórios públicos
- Versionamento
- Access control via IAM
- Suporte a múltiplos formatos

**Formatos Suportados:**
- npm, Maven, PyPI, NuGet

### CodeGuru
Revisão de código e profiling com ML.

**Componentes:**
- **CodeGuru Reviewer**: Code review automatizado
- **CodeGuru Profiler**: Performance profiling

**Recursos principais:**
- Machine learning recommendations
- Best practices detection
- Security vulnerabilities
- Performance bottlenecks
- Memory leaks detection

## CI/CD Pipeline Típico

```
Commit → CodeCommit
         ↓
      CodePipeline (orchestration)
         ↓
    ┌────┴────┐
    ↓         ↓
CodeBuild  CodeGuru (review)
(build)       ↓
    ↓    Approval
    ↓         ↓
CodeDeploy ←──┘
(deploy)
    ↓
X-Ray (monitoring)
```

## Casos de Uso

1. **CI/CD Completo**: CodePipeline + CodeBuild + CodeDeploy
2. **Microservices Deployment**: ECS/EKS + CodePipeline
3. **Serverless CI/CD**: Lambda + SAM + CodePipeline
4. **Infrastructure as Code**: CloudFormation + CodePipeline
5. **Multi-region Deployment**: CodePipeline cross-region

## Melhores Práticas

### CodeCommit
- Use branch protection
- Implement code reviews
- Use triggers para automação
- Enable notifications

### CodeBuild
- Use buildspec.yml no repositório
- Cache dependencies
- Use artifacts para sharing
- Implement build badges
- Separate build stages

### CodeDeploy
- Test em staging antes de production
- Use deployment configuration adequada
- Implement health checks
- Configure rollback automático
- Use lifecycle hooks

### CodePipeline
- Keep stages simple e focados
- Use manual approvals para production
- Implement automated testing
- Monitor pipeline execution
- Use cross-account deployments quando necessário

### Geral
- Automate everything
- Test automation code
- Keep secrets em Secrets Manager
- Use IAM roles (não credentials)
- Monitor e log everything

## Exemplo: buildspec.yml

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Build started on `date`
      - docker build -t $IMAGE_REPO_NAME:$IMAGE_TAG .
      - docker tag $IMAGE_REPO_NAME:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG
  post_build:
    commands:
      - echo Build completed on `date`
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$IMAGE_REPO_NAME:$IMAGE_TAG

artifacts:
  files:
    - '**/*'
```

## Recursos de Aprendizado

- [AWS Developer Tools](https://aws.amazon.com/products/developer-tools/)
- [CodePipeline Tutorial](https://docs.aws.amazon.com/codepipeline/latest/userguide/tutorials.html)
- [DevOps on AWS](https://aws.amazon.com/devops/)
- [AWS DevOps Blog](https://aws.amazon.com/blogs/devops/)
