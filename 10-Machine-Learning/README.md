# Machine Learning

## Visão Geral
Os serviços de Machine Learning da AWS permitem adicionar inteligência artificial às aplicações sem necessidade de expertise profunda em ML.

## Serviços

### SageMaker
Plataforma completa para construir, treinar e implantar modelos de ML.

**Componentes:**
- **SageMaker Studio**: IDE completo para ML
- **SageMaker Notebooks**: Jupyter notebooks gerenciados
- **SageMaker Training**: Treinamento distribuído
- **SageMaker Hosting**: Deploy de modelos
- **SageMaker Pipelines**: MLOps workflows
- **SageMaker Feature Store**: Feature management
- **SageMaker Model Monitor**: Monitoramento de modelos
- **SageMaker Autopilot**: AutoML
- **SageMaker Ground Truth**: Data labeling

**Recursos principais:**
- Built-in algorithms
- Custom algorithms (containers)
- Distributed training
- Hyperparameter tuning
- Model registry
- A/B testing

### Rekognition
Análise de imagens e vídeos com deep learning.

**Recursos principais:**
- Object e scene detection
- Face detection e recognition
- Face comparison
- Celebrity recognition
- Text detection (OCR)
- Content moderation
- PPE detection
- Custom labels

**Casos de Uso:**
- Content moderation
- Face verification
- Video analysis
- Media search
- Safety compliance

### Comprehend
Natural Language Processing (NLP) service.

**Recursos principais:**
- Sentiment analysis
- Entity recognition
- Key phrase extraction
- Language detection
- Topic modeling
- Custom classification
- PII detection

**Casos de Uso:**
- Customer feedback analysis
- Document classification
- Content categorization
- Social media monitoring

### Polly
Text-to-Speech (TTS) service.

**Recursos principais:**
- Neural voices (alta qualidade)
- 60+ languages
- SSML support
- Custom lexicons
- Speech marks
- Real-time e batch

**Casos de Uso:**
- Voice assistants
- E-learning applications
- Accessibility features
- IVR systems

### Lex
Conversational AI (chatbots e voice bots).

**Recursos principais:**
- Natural language understanding
- Automatic speech recognition
- Multi-turn conversations
- Integration com Lambda
- Slot types e intents
- Built-in integrations (Slack, Facebook, etc.)

**Casos de Uso:**
- Customer service chatbots
- Voice assistants
- FAQ bots
- Virtual agents

### Transcribe
Speech-to-Text (STT) automatic.

**Recursos principais:**
- Real-time e batch
- Speaker identification
- Custom vocabulary
- 100+ languages
- Medical transcription
- Call analytics
- PII redaction

**Casos de Uso:**
- Meeting transcription
- Call center analytics
- Subtitles generation
- Content indexing

### Translate
Neural machine translation.

**Recursos principais:**
- 75+ languages
- Real-time translation
- Batch translation
- Custom terminology
- Automatic language detection
- Formality customization

**Casos de Uso:**
- Website localization
- Content translation
- Customer communication
- Document translation

### Fraud Detector
Detecção de fraude com ML.

**Recursos principais:**
- Pre-built fraud detection models
- Custom models
- Real-time predictions
- Rules engine
- Event tracking

**Casos de Uso:**
- Online payment fraud
- Account takeover prevention
- New account fraud
- Guest checkout abuse

### Personalize
Sistema de recomendação com ML.

**Recursos principais:**
- Real-time personalization
- Batch recommendations
- Similar items
- Personalized rankings
- User segmentation
- Trending items

**Casos de Uso:**
- Product recommendations
- Content recommendations
- Email personalization
- Marketing campaigns

## ML Workflow com SageMaker

```
Data Preparation
├── S3 (data storage)
├── Ground Truth (labeling)
└── Feature Store (features)
    ↓
Model Development
├── Studio (development)
├── Notebooks (experimentation)
└── Processing Jobs (data prep)
    ↓
Training
├── Built-in algorithms
├── Custom algorithms
└── Distributed training
    ↓
Tuning
└── Hyperparameter optimization
    ↓
Deployment
├── Real-time endpoints
├── Batch transform
└── Serverless inference
    ↓
Monitoring
├── Model Monitor
├── Clarify (bias detection)
└── CloudWatch metrics
```

## Comparação de Serviços

### AI Services (Pre-trained Models)
| Serviço | Domínio | Input | Output |
|---------|---------|-------|--------|
| Rekognition | Computer Vision | Images/Videos | Labels, Objects, Faces |
| Comprehend | NLP | Text | Sentiment, Entities |
| Polly | Speech | Text | Audio (speech) |
| Transcribe | Speech | Audio | Text |
| Translate | Language | Text | Translated text |
| Lex | Conversational | Text/Voice | Intent, Slots |

### ML Platform
- **SageMaker**: Custom ML models, full control, any algorithm

## Casos de Uso por Indústria

### E-commerce
- Product recommendations (Personalize)
- Image search (Rekognition)
- Chatbot support (Lex)
- Sentiment analysis (Comprehend)

### Media & Entertainment
- Content moderation (Rekognition)
- Video indexing (Rekognition, Transcribe)
- Subtitles generation (Transcribe)
- Content recommendations (Personalize)

### Financial Services
- Fraud detection (Fraud Detector, SageMaker)
- Document processing (Textract, Comprehend)
- Customer service (Lex)
- Risk modeling (SageMaker)

### Healthcare
- Medical transcription (Transcribe Medical)
- Medical image analysis (SageMaker)
- Patient communication (Polly, Lex)
- Clinical decision support (SageMaker)

## Melhores Práticas

### SageMaker
- Use Spot Instances para training
- Implement MLOps com Pipelines
- Version models e datasets
- Monitor model performance
- Implement A/B testing
- Use Feature Store para reutilização
- Automate retraining

### AI Services
- Understand pricing model
- Use batch quando possível
- Implement caching
- Monitor usage e costs
- Use custom models quando apropriado
- Test accuracy para seu use case

### Geral ML
- Start simple, iterate
- Collect quality data
- Split data properly (train/validation/test)
- Monitor model drift
- Implement feedback loops
- Document experiments
- Consider ethics e bias
- Ensure data privacy

## Arquitetura de Referência - ML Pipeline

```
Data Sources
    ↓
S3 (Data Lake)
    ↓
SageMaker Processing (Data prep)
    ↓
SageMaker Training (Model training)
    ↓
SageMaker Model Registry
    ↓
    ├── Development
    ├── Staging (A/B testing)
    └── Production
         ↓
    Real-time Endpoint
         ↓
    API Gateway + Lambda
         ↓
    Application
```

## Exemplo: SageMaker Training Job

```python
import sagemaker
from sagemaker import get_execution_role
from sagemaker.sklearn import SKLearn

role = get_execution_role()
session = sagemaker.Session()

# Configure training
sklearn_estimator = SKLearn(
    entry_point='train.py',
    role=role,
    instance_type='ml.m5.xlarge',
    framework_version='0.23-1',
    hyperparameters={
        'max_depth': 10,
        'n_estimators': 100
    }
)

# Train model
sklearn_estimator.fit({'train': 's3://bucket/train-data'})

# Deploy model
predictor = sklearn_estimator.deploy(
    initial_instance_count=1,
    instance_type='ml.t2.medium'
)

# Make prediction
result = predictor.predict(data)
```

## Exemplo: Rekognition - Detect Labels

```python
import boto3

rekognition = boto3.client('rekognition')

response = rekognition.detect_labels(
    Image={
        'S3Object': {
            'Bucket': 'my-bucket',
            'Name': 'image.jpg'
        }
    },
    MaxLabels=10,
    MinConfidence=75
)

for label in response['Labels']:
    print(f"{label['Name']}: {label['Confidence']:.2f}%")
```

## Exemplo: Comprehend - Sentiment Analysis

```python
import boto3

comprehend = boto3.client('comprehend')

text = "This product is amazing! I love it."

response = comprehend.detect_sentiment(
    Text=text,
    LanguageCode='en'
)

print(f"Sentiment: {response['Sentiment']}")
print(f"Scores: {response['SentimentScore']}")
```

## Exemplo: Lex - Chatbot

```python
import boto3

lex = boto3.client('lexv2-runtime')

response = lex.recognize_text(
    botId='bot-id',
    botAliasId='alias-id',
    localeId='en_US',
    sessionId='user-session-id',
    text='I want to book a flight to New York'
)

intent = response['sessionState']['intent']['name']
slots = response['sessionState']['intent']['slots']
```

## MLOps com SageMaker

### Componentes:
1. **Version Control**: Git + CodeCommit
2. **CI/CD**: CodePipeline + CodeBuild
3. **Model Registry**: SageMaker Model Registry
4. **Monitoring**: SageMaker Model Monitor
5. **Feature Store**: SageMaker Feature Store
6. **Pipelines**: SageMaker Pipelines

### Workflow:
```
Code Change → CI/CD Pipeline
                    ↓
            SageMaker Pipeline
                    ↓
    ├── Data Processing
    ├── Training
    ├── Evaluation
    └── Conditional Deployment
                    ↓
            Model Registry
                    ↓
        Production Endpoint
                    ↓
          Model Monitor
```

## Recursos de Aprendizado

- [AWS Machine Learning](https://aws.amazon.com/machine-learning/)
- [SageMaker Documentation](https://docs.aws.amazon.com/sagemaker/)
- [AWS ML Training](https://aws.amazon.com/training/learn-about/machine-learning/)
- [AI Services](https://aws.amazon.com/machine-learning/ai-services/)
- [Machine Learning University](https://aws.amazon.com/machine-learning/mlu/)
- [SageMaker Examples](https://github.com/aws/amazon-sagemaker-examples)
