# Application Integration

## Visão Geral
Os serviços de integração de aplicações da AWS permitem desacoplar componentes de aplicações e construir arquiteturas escaláveis, resilientes e orientadas a eventos.

## Serviços

### SQS (Simple Queue Service)
Serviço de filas de mensagens totalmente gerenciado.

**Recursos principais:**
- Filas ilimitadas
- Mensagens de até 256 KB
- Retention de até 14 dias
- At-least-once delivery
- FIFO e Standard queues
- Dead Letter Queues (DLQ)
- Long polling

**Tipos de Fila:**
- **Standard**: Throughput ilimitado, at-least-once, best-effort ordering
- **FIFO**: Ordenação garantida, exactly-once, até 3000 msg/s

**Casos de Uso:**
- Desacoplamento de microservices
- Buffering de workloads
- Batch processing
- Event-driven architectures

### SNS (Simple Notification Service)
Pub/Sub messaging totalmente gerenciado.

**Recursos principais:**
- Publish to multiple subscribers
- Message filtering
- Mobile push notifications
- SMS, Email, HTTP/S endpoints
- Integration com SQS, Lambda
- Message fanout

**Tipos de Subscription:**
- SQS
- Lambda
- HTTP/HTTPS
- Email
- SMS
- Mobile Push (APNs, FCM, etc.)

**Casos de Uso:**
- Fan-out pattern
- Application alerts
- Mobile notifications
- System notifications

### EventBridge
Event bus serverless para integração de aplicações.

**Recursos principais:**
- Event routing baseado em regras
- Schema registry
- Archive e replay
- Integration com 100+ SaaS applications
- Custom event buses
- Event patterns para filtering

**Sources:**
- AWS services
- Custom applications
- SaaS partners (Zendesk, DataDog, etc.)

**Targets:**
- Lambda
- Step Functions
- SQS, SNS
- API destinations
- 20+ AWS services

### Step Functions
Orquestração de workflows visuais.

**Recursos principais:**
- State machines (JSON/ASL)
- Visual workflow designer
- Error handling e retry
- Parallel execution
- Human approval steps
- Integration com 200+ AWS services

**State Types:**
- Task
- Choice
- Parallel
- Wait
- Success/Fail
- Map (iterate)

**Casos de Uso:**
- Microservices orchestration
- ETL workflows
- Order processing
- Machine learning pipelines
- Human approval workflows

### AppSync
GraphQL API totalmente gerenciado.

**Recursos principais:**
- Real-time data sync
- Offline support
- GraphQL subscriptions
- Multiple data sources
- Built-in authentication
- Caching

**Data Sources:**
- DynamoDB
- Lambda
- HTTP endpoints
- RDS
- OpenSearch

**Casos de Uso:**
- Mobile/web applications
- Real-time dashboards
- Collaborative apps
- IoT applications

### Amazon MQ
Message broker gerenciado (Apache ActiveMQ, RabbitMQ).

**Recursos principais:**
- Industry-standard protocols
- High availability
- Automatic failover
- Durability
- Message persistence

**Use quando:**
- Migração de on-premises
- Necessita de protocolos específicos (AMQP, MQTT, STOMP)
- Aplicações legacy

### Simple Workflow (SWF)
Coordenação de tarefas distribuídas (legacy).

**Nota**: Considere usar Step Functions para novos projetos.

## Padrões de Integração

### 1. Point-to-Point (SQS)
```
Producer → SQS Queue → Consumer
```

### 2. Pub/Sub (SNS)
```
Publisher → SNS Topic → Multiple Subscribers
```

### 3. Fan-Out (SNS + SQS)
```
Publisher → SNS Topic → Multiple SQS Queues → Multiple Consumers
```

### 4. Event-Driven (EventBridge)
```
Event Source → EventBridge → Rules → Multiple Targets
```

### 5. Workflow Orchestration (Step Functions)
```
Trigger → Step Functions → Coordinated Tasks
```

## Comparação de Serviços

| Serviço | Padrão | Uso | Durabilidade |
|---------|--------|-----|--------------|
| SQS | Queue | Point-to-point | Até 14 dias |
| SNS | Pub/Sub | Fan-out | Não persiste |
| EventBridge | Event Bus | Event-driven | 24h archive |
| Step Functions | Workflow | Orchestration | - |
| AppSync | GraphQL | API/Real-time | - |
| MQ | Message Broker | Legacy/Standards | Persistente |

## Casos de Uso

1. **Microservices Communication**: SQS, SNS, EventBridge
2. **Order Processing**: Step Functions + SQS
3. **Real-time Notifications**: SNS, AppSync
4. **Event-Driven Architecture**: EventBridge
5. **Legacy Migration**: Amazon MQ
6. **Mobile/Web Apps**: AppSync
7. **Batch Processing**: SQS + Lambda

## Melhores Práticas

### SQS
- Use Dead Letter Queues
- Implement idempotent processing
- Configure visibility timeout adequadamente
- Use batch operations
- Monitor queue depth
- Implement backpressure

### SNS
- Use message filtering
- Implement retry policies
- Monitor delivery failures
- Use DLQ para failed deliveries
- Encrypt sensitive data

### EventBridge
- Use specific event patterns
- Implement schema registry
- Enable archiving para replay
- Monitor rule execution
- Use input transformation

### Step Functions
- Keep state machines simple
- Use Express workflows para high-volume
- Implement error handling
- Use service integrations quando possível
- Monitor execution history

### AppSync
- Use caching
- Implement pagination
- Optimize resolver logic
- Use batching
- Monitor performance metrics

### Geral
- Design for failure
- Implement retry logic
- Use exponential backoff
- Monitor metrics
- Implement circuit breakers
- Test error scenarios

## Exemplo: SQS com Lambda

### Enviar Mensagem
```python
import boto3
import json

sqs = boto3.client('sqs')
queue_url = 'https://sqs.us-east-1.amazonaws.com/123456789012/MyQueue'

response = sqs.send_message(
    QueueUrl=queue_url,
    MessageBody=json.dumps({
        'orderId': '12345',
        'items': ['item1', 'item2']
    })
)
```

### Processar Mensagem (Lambda)
```python
def lambda_handler(event, context):
    for record in event['Records']:
        message = json.loads(record['body'])
        order_id = message['orderId']
        
        # Processar pedido
        print(f"Processing order: {order_id}")
        
    return {
        'statusCode': 200,
        'body': 'Messages processed'
    }
```

## Exemplo: EventBridge Rule

```json
{
  "source": ["aws.ec2"],
  "detail-type": ["EC2 Instance State-change Notification"],
  "detail": {
    "state": ["terminated"]
  }
}
```

## Exemplo: Step Functions State Machine

```json
{
  "Comment": "Order Processing Workflow",
  "StartAt": "ValidateOrder",
  "States": {
    "ValidateOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ValidateOrder",
      "Next": "ProcessPayment"
    },
    "ProcessPayment": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ProcessPayment",
      "Next": "ShipOrder"
    },
    "ShipOrder": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:us-east-1:123456789012:function:ShipOrder",
      "End": true
    }
  }
}
```

## Recursos de Aprendizado

- [AWS Integration Services](https://aws.amazon.com/products/application-integration/)
- [SQS Best Practices](https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/sqs-best-practices.html)
- [EventBridge User Guide](https://docs.aws.amazon.com/eventbridge/)
- [Step Functions Workshop](https://catalog.workshops.aws/stepfunctions/)
- [AppSync Developer Guide](https://docs.aws.amazon.com/appsync/)
