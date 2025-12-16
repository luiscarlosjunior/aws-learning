# AWS Certificate Manager (ACM)

## O que é AWS Certificate Manager?

AWS Certificate Manager (ACM) é um serviço gerenciado que facilita o provisionamento, gerenciamento e implantação de certificados SSL/TLS públicos e privados para uso com serviços AWS e recursos internos conectados. O ACM remove a complexidade de comprar, carregar e renovar certificados SSL/TLS manualmente.

### Benefícios Principais

**Gerenciamento Simplificado:**
- Provisionamento de certificados com poucos cliques
- Renovação automática de certificados
- Integração nativa com serviços AWS

**Custo:**
- Certificados públicos gratuitos para usar com serviços AWS integrados
- Sem cobrança adicional para renovação automática

**Segurança:**
- Chaves privadas protegidas e gerenciadas pela AWS
- Conformidade com padrões da indústria
- Validação automática de domínio

## História e Evolução

### Lançamento e Motivação

**Janeiro de 2016**: AWS lança o Certificate Manager
- **Problema**: Gerenciar certificados SSL/TLS era complexo e caro
- **Custos**: Certificados comerciais custavam de $50 a $1000+ por ano
- **Renovação Manual**: Processos manuais propensos a erros
- **Downtime**: Certificados expirados causavam interrupções de serviço

**Por que foi criado?**

Antes do ACM, os clientes enfrentavam:
1. **Custos elevados** de certificados comerciais
2. **Processos complexos** de aquisição e instalação
3. **Renovação manual** propensa a falhas
4. **Gestão de chaves privadas** sensível
5. **Falta de padronização** entre ambientes

### Marcos Importantes

**2016**: Lançamento inicial
- Certificados SSL/TLS públicos gratuitos
- Renovação automática
- Integração com ELB e CloudFront

**2017**: ACM Private Certificate Authority (ACM PCA)
- Emissão de certificados privados
- Para recursos internos e aplicações

**2018**: Exportação de certificados
- Permitiu uso fora da AWS
- Suporte para workloads híbridos

**2019-2024**: Melhorias contínuas
- Mais integrações com serviços AWS
- Suporte expandido para regiões
- Validação aprimorada
- APIs mais robustas

### Por que continua relevante?

1. **HTTPS Universal**: Navegadores exigem HTTPS
2. **Segurança Padrão**: SSL/TLS é requisito básico
3. **Conformidade**: Regulamentações exigem criptografia
4. **Gratuito**: Elimina custos de certificados
5. **Automação**: Reduz carga operacional
6. **Integração**: Nativa com ecossistema AWS

## Conceitos Principais

### Tipos de Certificados

**Certificados Públicos:**
- Validados por autoridades certificadoras (CA) públicas
- Gratuitos quando usados com serviços AWS integrados
- Renovação automática
- Não podem ser exportados

**Certificados Privados:**
- Emitidos via ACM Private CA
- Para uso interno e aplicações privadas
- Podem ser exportados
- Cobrados por certificado ativo

### Validação de Certificados

**DNS Validation (Recomendada):**
- Adiciona registro CNAME ao DNS
- Automática para renovações
- Validação rápida (minutos)
- Ideal para automação

**Email Validation:**
- Email enviado para proprietários do domínio
- Requer ação manual
- Validação deve ser repetida para renovações
- Menos automatizada

### Ciclo de Vida de um Certificado

```
1. Solicitação
   ↓
2. Validação (DNS ou Email)
   ↓
3. Emissão
   ↓
4. Uso (Deploy em serviços AWS)
   ↓
5. Monitoramento
   ↓
6. Renovação Automática (60 dias antes)
   ↓
7. [Loop para renovação ou revogação]
```

### Integração com Serviços AWS

**Serviços Suportados Nativamente:**
- Elastic Load Balancing (ALB, NLB, CLB)
- Amazon CloudFront
- Amazon API Gateway
- AWS Elastic Beanstalk
- AWS App Runner
- AWS Nitro Enclaves
- Amazon CloudWatch

**Certificados Exportáveis (ACM PCA):**
- EC2 instances
- Containers (ECS, EKS)
- On-premises servers

## Funcionalidades Principais

### Renovação Automática

**Como Funciona:**
- ACM tenta renovar 60 dias antes da expiração
- Validação DNS automática (se configurada)
- Certificado renovado sem intervenção
- Serviços integrados atualizam automaticamente

**Requisitos:**
- Certificado deve estar em uso
- Validação DNS deve estar configurada
- Domínio deve ser acessível

### Wildcard Certificates

Suporte para certificados wildcard:
```
*.example.com
- api.example.com ✓
- www.example.com ✓
- blog.example.com ✓
- sub.domain.example.com ✗ (requer *.*.example.com)
```

### Multi-Domain (SAN) Certificates

Um certificado pode cobrir múltiplos domínios:
```
example.com
www.example.com
api.example.com
*.staging.example.com
```

## Guia Passo a Passo

### 1. Solicitar Certificado Público (Console)

**Passo 1: Acessar ACM Console**
```
AWS Console → Certificate Manager → Request a certificate
```

**Passo 2: Selecionar Tipo**
- Request a public certificate
- Next

**Passo 3: Configurar Domínios**
```
Domain names:
- example.com
- www.example.com
- *.example.com (opcional, para subdomínios)
```

**Passo 4: Escolher Validação**
- DNS validation (recomendado)
- Email validation

**Passo 5: Tags (opcional)**
```
Key: Environment
Value: Production
```

**Passo 6: Review e Request**

**Passo 7: Validação DNS**
- Copiar registros CNAME mostrados
- Adicionar ao DNS provider (Route 53, etc.)
- Aguardar validação (5-30 minutos)

### 2. Solicitar Certificado (AWS CLI)

```bash
# Solicitar certificado com validação DNS
aws acm request-certificate \
    --domain-name example.com \
    --subject-alternative-names www.example.com *.example.com \
    --validation-method DNS \
    --idempotency-token 91adc45q \
    --options CertificateTransparencyLoggingPreference=ENABLED \
    --tags Key=Environment,Value=Production

# Output: CertificateArn
# arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012
```

### 3. Obter Registros DNS para Validação

```bash
# Descrever certificado para obter registros DNS
aws acm describe-certificate \
    --certificate-arn arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012

# Output inclui DomainValidationOptions com Name, Type, Value
```

### 4. Adicionar Validação DNS (Route 53)

```bash
# Extrair informações de validação
CERT_ARN="arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"

# Obter registro CNAME
VALIDATION_RECORD=$(aws acm describe-certificate \
    --certificate-arn $CERT_ARN \
    --query 'Certificate.DomainValidationOptions[0].ResourceRecord' \
    --output json)

NAME=$(echo $VALIDATION_RECORD | jq -r '.Name')
VALUE=$(echo $VALIDATION_RECORD | jq -r '.Value')

# Criar registro no Route 53
HOSTED_ZONE_ID="Z1234567890ABC"

aws route53 change-resource-record-sets \
    --hosted-zone-id $HOSTED_ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "CREATE",
            "ResourceRecordSet": {
                "Name": "'$NAME'",
                "Type": "CNAME",
                "TTL": 300,
                "ResourceRecords": [{"Value": "'$VALUE'"}]
            }
        }]
    }'
```

### 5. Importar Certificado de Terceiros

```bash
# Importar certificado existente
aws acm import-certificate \
    --certificate fileb://certificate.pem \
    --certificate-chain fileb://certificate_chain.pem \
    --private-key fileb://private_key.pem \
    --tags Key=Name,Value=ImportedCert

# Estrutura dos arquivos:
# certificate.pem: Certificado público
# certificate_chain.pem: Cadeia de certificados intermediários
# private_key.pem: Chave privada (deve ser sem senha)
```

**Formato esperado:**
```
-----BEGIN CERTIFICATE-----
MIIEpAIBAAKCAQEA...
-----END CERTIFICATE-----
```

### 6. Exemplo: ACM com Application Load Balancer

**Criar ALB com HTTPS:**

```bash
# 1. Criar target group
aws elbv2 create-target-group \
    --name my-targets \
    --protocol HTTP \
    --port 80 \
    --vpc-id vpc-12345678 \
    --health-check-protocol HTTP \
    --health-check-path /health

# 2. Criar ALB
aws elbv2 create-load-balancer \
    --name my-alb \
    --subnets subnet-12345678 subnet-87654321 \
    --security-groups sg-12345678 \
    --scheme internet-facing \
    --type application \
    --ip-address-type ipv4

# 3. Criar listener HTTPS com certificado ACM
CERT_ARN="arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"
ALB_ARN="arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-alb/1234567890"
TG_ARN="arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/1234567890"

aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=$CERT_ARN \
    --default-actions Type=forward,TargetGroupArn=$TG_ARN \
    --ssl-policy ELBSecurityPolicy-TLS-1-2-2017-01

# 4. (Opcional) Redirecionar HTTP para HTTPS
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
```

**Configuração via Terraform:**

```hcl
# Certificado ACM
resource "aws_acm_certificate" "main" {
  domain_name               = "example.com"
  subject_alternative_names = ["www.example.com", "*.example.com"]
  validation_method         = "DNS"

  lifecycle {
    create_before_destroy = true
  }

  tags = {
    Name = "example-cert"
  }
}

# Validação DNS (Route 53)
resource "aws_route53_record" "cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.main.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = aws_route53_zone.main.zone_id
}

# Aguardar validação
resource "aws_acm_certificate_validation" "main" {
  certificate_arn         = aws_acm_certificate.main.arn
  validation_record_fqdns = [for record in aws_route53_record.cert_validation : record.fqdn]
}

# Application Load Balancer
resource "aws_lb" "main" {
  name               = "example-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = aws_subnet.public[*].id

  enable_deletion_protection = true
  enable_http2              = true

  tags = {
    Name = "example-alb"
  }
}

# Listener HTTPS
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.main.arn
  port              = "443"
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = aws_acm_certificate_validation.main.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.main.arn
  }
}

# Listener HTTP (redirect para HTTPS)
resource "aws_lb_listener" "http" {
  load_balancer_arn = aws_lb.main.arn
  port              = "80"
  protocol          = "HTTP"

  default_action {
    type = "redirect"

    redirect {
      port        = "443"
      protocol    = "HTTPS"
      status_code = "HTTP_301"
    }
  }
}
```

### 7. Exemplo: ACM com CloudFront

**Criar CloudFront Distribution com HTTPS:**

```bash
# Nota: Certificados para CloudFront devem estar em us-east-1

# 1. Solicitar certificado em us-east-1
aws acm request-certificate \
    --domain-name example.com \
    --subject-alternative-names www.example.com \
    --validation-method DNS \
    --region us-east-1

# 2. Criar CloudFront distribution
CERT_ARN="arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012"

aws cloudfront create-distribution --distribution-config '{
    "CallerReference": "my-distribution-2024",
    "Comment": "My CloudFront Distribution",
    "Enabled": true,
    "Origins": {
        "Quantity": 1,
        "Items": [{
            "Id": "S3-my-bucket",
            "DomainName": "my-bucket.s3.amazonaws.com",
            "S3OriginConfig": {
                "OriginAccessIdentity": ""
            }
        }]
    },
    "DefaultCacheBehavior": {
        "TargetOriginId": "S3-my-bucket",
        "ViewerProtocolPolicy": "redirect-to-https",
        "TrustedSigners": {
            "Enabled": false,
            "Quantity": 0
        },
        "ForwardedValues": {
            "QueryString": false,
            "Cookies": {"Forward": "none"}
        },
        "MinTTL": 0
    },
    "ViewerCertificate": {
        "ACMCertificateArn": "'$CERT_ARN'",
        "SSLSupportMethod": "sni-only",
        "MinimumProtocolVersion": "TLSv1.2_2021"
    },
    "Aliases": {
        "Quantity": 2,
        "Items": ["example.com", "www.example.com"]
    }
}'
```

**Terraform para CloudFront com ACM:**

```hcl
# Certificado em us-east-1 (obrigatório para CloudFront)
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "cloudfront" {
  provider                  = aws.us_east_1
  domain_name              = "example.com"
  subject_alternative_names = ["www.example.com"]
  validation_method        = "DNS"

  lifecycle {
    create_before_destroy = true
  }
}

# Validação DNS
resource "aws_route53_record" "cloudfront_cert_validation" {
  for_each = {
    for dvo in aws_acm_certificate.cloudfront.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }

  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = aws_route53_zone.main.zone_id
}

resource "aws_acm_certificate_validation" "cloudfront" {
  provider                = aws.us_east_1
  certificate_arn         = aws_acm_certificate.cloudfront.arn
  validation_record_fqdns = [for record in aws_route53_record.cloudfront_cert_validation : record.fqdn]
}

# S3 bucket para origem
resource "aws_s3_bucket" "website" {
  bucket = "my-website-bucket"
}

# CloudFront Origin Access Identity
resource "aws_cloudfront_origin_access_identity" "main" {
  comment = "OAI for my website"
}

# CloudFront Distribution
resource "aws_cloudfront_distribution" "main" {
  enabled             = true
  is_ipv6_enabled     = true
  comment             = "My website distribution"
  default_root_object = "index.html"
  aliases             = ["example.com", "www.example.com"]

  origin {
    domain_name = aws_s3_bucket.website.bucket_regional_domain_name
    origin_id   = "S3-${aws_s3_bucket.website.id}"

    s3_origin_config {
      origin_access_identity = aws_cloudfront_origin_access_identity.main.cloudfront_access_identity_path
    }
  }

  default_cache_behavior {
    allowed_methods        = ["GET", "HEAD", "OPTIONS"]
    cached_methods         = ["GET", "HEAD"]
    target_origin_id       = "S3-${aws_s3_bucket.website.id}"
    viewer_protocol_policy = "redirect-to-https"

    forwarded_values {
      query_string = false
      cookies {
        forward = "none"
      }
    }

    min_ttl     = 0
    default_ttl = 3600
    max_ttl     = 86400
    compress    = true
  }

  # Certificado ACM
  viewer_certificate {
    acm_certificate_arn      = aws_acm_certificate_validation.cloudfront.certificate_arn
    ssl_support_method       = "sni-only"
    minimum_protocol_version = "TLSv1.2_2021"
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  tags = {
    Name = "my-website-distribution"
  }
}

# Route 53 Records
resource "aws_route53_record" "www" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "www.example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}

resource "aws_route53_record" "apex" {
  zone_id = aws_route53_zone.main.zone_id
  name    = "example.com"
  type    = "A"

  alias {
    name                   = aws_cloudfront_distribution.main.domain_name
    zone_id                = aws_cloudfront_distribution.main.hosted_zone_id
    evaluate_target_health = false
  }
}
```

### 8. Exemplo: Python Boto3

```python
import boto3
import time

class ACMManager:
    def __init__(self, region='us-east-1'):
        self.acm = boto3.client('acm', region_name=region)
        self.route53 = boto3.client('route53')
    
    def request_certificate(self, domain_name, san_list=None):
        """Solicita novo certificado"""
        params = {
            'DomainName': domain_name,
            'ValidationMethod': 'DNS',
            'Options': {
                'CertificateTransparencyLoggingPreference': 'ENABLED'
            }
        }
        
        if san_list:
            params['SubjectAlternativeNames'] = san_list
        
        response = self.acm.request_certificate(**params)
        cert_arn = response['CertificateArn']
        
        print(f"Certificado solicitado: {cert_arn}")
        return cert_arn
    
    def get_validation_records(self, cert_arn):
        """Obtém registros DNS para validação"""
        # Aguardar ACM processar a solicitação
        time.sleep(5)
        
        response = self.acm.describe_certificate(CertificateArn=cert_arn)
        
        validation_options = response['Certificate']['DomainValidationOptions']
        records = []
        
        for option in validation_options:
            if 'ResourceRecord' in option:
                record = option['ResourceRecord']
                records.append({
                    'domain': option['DomainName'],
                    'name': record['Name'],
                    'type': record['Type'],
                    'value': record['Value']
                })
        
        return records
    
    def create_validation_records(self, cert_arn, hosted_zone_id):
        """Cria registros DNS para validação no Route 53"""
        records = self.get_validation_records(cert_arn)
        
        changes = []
        for record in records:
            changes.append({
                'Action': 'UPSERT',
                'ResourceRecordSet': {
                    'Name': record['name'],
                    'Type': record['type'],
                    'TTL': 300,
                    'ResourceRecords': [{'Value': record['value']}]
                }
            })
        
        if changes:
            response = self.route53.change_resource_record_sets(
                HostedZoneId=hosted_zone_id,
                ChangeBatch={'Changes': changes}
            )
            print(f"Registros DNS criados. Change ID: {response['ChangeInfo']['Id']}")
            return response['ChangeInfo']['Id']
        
        return None
    
    def wait_for_validation(self, cert_arn, timeout=900):
        """Aguarda validação do certificado"""
        print("Aguardando validação...")
        start_time = time.time()
        
        while time.time() - start_time < timeout:
            response = self.acm.describe_certificate(CertificateArn=cert_arn)
            status = response['Certificate']['Status']
            
            print(f"Status: {status}")
            
            if status == 'ISSUED':
                print("Certificado emitido com sucesso!")
                return True
            elif status == 'FAILED':
                print("Validação falhou")
                return False
            
            time.sleep(30)
        
        print("Timeout aguardando validação")
        return False
    
    def list_certificates(self, status='ISSUED'):
        """Lista certificados"""
        paginator = self.acm.get_paginator('list_certificates')
        
        certificates = []
        for page in paginator.paginate(CertificateStatuses=[status]):
            certificates.extend(page['CertificateSummaryList'])
        
        return certificates
    
    def get_certificate_details(self, cert_arn):
        """Obtém detalhes do certificado"""
        response = self.acm.describe_certificate(CertificateArn=cert_arn)
        cert = response['Certificate']
        
        return {
            'arn': cert['CertificateArn'],
            'domain': cert['DomainName'],
            'status': cert['Status'],
            'type': cert['Type'],
            'san': cert.get('SubjectAlternativeNames', []),
            'created': cert['CreatedAt'],
            'not_before': cert.get('NotBefore'),
            'not_after': cert.get('NotAfter'),
            'issuer': cert.get('Issuer'),
            'in_use': len(cert.get('InUseBy', [])) > 0
        }
    
    def import_certificate(self, cert_file, private_key_file, chain_file):
        """Importa certificado de terceiros"""
        with open(cert_file, 'rb') as f:
            certificate = f.read()
        
        with open(private_key_file, 'rb') as f:
            private_key = f.read()
        
        with open(chain_file, 'rb') as f:
            certificate_chain = f.read()
        
        response = self.acm.import_certificate(
            Certificate=certificate,
            PrivateKey=private_key,
            CertificateChain=certificate_chain
        )
        
        print(f"Certificado importado: {response['CertificateArn']}")
        return response['CertificateArn']
    
    def delete_certificate(self, cert_arn):
        """Deleta certificado"""
        try:
            self.acm.delete_certificate(CertificateArn=cert_arn)
            print(f"Certificado deletado: {cert_arn}")
            return True
        except Exception as e:
            print(f"Erro ao deletar certificado: {e}")
            return False
    
    def check_renewal_eligibility(self, cert_arn):
        """Verifica elegibilidade para renovação"""
        response = self.acm.describe_certificate(CertificateArn=cert_arn)
        cert = response['Certificate']
        
        renewal_eligible = cert.get('RenewalEligibility', 'INELIGIBLE')
        renewal_summary = cert.get('RenewalSummary', {})
        
        return {
            'eligible': renewal_eligible == 'ELIGIBLE',
            'status': renewal_summary.get('RenewalStatus'),
            'reason': renewal_summary.get('RenewalStatusReason'),
            'updated': renewal_summary.get('UpdatedAt')
        }

# Uso
manager = ACMManager(region='us-east-1')

# Solicitar certificado
cert_arn = manager.request_certificate(
    domain_name='example.com',
    san_list=['www.example.com', '*.example.com']
)

# Criar registros de validação
hosted_zone_id = 'Z1234567890ABC'
manager.create_validation_records(cert_arn, hosted_zone_id)

# Aguardar validação
manager.wait_for_validation(cert_arn)

# Listar certificados
certs = manager.list_certificates()
for cert in certs:
    details = manager.get_certificate_details(cert['CertificateArn'])
    print(f"Domain: {details['domain']}, Status: {details['status']}, In Use: {details['in_use']}")

# Verificar renovação
renewal_info = manager.check_renewal_eligibility(cert_arn)
print(f"Renewal eligible: {renewal_info['eligible']}")
```

## Segurança e Compliance

### Renovação Automática

**Como Funciona:**

1. **60 dias antes da expiração**: ACM inicia processo de renovação
2. **Validação**: Se DNS validation está configurada, renovação é automática
3. **Novo certificado**: Emitido com nova chave privada
4. **Deploy automático**: Serviços integrados atualizam automaticamente
5. **Sem downtime**: Transição é transparente

**Requisitos para Renovação Automática:**
- Certificado deve estar em uso por serviço AWS
- Validação DNS configurada corretamente
- Registros DNS acessíveis
- Certificado não importado (apenas ACM-issued)

**Monitoramento de Renovação:**

```python
import boto3
from datetime import datetime, timedelta

def check_certificates_expiration(region='us-east-1'):
    """Verifica certificados próximos da expiração"""
    acm = boto3.client('acm', region_name=region)
    
    response = acm.list_certificates(CertificateStatuses=['ISSUED'])
    
    warning_threshold = timedelta(days=30)
    now = datetime.now(tz=None)
    
    for cert_summary in response['CertificateSummaryList']:
        cert = acm.describe_certificate(
            CertificateArn=cert_summary['CertificateArn']
        )['Certificate']
        
        not_after = cert['NotAfter'].replace(tzinfo=None)
        days_until_expiration = (not_after - now).days
        
        if days_until_expiration < warning_threshold.days:
            print(f"⚠️  {cert['DomainName']}: {days_until_expiration} dias")
            
            renewal = cert.get('RenewalSummary', {})
            if renewal:
                print(f"   Status de renovação: {renewal.get('RenewalStatus')}")
        else:
            print(f"✓  {cert['DomainName']}: {days_until_expiration} dias")

check_certificates_expiration()
```

**CloudWatch Alarm para Expiração:**

```bash
# Criar métrica customizada
aws cloudwatch put-metric-alarm \
    --alarm-name acm-certificate-expiring \
    --alarm-description "Alert when certificate expires in 30 days" \
    --metric-name DaysToExpiry \
    --namespace ACM \
    --statistic Minimum \
    --period 86400 \
    --threshold 30 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 1
```

### Validação e Segurança

**Certificate Transparency Logging:**
- Habilitado por padrão
- Registros públicos de certificados emitidos
- Previne emissão não autorizada
- Compliance com políticas de segurança

**Proteção de Chaves Privadas:**
- Gerenciadas pela AWS
- Armazenadas em HSMs (Hardware Security Modules)
- Nunca expostas ou exportáveis (certificados públicos)
- Rotação automática na renovação

**Validação de Domínio:**
- Garante controle do domínio
- DNS validation mais segura que email
- Renovação automática sem intervenção

### Melhores Práticas

**1. Use DNS Validation**
```
✓ Automática
✓ Renovação sem intervenção
✓ Mais rápida
✓ Mais segura
```

**2. Wildcard para Subdomínios**
```
*.example.com
- Cobre todos os subdomínios de primeiro nível
- Reduz número de certificados
- Simplifica gestão
```

**3. Monitore Expiração**
```python
# Script de monitoramento diário
# CloudWatch Events + Lambda
# SNS notifications para alertas
```

**4. Use Tags**
```bash
aws acm add-tags-to-certificate \
    --certificate-arn $CERT_ARN \
    --tags Key=Environment,Value=Production \
           Key=Team,Value=DevOps \
           Key=CostCenter,Value=Engineering
```

**5. Multi-Region para DR**
```
Certificados são regionais
- Solicite em cada região necessária
- us-east-1 obrigatório para CloudFront
- Cross-region para disaster recovery
```

**6. Integração com AWS Config**
```
Rastreie configurações de certificados
- Compliance checks
- Audit trail
- Configuration history
```

**7. Política de SSL/TLS Forte**
```
ALB/NLB:
- ELBSecurityPolicy-TLS-1-2-2017-01 (mínimo)
- ELBSecurityPolicy-TLS-1-2-Ext-2018-06 (recomendado)

CloudFront:
- TLSv1.2_2021 (recomendado)
```

## Limitações e Questões Comuns

### Limitações por Região

**Certificados Públicos:**
- 2,500 certificados por região por ano (pode aumentar)
- 100 certificados ACM por conta (pode aumentar)

**Certificados Privados (ACM PCA):**
- Limite baseado em CA criadas
- Custos por certificado ativo

**Rate Limits:**
- 10 certificados por segundo (DescribeCertificate)
- 5 solicitações por segundo (RequestCertificate)
- 5 importações por segundo (ImportCertificate)

### Restrições Importantes

**Certificados Públicos ACM:**
- ✗ Não podem ser exportados
- ✗ Devem ser usados com serviços AWS integrados
- ✗ Não suportam certificados de código
- ✓ Renovação automática
- ✓ Gratuitos

**Certificados Privados ACM:**
- ✓ Podem ser exportados
- ✓ Uso fora da AWS
- ✗ Cobrados por certificado ativo
- ✗ Requer ACM Private CA ($400/mês)

**Regiões Especiais:**
- CloudFront: Certificado deve estar em **us-east-1**
- API Gateway Edge: Certificado deve estar em **us-east-1**
- API Gateway Regional: Certificado na mesma região

### Troubleshooting Comum

**1. Validação DNS não completa**

**Problema**: Certificado fica "Pending validation"

**Soluções**:
```bash
# Verificar registro DNS
dig _abc123.example.com CNAME

# Verificar se registros estão corretos
aws acm describe-certificate --certificate-arn $CERT_ARN \
    --query 'Certificate.DomainValidationOptions[*].ResourceRecord'

# Verificar TTL do DNS
# Aguardar propagação DNS (até 48h, geralmente < 1h)
```

**2. Certificado não renovando automaticamente**

**Problema**: Certificado próximo da expiração, sem renovação

**Verificações**:
```python
# Verificar elegibilidade
response = acm.describe_certificate(CertificateArn=cert_arn)
print(response['Certificate']['RenewalEligibility'])
print(response['Certificate'].get('RenewalSummary'))

# Causas comuns:
# - Certificado não está em uso
# - Validação DNS removida/alterada
# - Email validation (não automática)
# - Certificado importado (não renova)
```

**Solução**:
```bash
# Verificar se certificado está em uso
aws acm describe-certificate --certificate-arn $CERT_ARN \
    --query 'Certificate.InUseBy'

# Verificar registros DNS de validação
# Re-importar se for certificado importado
```

**3. Erro ao usar certificado em CloudFront**

**Problema**: "Certificate not found" ou região incorreta

**Solução**:
```bash
# CloudFront requer certificado em us-east-1
aws acm request-certificate \
    --domain-name example.com \
    --region us-east-1  # OBRIGATÓRIO

# Verificar região do certificado
aws acm describe-certificate \
    --certificate-arn $CERT_ARN \
    --region us-east-1
```

**4. Certificado em uso não pode ser deletado**

**Problema**: "Certificate is in use"

**Solução**:
```bash
# Listar onde certificado está sendo usado
aws acm describe-certificate --certificate-arn $CERT_ARN \
    --query 'Certificate.InUseBy'

# Remover de todos os recursos primeiro:
# - ALB/NLB listeners
# - CloudFront distributions
# - API Gateway
# Depois deletar certificado
```

**5. Importação falha**

**Problema**: Erro ao importar certificado

**Verificações**:
```bash
# Formato correto (PEM)
# Chave privada sem senha
openssl rsa -in encrypted.key -out decrypted.key

# Certificado e chave devem corresponder
openssl x509 -noout -modulus -in certificate.pem | openssl md5
openssl rsa -noout -modulus -in private.key | openssl md5
# MD5 deve ser igual

# Cadeia completa de certificados
# certificate.pem + intermediates + root CA
```

**6. Wildcard não cobre subdomínios aninhados**

**Problema**: `*.example.com` não cobre `api.staging.example.com`

**Solução**:
```bash
# Solicitar certificados adicionais
*.example.com          # www.example.com ✓
*.staging.example.com  # api.staging.example.com ✓

# Ou adicionar como SAN
aws acm request-certificate \
    --domain-name example.com \
    --subject-alternative-names \
        "*.example.com" \
        "*.staging.example.com" \
        "*.prod.example.com"
```

**7. Renovação Email Validation**

**Problema**: Email validation requer ação manual para renovação

**Solução**:
```bash
# Preferir DNS validation
# Ou responder emails de renovação em tempo hábil

# Migrar para DNS validation:
# 1. Solicitar novo certificado com DNS validation
# 2. Atualizar recursos para usar novo certificado
# 3. Deletar certificado antigo com email validation
```

## Casos de Uso Comuns

### 1. Site Estático no S3 + CloudFront

**Cenário**: Website estático com HTTPS customizado

```bash
# 1. Certificado (us-east-1)
aws acm request-certificate \
    --domain-name example.com \
    --subject-alternative-names www.example.com \
    --validation-method DNS \
    --region us-east-1

# 2. Validação DNS
# 3. CloudFront com ACM
# 4. Route 53 apontando para CloudFront
```

**Benefícios:**
- HTTPS gratuito
- Renovação automática
- Performance global
- Baixo custo

### 2. API em ALB com Auto Scaling

**Cenário**: API REST em EC2 com ALB

```bash
# 1. Certificado regional
# 2. ALB com HTTPS listener
# 3. Target group com EC2s
# 4. Auto Scaling group

# Resultado:
# - HTTPS na edge
# - HTTP interno (segurança via SG)
# - Certificado gerenciado
```

### 3. Multi-Region Application

**Cenário**: Aplicação em múltiplas regiões

```bash
# us-east-1
aws acm request-certificate --domain-name app.example.com --region us-east-1

# eu-west-1
aws acm request-certificate --domain-name app.example.com --region eu-west-1

# ap-southeast-1
aws acm request-certificate --domain-name app.example.com --region ap-southeast-1

# Route 53 geolocation routing
# ALBs regionais com certificados locais
```

### 4. Microservices com API Gateway

**Cenário**: Múltiplos microsserviços

```bash
# Certificado wildcard
*.api.example.com

# API Gateway custom domains:
users.api.example.com
orders.api.example.com
payments.api.example.com
products.api.example.com
```

### 5. Workloads Híbridos com ACM PCA

**Cenário**: Certificados privados para recursos internos

```bash
# 1. Criar Private CA
aws acm-pca create-certificate-authority \
    --certificate-authority-configuration file://ca-config.json \
    --certificate-authority-type ROOT

# 2. Emitir certificados privados
# 3. Exportar para uso em:
#    - On-premises servers
#    - EC2 instances
#    - Containers
#    - IoT devices
```

## Integração com Outros Serviços

### CloudWatch Integration

```python
import boto3

def create_acm_dashboard():
    """Cria dashboard CloudWatch para ACM"""
    cloudwatch = boto3.client('cloudwatch')
    
    dashboard_body = {
        "widgets": [
            {
                "type": "metric",
                "properties": {
                    "metrics": [
                        ["AWS/CertificateManager", "DaysToExpiry", {"stat": "Minimum"}]
                    ],
                    "period": 86400,
                    "stat": "Minimum",
                    "region": "us-east-1",
                    "title": "Certificate Expiry"
                }
            }
        ]
    }
    
    cloudwatch.put_dashboard(
        DashboardName='ACM-Monitoring',
        DashboardBody=str(dashboard_body)
    )
```

### AWS Config Rules

```python
# Verificar certificados expirados
{
    "ConfigRuleName": "acm-certificate-expiration-check",
    "Description": "Checks if certificates are expiring soon",
    "Source": {
        "Owner": "AWS",
        "SourceIdentifier": "ACM_CERTIFICATE_EXPIRATION_CHECK"
    },
    "InputParameters": {
        "daysToExpiration": "30"
    }
}
```

### EventBridge for Automation

```json
{
  "source": ["aws.acm"],
  "detail-type": ["ACM Certificate Approaching Expiration"],
  "detail": {
    "DaysToExpiry": [30]
  }
}
```

## Comandos Úteis

```bash
# Listar certificados
aws acm list-certificates

# Listar apenas emitidos
aws acm list-certificates --certificate-statuses ISSUED

# Descrever certificado específico
aws acm describe-certificate --certificate-arn $CERT_ARN

# Solicitar certificado
aws acm request-certificate \
    --domain-name example.com \
    --validation-method DNS

# Adicionar tags
aws acm add-tags-to-certificate \
    --certificate-arn $CERT_ARN \
    --tags Key=Name,Value=MyApp

# Listar tags
aws acm list-tags-for-certificate --certificate-arn $CERT_ARN

# Remover tags
aws acm remove-tags-from-certificate \
    --certificate-arn $CERT_ARN \
    --tags Key=Name

# Deletar certificado
aws acm delete-certificate --certificate-arn $CERT_ARN

# Exportar certificado privado (ACM PCA)
aws acm export-certificate \
    --certificate-arn $CERT_ARN \
    --passphrase $(echo -n "mypassword" | base64)

# Renovar certificado (não necessário, automático)
# ACM renova automaticamente

# Verificar uso do certificado
aws acm describe-certificate --certificate-arn $CERT_ARN \
    --query 'Certificate.InUseBy'

# Importar certificado
aws acm import-certificate \
    --certificate fileb://cert.pem \
    --private-key fileb://key.pem \
    --certificate-chain fileb://chain.pem
```

## Custo

### Certificados Públicos
- **Gratuito** quando usado com:
  - Elastic Load Balancing
  - CloudFront
  - API Gateway
  - Elastic Beanstalk
  - App Runner

### Certificados Privados (ACM PCA)
- **Private CA**: $400/mês
- **Certificados**: $0.75/mês por certificado ativo
- **Primeiros 1000 certificados**: $0.75 cada
- **1001-10000**: $0.35 cada
- **10001+**: $0.001 cada

### Não Há Cobrança Para:
- Solicitação de certificados
- Validação de domínio
- Renovação automática
- Transferência de dados

## Recursos de Aprendizado

### Documentação Oficial
- [AWS Certificate Manager Documentation](https://docs.aws.amazon.com/acm/)
- [ACM User Guide](https://docs.aws.amazon.com/acm/latest/userguide/)
- [ACM API Reference](https://docs.aws.amazon.com/acm/latest/APIReference/)
- [ACM Private CA](https://docs.aws.amazon.com/acm-pca/)

### Best Practices
- [ACM Best Practices](https://docs.aws.amazon.com/acm/latest/userguide/acm-bestpractices.html)
- [SSL/TLS Best Practices for ALB](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)
- [CloudFront SSL/TLS](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https.html)

### Tutoriais
- [Request a Public Certificate](https://docs.aws.amazon.com/acm/latest/userguide/gs-acm-request-public.html)
- [Validate with DNS](https://docs.aws.amazon.com/acm/latest/userguide/dns-validation.html)
- [Import Certificate](https://docs.aws.amazon.com/acm/latest/userguide/import-certificate.html)

### Vídeos
- [AWS re:Invent - Certificate Manager](https://www.youtube.com/results?search_query=aws+reinvent+certificate+manager)
- [AWS Certificate Manager Overview](https://www.youtube.com/watch?v=6mQRv0TYpvg)

### Blogs
- [AWS Security Blog - ACM](https://aws.amazon.com/blogs/security/tag/aws-certificate-manager/)
- [How to use ACM for automated certificate renewal](https://aws.amazon.com/blogs/security/how-to-use-aws-certificate-manager-with-aws-cloud-development-kit/)

### Ferramentas
- [ACM Certificate Automation](https://github.com/awslabs/aws-certificate-manager-automation)
- [Terraform ACM Module](https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/acm_certificate)
- [CloudFormation ACM](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-resource-certificatemanager-certificate.html)

### Comparações
- [ACM vs Let's Encrypt](https://www.freecodecamp.org/news/aws-certificate-manager-vs-lets-encrypt/)
- [ACM vs Traditional CAs](https://aws.amazon.com/certificate-manager/faqs/)

### Comunidade
- [Stack Overflow - ACM](https://stackoverflow.com/questions/tagged/aws-certificate-manager)
- [AWS Forums - ACM](https://repost.aws/tags/TA4iqAM-6HRBWKhYKOKTNp6A/aws-certificate-manager)
- [Reddit r/aws](https://www.reddit.com/r/aws/)
