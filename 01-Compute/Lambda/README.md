# AWS Lambda

## O que é Lambda?

AWS Lambda é um serviço de computação serverless que executa código em resposta a eventos e automaticamente gerencia os recursos de computação. Você paga apenas pelo tempo de computação que consome.

## Conceitos Principais

### Função Lambda
Código que é executado em resposta a eventos. Pode ser escrito em várias linguagens.

**Linguagens Suportadas:**
- Python
- Node.js
- Java
- C# (.NET)
- Go
- Ruby
- PowerShell
- Custom Runtime (via Lambda Layers)

### Eventos (Event Sources)
Triggers que invocam funções Lambda:
- **S3**: Upload de objetos
- **DynamoDB**: Mudanças em streams
- **API Gateway**: Requisições HTTP
- **CloudWatch Events/EventBridge**: Eventos agendados ou baseados em regras
- **SNS/SQS**: Mensagens
- **Kinesis**: Streams de dados
- **Application Load Balancer**: Requisições HTTP
- **Alexa Skills**: Voice commands
- **CloudFront Lambda@Edge**: Requisições CDN

### Execution Environment
Ambiente isolado onde a função é executada.

**Recursos:**
- Memória: 128 MB a 10 GB (incrementos de 1 MB)
- CPU: Proporcional à memória alocada
- Storage temporário (/tmp): Até 10 GB
- Timeout: Máximo 15 minutos
- Concurrent Executions: 1000 por região (pode ser aumentado)

### Cold Start vs Warm Start
- **Cold Start**: Primeira execução ou após período de inatividade
- **Warm Start**: Reutilização de ambiente existente

## Recursos

### Lambda Layers
Pacotes de código e dependências compartilhados entre múltiplas funções.

**Casos de Uso:**
- Bibliotecas compartilhadas
- Dependências customizadas
- Configurações comuns

### Environment Variables
Variáveis de configuração disponíveis durante execução.

### VPC Integration
Execute Lambda dentro de uma VPC para acessar recursos privados (RDS, ElastiCache, etc.).

### Function URLs
URLs HTTPS dedicadas para invocar funções Lambda diretamente.

### Aliases e Versions
- **Versions**: Snapshots imutáveis de função
- **Aliases**: Ponteiros para versões específicas

### Destinations
Configure destinos para resultados de execução (sucesso/falha).

## Pricing

**Componentes de Custo:**
1. **Requests**: Primeiros 1M grátis/mês, depois $0.20 por 1M
2. **Duration**: Primeiros 400.000 GB-seconds grátis/mês
   - Calculado como: memória alocada × tempo de execução
3. **Data Transfer**: Cobrado como outros serviços AWS

**Exemplo:**
- 1M invocações
- 512 MB memória
- 100ms duração média
- Custo ≈ $0.20 (requests) + $0.83 (duration) = ~$1.03

## Casos de Uso

1. **APIs Serverless**: Lambda + API Gateway
2. **Processamento de Dados**: S3 triggers, stream processing
3. **Automação**: Tarefas agendadas, resposta a eventos
4. **Backend para Mobile/Web**: Apps serverless
5. **Real-time File Processing**: Transformação de imagens/vídeos
6. **ETL Jobs**: Extract, Transform, Load
7. **ChatBots e Alexa Skills**
8. **IoT Backend**: Processar dados de dispositivos

## Melhores Práticas

### Performance
- Minimize cold starts:
  - Use Provisioned Concurrency
  - Mantenha código leve
  - Reutilize conexões (databases, HTTP clients)
- Otimize tamanho do deployment package
- Use variáveis de ambiente para configuração
- Implemente retry logic e idempotência

### Segurança
- Use IAM roles com princípio de menor privilégio
- Encrypt environment variables
- Use Secrets Manager para credenciais
- Habilite AWS X-Ray para tracing
- Valide inputs
- Use VPC quando necessário

### Código
- Separe handler da lógica de negócio
- Mantenha funções focadas (single responsibility)
- Use Lambda Layers para dependências
- Implemente logging estruturado
- Trate erros adequadamente

### Custo
- Otimize memória (mais memória = mais CPU = potencialmente mais rápido)
- Monitore unused functions
- Use timeouts apropriados
- Considere Compute Savings Plans

## Exemplo Básico

### Python Handler

```python
import json

def lambda_handler(event, context):
    """
    Handler básico que processa eventos
    """
    print(f"Evento recebido: {json.dumps(event)}")
    
    # Processar evento
    body = json.loads(event.get('body', '{}'))
    name = body.get('name', 'World')
    
    # Resposta
    response = {
        'statusCode': 200,
        'headers': {
            'Content-Type': 'application/json'
        },
        'body': json.dumps({
            'message': f'Hello, {name}!'
        })
    }
    
    return response
```

### Node.js Handler

```javascript
exports.handler = async (event) => {
    console.log('Event:', JSON.stringify(event, null, 2));
    
    const body = JSON.parse(event.body || '{}');
    const name = body.name || 'World';
    
    const response = {
        statusCode: 200,
        headers: {
            'Content-Type': 'application/json'
        },
        body: JSON.stringify({
            message: `Hello, ${name}!`
        })
    };
    
    return response;
};
```

### S3 Event Handler

```python
import boto3
import os

s3 = boto3.client('s3')

def lambda_handler(event, context):
    """
    Processa uploads no S3
    """
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        print(f"Processando arquivo: {key} do bucket: {bucket}")
        
        # Processar arquivo
        # response = s3.get_object(Bucket=bucket, Key=key)
        # content = response['Body'].read()
        
    return {
        'statusCode': 200,
        'body': 'Processamento concluído'
    }
```

## Monitoramento

### CloudWatch Metrics
- Invocations
- Duration
- Errors
- Throttles
- Concurrent Executions
- Dead Letter Queue Errors

### CloudWatch Logs
Cada função Lambda tem um log group automático.

### AWS X-Ray
Tracing distribuído para debugging e análise de performance.

## Comandos Úteis

```bash
# Criar função
aws lambda create-function \
    --function-name my-function \
    --runtime python3.9 \
    --role arn:aws:iam::123456789012:role/lambda-role \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip

# Invocar função
aws lambda invoke \
    --function-name my-function \
    --payload '{"key":"value"}' \
    response.json

# Atualizar código
aws lambda update-function-code \
    --function-name my-function \
    --zip-file fileb://function.zip

# Listar funções
aws lambda list-functions

# Ver logs
aws logs tail /aws/lambda/my-function --follow
```

## Troubleshooting Comum

### Cold Start Lento
**Problema**: Primeira invocação muito lenta
**Soluções**:
- Use Provisioned Concurrency para manter funções warm
- Minimize deployment package size
- Remova dependências desnecessárias
- Use Lambda Layers para dependências pesadas
- Considere linguagens com startup mais rápido (Python, Node.js vs Java, .NET)
- Mantenha código fora do handler quando possível

### Timeout Errors
**Problema**: Função excede tempo limite
**Soluções**:
- Aumentar timeout (máximo 15 minutos)
- Otimizar código
- Fazer operações assíncronas quando possível
- Quebrar função grande em múltiplas funções menores
- Usar Step Functions para workflows longos
- Verificar se não há wait desnecessário

### Out of Memory Errors
**Problema**: Função fica sem memória
**Soluções**:
- Aumentar memória alocada
- Otimizar uso de memória no código
- Processar dados em chunks ao invés de tudo de uma vez
- Liberar recursos explicitamente
- Usar /tmp com cuidado (limite de 10 GB)
- Considerar streaming para grandes arquivos

### Throttling Errors
**Problema**: TooManyRequestsException
**Soluções**:
- Aumentar reserved concurrency
- Solicitar aumento de concurrent execution limit
- Implementar retry com exponential backoff
- Usar SQS como buffer
- Distribuir carga ao longo do tempo
- Verificar se não há recursive invocation acidental

### VPC Timeout Issues
**Problema**: Lambda em VPC não consegue acessar recursos
**Soluções**:
- Verificar security groups e NACLs
- Adicionar NAT Gateway para acesso à internet
- Usar VPC endpoints para serviços AWS
- Verificar route tables
- Ensure ENI capacity na subnet
- Considere usar Hyperplane ENIs (automático em funções novas)

## Perguntas e Respostas (Q&A)

### Conceitos Fundamentais

**P: Quando devo usar Lambda vs EC2 vs Fargate?**
R:
- **Lambda**: Event-driven, curta duração (<15 min), stateless, pagamento por execução
- **EC2**: Controle total, long-running, stateful, aplicações legacy
- **Fargate**: Containers, longa duração, melhor para microservices consistentes

Use Lambda quando:
- Workload é event-driven e intermitente
- Execução < 15 minutos
- Quer zero gerenciamento de infraestrutura
- Custo é proporcional ao uso real

**P: Lambda é realmente serverless?**
R: Sim, no sentido que você não gerencia servidores. Mas há servidores executando seu código - você apenas não os vê ou gerencia. AWS cuida de provisioning, patching, scaling, alta disponibilidade.

**P: Como Lambda escalona?**
R: Automaticamente e instantaneamente. Cada invocação pode executar em parallel. Limite padrão: 1000 concurrent executions por região (pode ser aumentado). Se exceder, novas invocações são throttled.

**P: O que acontece quando Lambda atinge o limite de concorrência?**
R: Invocações são throttled (TooManyRequestsException). Para invocações síncronas, cliente recebe erro 429. Para invocações assíncronas, Lambda retenta automaticamente e depois envia para DLQ se configurado.

### Cold Start vs Warm Start

**P: O que causa cold start?**
R: Cold start ocorre quando:
- Primeira invocação da função
- Função não foi invocada por ~15 minutos
- Código ou configuração mudou
- Concorrência aumenta (novos execution environments)

**P: Quanto tempo dura um cold start?**
R: Depende da linguagem e tamanho:
- Python/Node.js: 100-300ms
- Java/.NET: 500-1000ms (ou mais)
- Go: 200-400ms
- Com VPC: Adicione 1-2 segundos (mas melhorou muito com Hyperplane ENIs)

**P: Como minimizar impacto de cold starts?**
R:
1. Use Provisioned Concurrency (custo extra)
2. Mantenha deployment package pequeno
3. Minimize dependências
4. Use Lambda SnapStart (Java 11+)
5. Escolha linguagens com startup rápido
6. Implemente lazy loading
7. Reutilize conexões (databases, HTTP clients) fora do handler
8. Use CloudWatch Events para "warm" função periodicamente (último recurso)

**P: Provisioned Concurrency vale a pena?**
R: Depende:
- **Sim** se: Latência previsível é crítica, custo de cold starts é alto, tráfego consistente
- **Não** se: Tráfego muito variável, cold starts toleráveis, custo é limitante
- Custo: ~$0.015/GB-hour (além do custo normal de execução)

### Performance e Otimização

**P: Mais memória = melhor performance?**
R: Sim! Memória e CPU são proporcionais. Dobrar memória dobra CPU. Muitas vezes, mais memória = execução mais rápida = custo total menor. Use AWS Lambda Power Tuning para encontrar sweet spot.

**P: Como otimizar tamanho do deployment package?**
R:
1. Remova dependências não utilizadas
2. Use Lambda Layers para código compartilhado
3. Minimize devDependencies (Node.js)
4. Use tree shaking e minification
5. Exclua arquivos desnecessários (.git, tests, docs)
6. Considere usar Lambda Container Images (até 10 GB)

**P: Devo usar /tmp directory?**
R: Sim, mas com cuidado:
- Até 10 GB disponível (configurável desde 512 MB)
- Persiste durante warm starts
- Não compartilhado entre executions
- Use para cache temporário, downloads
- Limpe arquivos grandes para liberar espaço

**P: Como reutilizar conexões de database?**
R: Declare conexão fora do handler. Ou use RDS Proxy para pooling gerenciado.

### Custos

**P: Como é calculado o custo do Lambda?**
R: Dois componentes:
1. **Requests**: $0.20 por 1M requests (primeiro 1M grátis/mês)
2. **Duration**: $0.0000166667 por GB-second (400.000 GB-s grátis/mês)

Exemplo:
- 1M invocações, 512 MB, 200ms média
- Requests: $0.20
- Duration: 1M × 0.5 GB × 0.2s × $0.0000166667 = $1.67
- **Total: $1.87/mês**

**P: Como reduzir custos do Lambda?**
R:
1. Otimize duração (código mais rápido)
2. Use memória adequada
3. Evite polling desnecessário
4. Use batching quando possível
5. Compute Savings Plans (até 17% desconto)
6. Delete funções não utilizadas

**P: Lambda é sempre mais barato que EC2?**
R: Não! Depende do uso:
- **Lambda mais barato**: Tráfego intermitente, baixo volume
- **EC2 mais barato**: Alto volume constante (>40% utilização 24/7)
- **Breakeven**: ~40-50% utilização constante

### Segurança

**P: Como Lambda acessa outros serviços AWS?**
R: Via IAM Execution Role. Role define quais serviços Lambda pode acessar e quais ações pode realizar. Nunca hardcode credentials!

**P: Como proteger secrets em Lambda?**
R:
1. **AWS Secrets Manager**: Melhor para rotation automática
2. **Systems Manager Parameter Store**: Grátis para parâmetros standard
3. **Environment Variables**: Apenas encrypted
4. **KMS**: Para encryption adicional

**P: Lambda em VPC é seguro?**
R: Não necessariamente mais seguro, mas oferece acesso a recursos privados (RDS, ElastiCache), network isolation, e security groups para controle de tráfego. Trade-off: Adiciona complexidade.

### Integração e Padrões

**P: Como orquestrar múltiplas Lambdas?**
R: Use **AWS Step Functions** para visual workflow, error handling, retry, parallel execution, e state management. Alternativa: EventBridge para event-driven choreography.

**P: Como processar arquivos grandes (>GB) com Lambda?**
R: Use S3 Select, streaming em chunks, split de arquivos, Lambda chain, ou considere ECS/Fargate para processamento muito grande.

**P: Como implementar API REST com Lambda?**
R: API Gateway + Lambda (mais comum), Lambda Function URLs (simples), ou ALB + Lambda (integração com ALB existente).

## Recursos de Aprendizado

- [Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/)
- [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)
- [Serverless Framework](https://www.serverless.com/)
- [AWS Serverless Workshops](https://workshops.aws/)
- [AWS Lambda Powertools](https://awslabs.github.io/aws-lambda-powertools-python/)
- [Serverless Land](https://serverlessland.com/)
- [AWS re:Invent Lambda Sessions](https://www.youtube.com/results?search_query=reinvent+lambda)
- [Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
