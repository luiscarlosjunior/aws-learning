# Serverless

## Visão Geral
Serverless permite construir e executar aplicações sem gerenciar servidores. Você paga apenas pelos recursos consumidos durante a execução.

## Filosofia Serverless

**Princípios:**
- No server management
- Automatic scaling
- Pay-per-use pricing
- Built-in high availability
- Event-driven architecture

**Benefícios:**
- Redução de custos operacionais
- Faster time to market
- Foco no código de negócio
- Escalabilidade automática
- Resiliência built-in

## Serviços Core

### Lambda
Computação serverless que executa código em resposta a eventos.

**Características:**
- Suporta múltiplas linguagens
- Timeout máximo de 15 minutos
- Memória: 128 MB a 10 GB
- Storage temporário: até 10 GB
- Concurrent executions: 1000 (default)

Ver detalhes completos em: [/01-Compute/Lambda/](../01-Compute/Lambda/)

### API Gateway
Criação, publicação e gerenciamento de APIs.

**Tipos:**
- REST APIs
- HTTP APIs (mais baratas e simples)
- WebSocket APIs

Ver detalhes completos em: [/04-Networking-Content-Delivery/API-Gateway/](../04-Networking-Content-Delivery/API-Gateway/)

### DynamoDB
Banco de dados NoSQL serverless.

**Características:**
- Performance consistente em milissegundos
- Fully managed
- Automatic scaling
- Global Tables
- Streams para event-driven

Ver detalhes completos em: [/03-Database/DynamoDB/](../03-Database/DynamoDB/)

### Step Functions
Orquestração de workflows serverless.

**Tipos:**
- Standard Workflows: Long-running, exactly-once
- Express Workflows: High-volume, at-least-once

Ver detalhes completos em: [/08-Application-Integration/Step-Functions/](../08-Application-Integration/Step-Functions/)

### EventBridge
Event bus serverless para integração event-driven.

**Recursos:**
- Event routing
- Schema registry
- Archive e replay
- 100+ SaaS integrations

Ver detalhes completos em: [/08-Application-Integration/EventBridge/](../08-Application-Integration/EventBridge/)

### AppSync
GraphQL API serverless.

**Recursos:**
- Real-time subscriptions
- Offline sync
- Multiple data sources
- Automatic caching

Ver detalhes completos em: [/08-Application-Integration/AppSync/](../08-Application-Integration/AppSync/)

## Serverless Application Patterns

### 1. Web Application (Three-Tier)

```
CloudFront (CDN)
    ↓
S3 (Static content)
    ↓
API Gateway (API)
    ↓
Lambda (Business logic)
    ↓
DynamoDB (Data)
```

**Características:**
- Fully serverless
- Global distribution
- Auto-scaling
- Pay per request

### 2. REST API

```
Client
    ↓
API Gateway (REST)
    ↓
Lambda (Handler)
    ↓
├── DynamoDB (Primary data)
├── S3 (File storage)
└── RDS Proxy → RDS (Relational data)
```

### 3. Event-Driven Processing

```
Event Source (S3, DynamoDB, etc.)
    ↓
EventBridge
    ↓
├── Lambda 1 (Processing)
├── Lambda 2 (Notification)
└── Step Functions (Orchestration)
```

### 4. Real-Time Data Processing

```
Data Producer
    ↓
Kinesis Data Streams
    ↓
Lambda (Stream processing)
    ↓
├── DynamoDB (Store)
├── S3 (Archive)
└── OpenSearch (Analytics)
```

### 5. Scheduled Tasks

```
EventBridge (Cron/Rate)
    ↓
Lambda (Batch job)
    ↓
SQS (Queue for processing)
    ↓
Lambda (Workers)
```

### 6. Async Processing

```
API Gateway
    ↓
Lambda (Receive request)
    ↓
SQS (Queue)
    ↓
Lambda (Process)
    ↓
SNS (Notify)
```

## Serverless Stack Completo

### Frontend
- **S3**: Hosting de static assets
- **CloudFront**: CDN global
- **Amplify**: Framework full-stack

### Backend
- **API Gateway**: API REST/HTTP/WebSocket
- **Lambda**: Business logic
- **AppSync**: GraphQL APIs

### Data
- **DynamoDB**: NoSQL database
- **Aurora Serverless**: Relational database
- **S3**: Object storage

### Integration
- **EventBridge**: Event bus
- **SQS**: Message queuing
- **SNS**: Pub/Sub messaging
- **Step Functions**: Workflow orchestration

### Observability
- **CloudWatch**: Logs, metrics, alarms
- **X-Ray**: Distributed tracing
- **CloudWatch Insights**: Log analytics

## Ferramentas e Frameworks

### AWS SAM (Serverless Application Model)
Framework para build e deploy de aplicações serverless.

**Recursos:**
- Template specification
- Local testing (SAM CLI)
- CI/CD integration
- Built-in best practices

**Template Exemplo:**
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  MyFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: index.handler
      Runtime: python3.9
      Events:
        ApiEvent:
          Type: Api
          Properties:
            Path: /hello
            Method: get
```

### Serverless Framework
Framework multi-cloud para serverless.

**serverless.yml:**
```yaml
service: my-service

provider:
  name: aws
  runtime: python3.9
  region: us-east-1

functions:
  hello:
    handler: handler.hello
    events:
      - http:
          path: hello
          method: get
```

### AWS Amplify
Framework full-stack para web e mobile.

**Recursos:**
- CLI para scaffolding
- Libraries para frontend
- Backend services integration
- Hosting

### AWS CDK (Cloud Development Kit)
Infrastructure as Code usando linguagens de programação.

**Exemplo TypeScript:**
```typescript
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';

const fn = new lambda.Function(this, 'MyFunction', {
  runtime: lambda.Runtime.PYTHON_3_9,
  handler: 'index.handler',
  code: lambda.Code.fromAsset('lambda')
});

new apigateway.LambdaRestApi(this, 'MyApi', {
  handler: fn
});
```

## Melhores Práticas

### Lambda
- Keep functions small and focused
- Use environment variables
- Implement proper error handling
- Optimize cold starts
- Use Lambda Layers para dependencies
- Implement idempotency
- Set appropriate timeout e memory

### API Gateway
- Use API keys e usage plans
- Implement throttling
- Enable caching quando apropriado
- Use custom domains
- Implement request validation
- Version APIs properly

### DynamoDB
- Design for access patterns
- Use partition keys adequadamente
- Implement GSIs e LSIs quando necessário
- Use DynamoDB Streams para event-driven
- Enable point-in-time recovery
- Use on-demand para workloads imprevisíveis

### Step Functions
- Keep state machines simple
- Use service integrations
- Implement error handling
- Use Express workflows para high-volume
- Monitor executions

### Geral
- Design for failure
- Implement distributed tracing (X-Ray)
- Use managed services
- Implement least privilege IAM
- Monitor costs
- Use Infrastructure as Code
- Automate deployments

## Serverless vs Traditional

| Aspecto | Serverless | Traditional |
|---------|-----------|-------------|
| **Server Management** | None | Required |
| **Scaling** | Automatic | Manual/Auto-scaling |
| **Pricing** | Pay-per-use | Always running |
| **Startup** | Cold starts | Always warm |
| **State** | Stateless | Stateful possible |
| **Execution Limits** | Time limits (15min) | No limits |
| **Best For** | Event-driven, variable load | Consistent load, long-running |

## Casos de Uso Ideais para Serverless

1. **APIs REST/GraphQL**: API Gateway + Lambda
2. **Web Applications**: S3 + CloudFront + Lambda + DynamoDB
3. **Data Processing**: S3 events + Lambda + DynamoDB
4. **Scheduled Tasks**: EventBridge + Lambda
5. **Real-time Processing**: Kinesis + Lambda
6. **Chatbots**: Lex + Lambda
7. **IoT Backends**: IoT Core + Lambda + DynamoDB
8. **Image/Video Processing**: S3 + Lambda + MediaConvert
9. **ETL Jobs**: Glue + Lambda + S3
10. **Webhooks**: API Gateway + Lambda

## Cost Optimization

### Estratégias:
1. Right-size Lambda memory
2. Use Compute Savings Plans
3. Enable API caching
4. Use DynamoDB on-demand sabiamente
5. Implement lifecycle policies (S3)
6. Clean up unused resources
7. Use CloudWatch Logs Insights para debug
8. Monitor com Cost Explorer
9. Set up budgets e alerts

## Exemplo Completo: TODO API

### Lambda Handler (Python)
```python
import json
import boto3
import uuid

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('todos')

def handler(event, context):
    http_method = event['httpMethod']
    
    if http_method == 'GET':
        response = table.scan()
        return {
            'statusCode': 200,
            'body': json.dumps(response['Items'])
        }
    
    elif http_method == 'POST':
        body = json.loads(event['body'])
        item = {
            'id': str(uuid.uuid4()),
            'title': body['title'],
            'completed': False
        }
        table.put_item(Item=item)
        return {
            'statusCode': 201,
            'body': json.dumps(item)
        }
```

### SAM Template
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Resources:
  TodoApi:
    Type: AWS::Serverless::Api
    Properties:
      StageName: prod
      
  TodoFunction:
    Type: AWS::Serverless::Function
    Properties:
      Handler: app.handler
      Runtime: python3.9
      Policies:
        - DynamoDBCrudPolicy:
            TableName: !Ref TodoTable
      Environment:
        Variables:
          TABLE_NAME: !Ref TodoTable
      Events:
        GetTodos:
          Type: Api
          Properties:
            RestApiId: !Ref TodoApi
            Path: /todos
            Method: get
        CreateTodo:
          Type: Api
          Properties:
            RestApiId: !Ref TodoApi
            Path: /todos
            Method: post
            
  TodoTable:
    Type: AWS::DynamoDB::Table
    Properties:
      TableName: todos
      BillingMode: PAY_PER_REQUEST
      AttributeDefinitions:
        - AttributeName: id
          AttributeType: S
      KeySchema:
        - AttributeName: id
          KeyType: HASH
```

## Recursos de Aprendizado

- [AWS Serverless](https://aws.amazon.com/serverless/)
- [AWS SAM Documentation](https://docs.aws.amazon.com/serverless-application-model/)
- [Serverless Land](https://serverlessland.com/)
- [Serverless Patterns](https://serverlessland.com/patterns)
- [AWS Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning)
- [Serverless Stack](https://serverless-stack.com/)
- [AWS Serverless Workshops](https://workshops.aws/categories/Serverless)
