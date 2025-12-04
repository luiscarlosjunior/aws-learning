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

## Recursos de Aprendizado

- [Lambda Developer Guide](https://docs.aws.amazon.com/lambda/)
- [Serverless Application Model (SAM)](https://aws.amazon.com/serverless/sam/)
- [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)
- [Serverless Framework](https://www.serverless.com/)
- [AWS Serverless Workshops](https://workshops.aws/)
