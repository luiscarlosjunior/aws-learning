# Algoritmos de Criptografia em AWS

## Visão Geral

Este documento explora em profundidade os algoritmos de criptografia utilizados pela AWS Certificate Manager (ACM) e outros serviços AWS. Compreender esses algoritmos é essencial para garantir a segurança adequada, especialmente ao trabalhar com autenticação mTLS (mutual TLS) em serviços como MSK (Managed Streaming for Apache Kafka) e Lambda.

## Contexto Histórico

### Por que a Criptografia é Importante na AWS?

A criptografia é fundamental para proteger dados em trânsito e em repouso na nuvem. A AWS utiliza vários algoritmos criptográficos em diferentes serviços:

- **ACM (Certificate Manager)**: Gerenciamento de certificados SSL/TLS
- **KMS (Key Management Service)**: Criptografia de dados em repouso
- **MSK**: Comunicação segura entre brokers e clientes Kafka
- **Lambda**: Variáveis de ambiente criptografadas e comunicação segura

### Evolução dos Algoritmos

**1970s-1980s**: Desenvolvimento de algoritmos fundamentais
- RSA (1977): Primeiro algoritmo de chave pública prático
- DES (1977): Padrão de criptografia de dados

**1990s-2000s**: Melhorias e novas abordagens
- AES (2001): Substituiu DES como padrão
- Curvas Elípticas: Criptografia mais eficiente

**2010s-presente**: Foco em segurança e desempenho
- Algoritmos pós-quânticos em desenvolvimento
- Depreciação de algoritmos mais fracos (SHA-1, MD5)

## RSA (Rivest-Shamir-Adleman)

### O que é RSA?

RSA é um algoritmo de criptografia assimétrica (chave pública) inventado em 1977 por Ron Rivest, Adi Shamir e Leonard Adleman no MIT. É um dos algoritmos mais amplamente utilizados para transmissão segura de dados.

### Como Funciona o RSA?

RSA baseia-se na dificuldade matemática de fatorar números grandes. O algoritmo utiliza dois números primos grandes para gerar um par de chaves: uma pública e uma privada.

#### Geração de Chaves

**Passo 1: Escolher dois números primos grandes**
```
p = 61 (exemplo simplificado - na prática, usa-se primos com centenas de dígitos)
q = 53
```

**Passo 2: Calcular n (módulo)**
```
n = p × q = 61 × 53 = 3233
```

**Passo 3: Calcular φ(n) - função totiente de Euler**
```
φ(n) = (p-1) × (q-1) = 60 × 52 = 3120
```

**Passo 4: Escolher expoente público e**
```
e = 17 (deve ser coprimo com φ(n) e 1 < e < φ(n))
Geralmente, utiliza-se e = 65537 (2¹⁶ + 1), que é o expoente público padrão em implementações RSA.
```

**Passo 5: Calcular expoente privado d**
```
d × e ≡ 1 (mod φ(n))
d = 2753 (inverso modular de e)
```

**Resultado:**
- **Chave Pública**: (n, e) = (3233, 17)
- **Chave Privada**: (n, d) = (3233, 2753)

#### Criptografia e Descriptografia

**Criptografar (usando chave pública):**
```
Mensagem m = 123
Cifrado c = m^e mod n
c = 123^17 mod 3233 = 855
```

**Descriptografar (usando chave privada):**
```
Cifrado c = 855
Mensagem m = c^d mod n
m = 855^2753 mod 3233 = 123
```

### Tamanhos de Chave RSA

| Tamanho | Segurança | Uso Atual | Recomendação |
|---------|-----------|-----------|--------------|
| 1024 bits | Fraca | Depreciado | ❌ Não usar |
| 2048 bits | Adequada | Padrão atual | ✅ Uso geral |
| 3072 bits | Forte | Compliance específico | ✅ Alta segurança |
| 4096 bits | Muito forte | Requisitos rigorosos | ✅ Máxima segurança |

**Por que 2048 bits é o mínimo recomendado?**
- Com o poder computacional atual, chaves de 1024 bits podem ser quebradas
- NIST recomenda 2048 bits como mínimo até 2030
- 2048 bits oferece segurança equivalente a AES-112

### RSA na AWS

#### AWS Certificate Manager (ACM)

**Tipos de certificados RSA suportados:**
```bash
# Solicitar certificado RSA 2048 bits (padrão)
aws acm request-certificate \
    --domain-name example.com \
    --validation-method DNS \
    --key-algorithm RSA_2048

# Solicitar certificado RSA 4096 bits (maior segurança)
aws acm request-certificate \
    --domain-name example.com \
    --validation-method DNS \
    --key-algorithm RSA_4096
```

**Algoritmos RSA disponíveis no ACM:**
- `RSA_1024`: Depreciado, não recomendado
- `RSA_2048`: Padrão, recomendado para uso geral
- `RSA_3072`: Alta segurança
- `RSA_4096`: Máxima segurança

#### Importação de Certificados RSA

Ao importar certificados para ACM, a chave privada RSA deve estar em formato PEM sem senha:

```bash
# Remover senha de chave privada
openssl rsa -in encrypted_private.key -out decrypted_private.key

# Importar para ACM
aws acm import-certificate \
    --certificate fileb://certificate.pem \
    --private-key fileb://decrypted_private.key \
    --certificate-chain fileb://chain.pem
```

### Vantagens e Desvantagens do RSA

**Vantagens:**
- ✅ Amplamente suportado e testado
- ✅ Implementações maduras e confiáveis
- ✅ Suporte universal em navegadores e sistemas
- ✅ Pode ser usado para criptografia e assinatura digital

**Desvantagens:**
- ❌ Chaves grandes (2048+ bits) resultam em operações mais lentas
- ❌ Requer mais poder computacional que ECC para mesma segurança
- ❌ Certificados maiores aumentam latência em handshake TLS
- ❌ Vulnerável a computação quântica (no futuro)

### Assinatura Digital com RSA

RSA também é usado para assinaturas digitais:

**Processo de assinatura:**
```
1. Calcular hash da mensagem: h = SHA-256(mensagem)
2. "Criptografar" hash com chave privada: s = h^d mod n
3. Assinatura: s
```

**Verificação:**
```
1. "Descriptografar" assinatura com chave pública: h' = s^e mod n
2. Calcular hash da mensagem: h = SHA-256(mensagem)
3. Comparar: h' == h?
```

**Exemplo prático:**
```bash
# Criar chave privada RSA
openssl genrsa -out private.key 2048

# Extrair chave pública
openssl rsa -in private.key -pubout -out public.key

# Assinar arquivo
openssl dgst -sha256 -sign private.key -out signature.bin data.txt

# Verificar assinatura
openssl dgst -sha256 -verify public.key -signature signature.bin data.txt
```

## Criptografia de Curva Elíptica (ECC)

### O que é ECC?

Elliptic Curve Cryptography (ECC) é uma abordagem de criptografia de chave pública baseada na estrutura algébrica de curvas elípticas sobre corpos finitos. Foi proposta independentemente por Neal Koblitz e Victor Miller em 1985.

### Como Funciona ECC?

ECC baseia-se no problema matemático do logaritmo discreto de curva elíptica (ECDLP), que é computacionalmente difícil de resolver.

#### Fundamentos Matemáticos

**Equação da Curva Elíptica:**
```
y² = x³ + ax + b

Exemplo: secp256r1 (P-256)
y² = x³ - 3x + b (mod p)
onde p é um número primo grande
```

**Operação de Grupo - Adição de Pontos:**

Na curva elíptica, define-se uma operação de "adição" entre pontos:
- P + Q = R (soma de dois pontos resulta em terceiro ponto)
- P + P = 2P (dobrar um ponto)
- kP = P + P + ... + P (k vezes) - multiplicação escalar

**Geração de Chaves:**

```
1. Escolher curva elíptica padrão (ex: P-256)
2. Ponto gerador G (ponto conhecido na curva)
3. Escolher número privado aleatório: d (chave privada)
4. Calcular Q = d × G (chave pública)
```

**Por que é seguro?**
- Dado Q e G, é computacionalmente inviável calcular d
- Problema do logaritmo discreto de curva elíptica (ECDLP)

### Curvas Elípticas Comuns

#### NIST Curves (recomendadas pelo NIST)

| Curva | Bits | Segurança Equivalente | Uso |
|-------|------|----------------------|-----|
| P-256 (secp256r1) | 256 | ~128-bit (AES-128) | ✅ Recomendado, uso geral |
| P-384 (secp384r1) | 384 | ~192-bit (AES-192) | ✅ Alta segurança |
| P-521 (secp521r1) | 521 | ~256-bit (AES-256) | ✅ Máxima segurança |

#### Outras Curvas

| Curva | Tipo | Características | Uso |
|-------|------|-----------------|-----|
| secp256k1 | Koblitz | Usada em Bitcoin | Blockchain |
| Curve25519 | Montgomery | Rápida e segura | SSH, TLS 1.3 |
| Ed25519 | Edwards | Assinaturas rápidas | SSH, GPG |

### Comparação ECC vs RSA

**Tamanho de Chave para Segurança Equivalente:**

| Segurança | RSA | ECC | Fator |
|-----------|-----|-----|-------|
| 80 bits | 1024 bits | 160 bits | 6.4x |
| 112 bits | 2048 bits | 224 bits | 9.1x |
| 128 bits | 3072 bits | 256 bits | 12x |
| 192 bits | 7680 bits | 384 bits | 20x |
| 256 bits | 15360 bits | 521 bits | 29x |

**Implicações Práticas:**
- Chaves ECC são **muito menores** que RSA para mesma segurança
- Operações ECC são **mais rápidas** (menos CPU)
- Certificados ECC são **menores** (menos largura de banda)
- Handshake TLS é **mais rápido** com ECC

### ECC na AWS

#### AWS Certificate Manager (ACM)

```bash
# Solicitar certificado com curva elíptica P-256
aws acm request-certificate \
    --domain-name example.com \
    --validation-method DNS \
    --key-algorithm EC_prime256v1

# Solicitar certificado com curva elíptica P-384
aws acm request-certificate \
    --domain-name example.com \
    --validation-method DNS \
    --key-algorithm EC_secp384r1
```

**Algoritmos ECC disponíveis no ACM:**
- `EC_prime256v1`: P-256 / secp256r1 (recomendado)
- `EC_secp384r1`: P-384 (alta segurança)
- `EC_secp521r1`: P-521 (máxima segurança - suporte limitado)

#### Exemplo: Gerar Certificado ECC

```bash
# 1. Gerar chave privada ECC (P-256)
openssl ecparam -genkey -name prime256v1 -out ec-private-key.pem

# 2. Gerar CSR (Certificate Signing Request)
openssl req -new -key ec-private-key.pem -out csr.pem \
    -subj "/C=BR/ST=SP/L=SaoPaulo/O=MyCompany/CN=example.com"

# 3. Gerar certificado auto-assinado (teste)
openssl x509 -req -days 365 -in csr.pem \
    -signkey ec-private-key.pem -out ec-certificate.pem

# 4. Verificar certificado
openssl x509 -in ec-certificate.pem -text -noout | grep "Public Key Algorithm"
# Saída: Public Key Algorithm: id-ecPublicKey
# (Algoritmo de chave pública: id-ecPublicKey)

# 5. Ver curva utilizada
openssl x509 -in ec-certificate.pem -text -noout | grep "ASN1 OID"
# Saída: ASN1 OID: prime256v1
# (Curva utilizada: prime256v1)
```

### Vantagens e Desvantagens de ECC

**Vantagens:**
- ✅ Chaves menores para segurança equivalente
- ✅ Operações mais rápidas (melhor performance)
- ✅ Menor uso de CPU e memória
- ✅ Certificados menores (menor latência)
- ✅ Melhor para dispositivos com recursos limitados (IoT)

**Desvantagens:**
- ❌ Suporte ligeiramente menor que RSA (melhorando)
- ❌ Implementações mais complexas
- ❌ Alguns clientes antigos não suportam
- ❌ Patentes em algumas curvas (expiradas agora)

### Quando Usar ECC vs RSA?

**Use ECC quando:**
- ✅ Performance é crítica
- ✅ Trabalhando com dispositivos IoT
- ✅ Quer reduzir latência de TLS
- ✅ Clientes modernos (navegadores atualizados)
- ✅ APIs de alta performance

**Use RSA quando:**
- ✅ Máxima compatibilidade é necessária
- ✅ Trabalhando com sistemas legados
- ✅ Requisitos de compliance especificam RSA
- ✅ Clientes muito antigos devem ser suportados

## PBES (Password-Based Encryption Scheme)

### O que é PBES?

PBES (Password-Based Encryption Scheme) são esquemas padronizados para derivar chaves criptográficas de senhas. Definidos no PKCS #5 (Public-Key Cryptography Standards #5), permitem proteger chaves privadas com senhas.

### Por que PBES é Importante?

Senhas humanas são geralmente fracas. PBES aplica funções matemáticas que:
- Tornam ataques de força bruta mais custosos
- Derivam chaves fortes de senhas fracas
- Protegem contra rainbow tables
- Adicionam "salt" para prevenir ataques pré-computados

### PBES#1

#### Visão Geral

PBES#1 foi a primeira versão do esquema, definida no PKCS #5 v1.5. Usa MD2, MD5 ou SHA-1 como função hash.

#### Como Funciona PBES#1

**Algoritmo:**
```
1. Entrada: senha (P), salt (S), contador de iterações (c)
2. Derivar chave: K = Hash^c(P || S)
   - Hash aplicado c vezes
   - || significa concatenação
3. Criptografar dados com K usando DES ou RC2
```

**Exemplo conceitual:**
```python
import hashlib

def pbes1_derive_key(password, salt, iterations, hash_algo='md5'):
    """
    PBES#1 Key Derivation (simplificado para ilustração)
    
    ATENÇÃO: Esta é uma ilustração conceitual e NÃO representa 
    a implementação completa do PBES#1. O algoritmo real deriva 
    tanto a chave quanto o IV (Initialization Vector) e aplica 
    as iterações de forma diferente. Não use este código em produção.
    """
    # Concatenar senha e salt
    data = password + salt
    
    # Aplicar hash iterativamente
    for i in range(iterations):
        if hash_algo == 'md5':
            data = hashlib.md5(data).digest()
        elif hash_algo == 'sha1':
            data = hashlib.sha1(data).digest()
    
    return data

# Exemplo
password = b"mySecretPassword"
salt = b"\x12\x34\x56\x78\x9a\xbc\xde\xf0"
iterations = 1000

key = pbes1_derive_key(password, salt, iterations, 'md5')
print(f"Chave derivada: {key.hex()}")
```

#### Problemas com PBES#1

**Limitações críticas:**
- ❌ Usa algoritmos hash fracos (MD5, SHA-1)
- ❌ DES como algoritmo de criptografia (56 bits - inseguro)
- ❌ Chave derivada limitada a 64 bits
- ❌ Salt fixo de 8 bytes
- ❌ Vulnerável a ataques modernos

**Status atual:**
- **Depreciado** desde 2000s
- **Não deve ser usado** em novas aplicações
- Mantido apenas para compatibilidade legada

### PBES#2

#### Visão Geral

PBES#2 é a versão moderna e recomendada, definida no PKCS #5 v2.0 e v2.1. Usa PBKDF2 (Password-Based Key Derivation Function 2) para derivação de chaves.

#### Como Funciona PBES#2

**Componentes:**
1. **PBKDF2**: Função de derivação de chave
2. **HMAC**: Hash-based Message Authentication Code
3. **Algoritmo de criptografia**: AES, 3DES, etc.

**Algoritmo PBKDF2:**
```
PBKDF2(P, S, c, dkLen) = T1 || T2 || ... || Tdklen

onde:
P = senha (password)
S = salt (deve ser aleatório)
c = contador de iterações (>= 1000, recomendado >= 100000)
dkLen = comprimento da chave derivada desejada

Ti = F(P, S, c, i)
F(P, S, c, i) = U1 XOR U2 XOR ... XOR Uc

U1 = HMAC(P, S || INT(i))
U2 = HMAC(P, U1)
...
Uc = HMAC(P, Uc-1)
```

**Exemplo prático:**
```python
import hashlib
import hmac
import os

def pbkdf2_hmac_sha256(password, salt, iterations, key_length):
    """
    PBKDF2 com HMAC-SHA256 (implementação simplificada)
    Python tem hashlib.pbkdf2_hmac() nativo
    """
    return hashlib.pbkdf2_hmac('sha256', password, salt, iterations, key_length)

# Exemplo real
password = b"myStrongPassword123!"
salt = os.urandom(16)  # 16 bytes aleatórios
iterations = 100000     # OWASP recomenda >= 100000 para SHA-256
key_length = 32        # 256 bits

derived_key = pbkdf2_hmac_sha256(password, salt, iterations, key_length)

print(f"Salt: {salt.hex()}")
print(f"Iterações: {iterations}")
print(f"Chave derivada: {derived_key.hex()}")
print(f"Comprimento: {len(derived_key)} bytes ({len(derived_key)*8} bits)")
```

**Saída exemplo:**
```
Salt: a1b2c3d4e5f6789012345678abcdef01
Iterações: 100000
Chave derivada: f4a7b3c9d2e8f1a6b4c7d9e2f5a8b1c4d7e0f3a6b9c2d5e8f1a4b7c0d3e6f9
Comprimento: 32 bytes (256 bits)
```

#### Criptografia com PBES#2

Após derivar a chave com PBKDF2, criptografa-se os dados:

**Processo completo:**
```
1. Gerar salt aleatório (>= 16 bytes recomendado)
2. Derivar chave com PBKDF2:
   K = PBKDF2(senha, salt, iterations, keylen)
3. Gerar IV (Initialization Vector) aleatório
4. Criptografar com algoritmo escolhido (ex: AES-256-CBC):
   C = AES-256-CBC(K, IV, plaintext)
5. Armazenar: salt || IV || C
```

**Exemplo completo com OpenSSL:**
```bash
# Criptografar chave privada com PBES#2
openssl pkcs8 -topk8 -v2 aes-256-cbc \
    -v2prf hmacWithSHA256 \
    -iter 100000 \
    -in private_key.pem \
    -out encrypted_private_key.pem

# Verificar algoritmo usado
openssl pkcs8 -in encrypted_private_key.pem -noout -text
# Saída mostra: PBES2, PBKDF2, HMAC-SHA256, AES-256-CBC

# Descriptografar
openssl pkcs8 -in encrypted_private_key.pem -out decrypted_private_key.pem
# Pedirá a senha
```

**Estrutura do arquivo PBES#2:**
```asn1
EncryptedPrivateKeyInfo ::= SEQUENCE {
  encryptionAlgorithm  AlgorithmIdentifier,
  encryptedData        OCTET STRING
}

AlgorithmIdentifier ::= SEQUENCE {
  algorithm   OBJECT IDENTIFIER,  -- PBES2
  parameters  SEQUENCE {
    keyDerivationFunc  SEQUENCE {
      algorithm   OBJECT IDENTIFIER,  -- PBKDF2
      parameters  SEQUENCE {
        salt           OCTET STRING,
        iterationCount INTEGER,
        keyLength      INTEGER OPTIONAL,
        prf            AlgorithmIdentifier DEFAULT hmacWithSHA1
      }
    },
    encryptionScheme  SEQUENCE {
      algorithm   OBJECT IDENTIFIER,  -- AES-256-CBC
      parameters  OCTET STRING         -- IV
    }
  }
}
```

### Parâmetros de Segurança PBES#2

#### Número de Iterações

O número de iterações determina quantas vezes a função hash é aplicada. Mais iterações = mais seguro, mas mais lento.

**Recomendações OWASP (2024):**

| Algoritmo Hash | Iterações Mínimas | Recomendadas |
|----------------|-------------------|--------------|
| PBKDF2-HMAC-SHA1 | 600,000 | 1,300,000 |
| PBKDF2-HMAC-SHA256 | 600,000 | 1,300,000 |
| PBKDF2-HMAC-SHA512 | 210,000 | 600,000 |

**Nota:** Estes valores são baseados no OWASP Password Storage Cheat Sheet (2023-2024). Recomenda-se verificar as diretrizes mais recentes em https://cheatsheetseries.owasp.org/

**Por que tantas iterações?**
- Tornar ataques de força bruta impraticáveis
- Com GPU moderna: bilhões de tentativas/segundo
- Cada iteração adiciona custo computacional ao atacante

**Calibrar iterações para seu sistema:**
```python
import hashlib
import time

def calibrate_iterations(target_time=0.5, max_iterations=10000000):
    """
    Encontra número de iterações para atingir tempo alvo
    target_time: tempo desejado em segundos (0.5s recomendado)
    max_iterations: limite máximo de iterações (padrão 10M)
    """
    password = b"test_password"
    salt = b"test_salt_12345"
    iterations = 10000
    min_elapsed = 0.001  # Tempo mínimo para evitar divisão por valores muito pequenos
    
    max_attempts = 20  # Limitar tentativas para evitar loop infinito
    for attempt in range(max_attempts):
        start = time.time()
        hashlib.pbkdf2_hmac('sha256', password, salt, iterations, 32)
        elapsed = time.time() - start
        
        if elapsed >= target_time:
            return iterations
        
        # Proteger contra elapsed muito pequeno
        if elapsed < min_elapsed:
            elapsed = min_elapsed
        
        # Aumentar iterações proporcionalmente
        new_iterations = int(iterations * (target_time / elapsed))
        
        # Aplicar limite máximo
        iterations = min(new_iterations, max_iterations)
        
        if iterations >= max_iterations:
            print(f"⚠️ Atingiu limite máximo de {max_iterations} iterações sem alcançar tempo alvo de {target_time}s")
            return max_iterations
    
    print(f"⚠️ Atingiu máximo de {max_attempts} tentativas. Usando {iterations} iterações.")
    return iterations

recommended_iterations = calibrate_iterations(0.5)
print(f"Iterações recomendadas: {recommended_iterations}")
```

#### Comprimento do Salt

**Recomendações:**
- **Mínimo**: 16 bytes (128 bits)
- **Recomendado**: 32 bytes (256 bits)
- Deve ser **aleatório** (criptograficamente seguro)
- Deve ser **único** por senha

**Gerar salt seguro:**
```python
import os

# Python
salt = os.urandom(32)  # 32 bytes aleatórios
```

```bash
# Bash/Shell
openssl rand -hex 32  # 32 bytes em hexadecimal
```

#### Função Hash (PRF)

**Opções disponíveis:**
- `hmacWithSHA1`: Legado, menos seguro
- `hmacWithSHA256`: Recomendado, bom equilíbrio
- `hmacWithSHA512`: Mais seguro, mais lento

**Escolher PRF:**
- **Uso geral**: HMAC-SHA256
- **Alta segurança**: HMAC-SHA512
- **Evitar**: HMAC-SHA1 (legado)

### PBES na AWS

#### Problema: Lambda + MSK com mTLS

O problema mencionado no issue ocorre porque:

**Cenário:**
1. Você tem MSK (Kafka) configurado com autenticação mTLS
2. Seu Lambda precisa conectar ao MSK
3. Certificado cliente foi criado com CA usando PBES#2 + ECC
4. Lambda não suporta determinadas combinações de algoritmos

**Por que isso é um problema?**

Lambda tem restrições em:
- Versões OpenSSL específicas
- Suporte a algoritmos modernos
- Formato de chaves privadas

**Limitações do Lambda:**
- Runtime Python 3.x usa OpenSSL 1.0.2 ou 1.1.1 (depende da versão)
- OpenSSL 1.0.2 não suporta PBES#2 com curvas elípticas modernas
- OpenSSL 1.1.1 tem suporte melhor mas ainda limitado

#### Solução 1: Converter para Formato Compatível

**Problema**: Chave privada ECC com PBES#2 não suportada

```bash
# Verificar formato atual
openssl ec -in encrypted_key.pem -noout -text
# Erro: unable to load Private Key

# Descriptografar chave
openssl ec -in encrypted_key.pem -out decrypted_key.pem
# Digite a senha

# Verificar que está descriptografada
openssl ec -in decrypted_key.pem -noout -text
# Deve funcionar

# Re-criptografar com algoritmo suportado (AES-256-CBC simples)
openssl ec -aes256 -in decrypted_key.pem -out compatible_key.pem

# Ou deixar sem criptografia (se seguro no seu contexto)
# Uso no Lambda: chave sem senha
```

#### Solução 2: Usar RSA ao Invés de ECC

Se Lambda não suporta sua configuração ECC:

```bash
# Gerar chave privada RSA
openssl genrsa -out private_key.pem 2048

# Gerar CSR
openssl req -new -key private_key.pem -out csr.pem

# Enviar CSR para sua CA assinar
# Importar certificado resultante para Lambda
```

#### Solução 3: Atualizar Runtime Lambda

```python
# Usar runtime Lambda mais recente
# Python 3.11 ou 3.12 tem OpenSSL mais novo

# layers personalizadas com OpenSSL mais recente
# https://github.com/aws-samples/aws-lambda-layer-openssl
```

#### Solução 4: Armazenar Chave no Secrets Manager

```python
import boto3
import json
import base64

def lambda_handler(event, context):
    # Buscar chave privada do Secrets Manager
    secrets = boto3.client('secretsmanager')
    response = secrets.get_secret_value(SecretId='msk/client-key')
    
    # Chave armazenada como string
    private_key = response['SecretString']
    
    # Usar para conexão MSK
    # ...
```

**Armazenar no Secrets Manager:**
```bash
# Descriptografar chave localmente
openssl ec -in encrypted_key.pem -out decrypted_key.pem

# Armazenar no Secrets Manager
aws secretsmanager create-secret \
    --name msk/client-key \
    --secret-string file://decrypted_key.pem

# Dar permissão ao Lambda para acessar Secrets Manager
# Primeiro, obtenha o role do Lambda
LAMBDA_ROLE=$(aws lambda get-function-configuration \
    --function-name my-function \
    --query 'Role' --output text | sed 's:.*/::')

# Anexe política ao role do Lambda
aws iam attach-role-policy \
    --role-name $LAMBDA_ROLE \
    --policy-arn arn:aws:iam::aws:policy/SecretsManagerReadWrite

# Ou para permissão mais restrita, crie política customizada:
cat > /tmp/lambda-secrets-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "secretsmanager:GetSecretValue",
      "Resource": "arn:aws:secretsmanager:*:*:secret:msk/client-key*"
    }
  ]
}
EOF

aws iam put-role-policy \
    --role-name $LAMBDA_ROLE \
    --policy-name LambdaSecretsManagerAccess \
    --policy-document file:///tmp/lambda-secrets-policy.json
```

### Comparação: PBES#1 vs PBES#2

| Aspecto | PBES#1 | PBES#2 |
|---------|--------|--------|
| **Algoritmo Hash** | MD2, MD5, SHA-1 | Qualquer (SHA-256, SHA-512) |
| **Criptografia** | DES, RC2 | AES, 3DES, etc. |
| **Comprimento Chave** | Limitado pelo algoritmo (DES: 56 bits, RC2: variável) | Flexível (128-256 bits) |
| **Salt** | 8 bytes fixos | Flexível (16+ bytes) |
| **Iterações** | Tipicamente baixo | Alto (100k+) |
| **Segurança** | ❌ Inseguro | ✅ Seguro |
| **Status** | Depreciado | Recomendado |
| **Uso** | Legado apenas | Produção |

**Recomendação clara:**
- ✅ **Use PBES#2** para todas as novas aplicações
- ❌ **Evite PBES#1** completamente
- 🔄 **Migre** sistemas legados para PBES#2

## Outros Algoritmos Relevantes

### AES (Advanced Encryption Standard)

**O que é:**
- Algoritmo de criptografia simétrica
- Padrão desde 2001 (substituiu DES)
- Usado em KMS, S3, EBS, RDS

**Tamanhos de chave:**
- AES-128: 128 bits (adequado)
- AES-192: 192 bits (forte)
- AES-256: 256 bits (muito forte)

**Modos de operação:**
- **CBC** (Cipher Block Chaining): Tradicional, requer IV
- **GCM** (Galois/Counter Mode): Autenticado, recomendado
- **CTR** (Counter): Paralelizável, rápido

**AWS KMS usa:**
- AES-256-GCM para criptografia de data keys
- Envelopes de criptografia

### SHA (Secure Hash Algorithm)

**Família de funções hash:**

| Algoritmo | Output | Segurança | Status |
|-----------|--------|-----------|--------|
| SHA-1 | 160 bits | Fraco | ❌ Depreciado |
| SHA-224 | 224 bits | Adequado | ⚠️ Raro |
| SHA-256 | 256 bits | Forte | ✅ Recomendado |
| SHA-384 | 384 bits | Muito forte | ✅ Alta segurança |
| SHA-512 | 512 bits | Muito forte | ✅ Alta segurança |
| SHA-3 | Variável | Forte | ✅ Moderno |

**Uso na AWS:**
```bash
# ACM Certificate Signature Algorithm
aws acm describe-certificate --certificate-arn "$ARN" \
    --query 'Certificate.SignatureAlgorithm'
# Saída: SHA256WITHRSA ou SHA256WITHECDSA
```

### HMAC (Hash-based Message Authentication Code)

**O que é:**
- Código de autenticação de mensagem
- Usa função hash + chave secreta
- Garante integridade e autenticidade

**Fórmula:**
```
HMAC(K, m) = H((K' ⊕ opad) || H((K' ⊕ ipad) || m))

onde:
K = chave secreta
m = mensagem
H = função hash (SHA-256, etc.)
opad = 0x5c5c5c... (outer padding)
ipad = 0x363636... (inner padding)
```

**Exemplo prático:**
```python
import hmac
import hashlib

key = b"secret_key_12345"
message = b"Hello, World!"

# HMAC-SHA256
signature = hmac.new(key, message, hashlib.sha256).digest()
print(f"HMAC: {signature.hex()}")

# Verificar
def verify_hmac(key, message, signature):
    """
    Verifica assinatura HMAC de forma segura contra timing attacks.
    
    Args:
        key: chave secreta (bytes)
        message: mensagem original (bytes)
        signature: assinatura a verificar (bytes)
    
    Returns:
        bool: True se assinatura é válida
    """
    if not isinstance(signature, bytes):
        raise TypeError("Signature must be bytes")
    
    expected = hmac.new(key, message, hashlib.sha256).digest()
    return hmac.compare_digest(expected, signature)

is_valid = verify_hmac(key, message, signature)
print(f"Valid: {is_valid}")
```

**Uso na AWS:**
- Assinatura de requisições API (SigV4)
- PBKDF2 usa HMAC internamente
- S3 pre-signed URLs

## Compatibilidade de Algoritmos em Serviços AWS

### Application Load Balancer (ALB)

**Políticas de segurança TLS:**

| Política | TLS | Cipher Suites | Recomendação |
|----------|-----|---------------|--------------|
| ELBSecurityPolicy-2016-08 | 1.0-1.2 | RSA, ECDHE | ⚠️ Legado |
| ELBSecurityPolicy-TLS-1-2-2017-01 | 1.2 | RSA, ECDHE | ✅ Mínimo |
| ELBSecurityPolicy-TLS-1-2-Ext-2018-06 | 1.2 | RSA, ECDHE | ✅ Recomendado |
| ELBSecurityPolicy-FS-1-2-2019-08 | 1.2 | ECDHE apenas | ✅ Forward Secrecy |
| ELBSecurityPolicy-TLS13-1-2-2021-06 | 1.2-1.3 | TLS 1.3 | ✅ Moderno |

**Exemplo:**
```bash
aws elbv2 create-listener \
    --load-balancer-arn $ALB_ARN \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=$CERT_ARN \
    --ssl-policy ELBSecurityPolicy-TLS-1-2-Ext-2018-06
```

### CloudFront

**Versões TLS suportadas:**

| Versão | Segurança | Suporte |
|--------|-----------|---------|
| TLSv1 | Fraco | ❌ Não usar |
| TLSv1.1 | Fraco | ❌ Não usar |
| TLSv1.2_2021 | Forte | ✅ Recomendado |
| TLSv1.3 | Muito forte | ✅ Moderno |

**Configuração (exemplo parcial - ViewerCertificate apenas):**
```bash
# Nota: Este é um exemplo parcial mostrando apenas a configuração de certificado.
# Uma distribuição CloudFront completa requer Origins, DefaultCacheBehavior, 
# CallerReference e outros campos obrigatórios.

aws cloudfront create-distribution \
    --distribution-config '{
        "CallerReference": "my-dist-'$(date +%s)'",
        "Comment": "Example distribution",
        "Enabled": true,
        "Origins": {
            "Quantity": 1,
            "Items": [{
                "Id": "my-origin",
                "DomainName": "example.com",
                "CustomOriginConfig": {
                    "HTTPPort": 80,
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only"
                }
            }]
        },
        "DefaultCacheBehavior": {
            "TargetOriginId": "my-origin",
            "ViewerProtocolPolicy": "redirect-to-https",
            "TrustedSigners": {"Enabled": false, "Quantity": 0},
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
        }
    }'
```

### API Gateway

**Suporta:**
- TLS 1.2 (mínimo)
- Certificados ACM (RSA e ECC)
- mTLS (mutual TLS)

**Exemplo mTLS:**
```bash
# Criar domain name com mTLS
aws apigateway create-domain-name \
    --domain-name api.example.com \
    --regional-certificate-arn $CERT_ARN \
    --endpoint-configuration types=REGIONAL \
    --mutual-tls-authentication truststoreUri=s3://my-bucket/truststore.pem
```

### Lambda

**Limitações conhecidas:**
- Runtime-dependent (Python, Node.js, Java, etc.)
- OpenSSL version varies by runtime
- Algumas combinações PBES#2 + ECC não suportadas

**Runtime OpenSSL versions:**

| Runtime | OpenSSL | PBES#2 ECC | Recomendação |
|---------|---------|------------|--------------|
| Python 3.8 | 1.1.1d | ⚠️ Limitado | Atualizar |
| Python 3.9 | 1.1.1d | ⚠️ Limitado | Atualizar |
| Python 3.10 | 1.1.1n | ⚠️ Parcial | Atualizar |
| Python 3.11 | 1.1.1t | ✅ Suportado | ✅ Use |
| Python 3.12 | 3.0.8 | ✅ Completo | ✅ Melhor |

**Nota:** As versões do OpenSSL podem variar conforme atualizações da AWS. Sempre consulte a [documentação oficial dos runtimes Lambda](https://docs.aws.amazon.com/lambda/latest/dg/lambda-runtimes.html) para informações atualizadas.

**Workaround para runtimes antigos:**
```bash
# Usar Layer com OpenSSL mais recente
# Ou descriptografar chaves antes de usar no Lambda
```

### MSK (Managed Streaming for Kafka)

**Autenticação suportada:**
- IAM
- SASL/SCRAM
- **mTLS** (mutual TLS)

**mTLS requirements:**
- Certificado cliente em formato PEM
- Chave privada sem senha (armazenada de forma segura, ex: AWS Secrets Manager)
- CA raiz confiável

**Problema comum:**
```bash
# Erro: Lambda não consegue ler chave privada
# Causa: PBES#2 com ECC não suportado pelo runtime

# Solução:
# 1. Descriptografar chave
openssl ec -in encrypted.pem -out decrypted.pem

# 2. Ou usar RSA
openssl genrsa -out key.pem 2048

# 3. Armazenar em Secrets Manager
aws secretsmanager create-secret --name msk/key --secret-string file://decrypted.pem
```

## Melhores Práticas

### 1. Escolha de Algoritmo

**Para Certificados SSL/TLS:**
```
✅ Primeira escolha: ECC P-256 (EC_prime256v1)
   - Melhor performance
   - Segurança forte
   - Suporte moderno

✅ Alternativa: RSA 2048
   - Máxima compatibilidade
   - Amplamente suportado
   - Confiável

⚠️ Alta segurança: ECC P-384 ou RSA 4096
   - Compliance rigoroso
   - Performance impactada
   - Overkill para maioria dos casos
```

**Para Proteção de Chaves Privadas:**
```
✅ Use: PBES#2 com PBKDF2-HMAC-SHA256
   - >= 100.000 iterações
   - Salt >= 16 bytes
   - AES-256-CBC ou AES-256-GCM

❌ Evite: PBES#1
   - Inseguro
   - Depreciado
   - Apenas para legado
```

### 2. Compatibilidade

**Verificar antes de implementar:**

```bash
# Verificar algoritmo de certificado
openssl x509 -in cert.pem -text -noout | grep "Public Key Algorithm"

# Verificar algoritmo de chave privada
openssl rsa -in key.pem -text -noout
# ou
openssl ec -in key.pem -text -noout

# Verificar se chave está criptografada
openssl rsa -in key.pem -noout -text
# Se pedir senha, está criptografada

# Ver detalhes de PBES
openssl pkcs8 -in encrypted_key.pem -noout -text
```

**Matriz de compatibilidade:**

| Serviço | RSA 2048 | RSA 4096 | ECC P-256 | ECC P-384 | PBES#1 | PBES#2 |
|---------|----------|----------|-----------|-----------|--------|--------|
| ACM | ✅ | ✅ | ✅ | ✅ | ⚠️ | ✅ |
| ALB | ✅ | ✅ | ✅ | ✅ | N/A | N/A |
| CloudFront | ✅ | ✅ | ✅ | ✅ | N/A | N/A |
| API Gateway | ✅ | ✅ | ✅ | ✅ | N/A | N/A |
| Lambda (3.11+) | ✅ | ✅ | ✅ | ✅ | ❌ | ⚠️ |
| Lambda (3.8-3.10) | ✅ | ✅ | ⚠️ | ⚠️ | ❌ | ❌ |
| MSK | ✅ | ✅ | ✅ | ✅ | N/A | N/A |

### 3. Segurança

**Hierarquia de segurança:**

```
1. Algoritmo Criptográfico
   ├─ ✅ AES-256 > AES-128 > 3DES > ❌ DES
   └─ ✅ SHA-256 > SHA-1 > ❌ MD5

2. Tamanho de Chave
   ├─ ECC: ✅ P-384 > P-256 > ❌ P-192
   └─ RSA: ✅ 4096 > 2048 > ❌ 1024

3. Derivação de Chave
   ├─ ✅ PBKDF2 (PBES#2) > ❌ MD5 (PBES#1)
   └─ Iterações: ✅ 100000+ > 10000 > ❌ 1000

4. Protocolo TLS
   └─ ✅ TLS 1.3 > TLS 1.2 > ❌ TLS 1.1 > ❌ TLS 1.0
```

**Configuração recomendada em 2024:**

```yaml
Certificado:
  Algoritmo: EC_prime256v1  # ou RSA_2048
  Assinatura: SHA256

Chave Privada Criptografada:
  Esquema: PBES#2
  KDF: PBKDF2-HMAC-SHA256
  Iterações: 100000+
  Salt: 32 bytes aleatórios
  Criptografia: AES-256-CBC

TLS:
  Versão Mínima: TLS 1.2
  Cipher Suites: Forward Secrecy (ECDHE)
  
Lambda:
  Runtime: Python 3.11+ ou 3.12
  Chaves: Sem senha ou em Secrets Manager
```

### 4. Rotação de Chaves

**Frequência recomendada:**

| Tipo | Rotação | Motivo |
|------|---------|--------|
| Certificados SSL | 90 dias (ACM automático) | Limite risco de comprometimento |
| Chaves KMS | Anual (automático) | Compliance |
| Access Keys IAM | 90 dias | Best practice AWS |
| Senhas | 90 dias | Requisito comum |

**Automação:**
```bash
# ACM renova automaticamente certificados emitidos
# Para certificados importados, monitore a expiração

# Nota: A métrica DaysToExpiry do ACM requer configuração específica.
# Verifique a documentação oficial da AWS para métricas de ACM disponíveis:
# https://docs.aws.amazon.com/acm/latest/userguide/cloudwatch-metrics.html

# Exemplo de alarme (verifique disponibilidade da métrica na sua região):
aws cloudwatch put-metric-alarm \
    --alarm-name cert-expiring \
    --metric-name DaysToExpiry \
    --namespace AWS/CertificateManager \
    --statistic Minimum \
    --period 86400 \
    --threshold 30 \
    --comparison-operator LessThanThreshold \
    --evaluation-periods 1

# Alternativa: Use EventBridge para monitorar eventos ACM
# ou crie função Lambda para verificar expiração periodicamente
```

### 5. Monitoramento

**Métricas importantes:**

```python
import boto3
from datetime import datetime, timezone

def audit_certificates():
    """Auditar certificados ACM"""
    acm = boto3.client('acm')
    
    # Listar todos os certificados
    response = acm.list_certificates(CertificateStatuses=['ISSUED'])
    
    warnings = []
    for cert in response['CertificateSummaryList']:
        details = acm.describe_certificate(
            CertificateArn=cert['CertificateArn']
        )['Certificate']
        
        # Verificar algoritmo
        key_algo = details['KeyAlgorithm']
        if key_algo == 'RSA_1024':
            warnings.append(f"⚠️ {details['DomainName']}: RSA 1024 inseguro")
        
        # Verificar expiração
        # ACM retorna datetime timezone-aware, então usamos datetime.now(timezone.utc)
        not_after = details['NotAfter']
        now = datetime.now(timezone.utc)
        days_left = (not_after - now).days
        if days_left < 30:
            warnings.append(f"⚠️ {details['DomainName']}: Expira em {days_left} dias")
        
        # Verificar renovação
        renewal = details.get('RenewalEligibility', 'INELIGIBLE')
        if renewal == 'INELIGIBLE':
            warnings.append(f"⚠️ {details['DomainName']}: Renovação não elegível")
    
    return warnings

# Executar auditoria
issues = audit_certificates()
for issue in issues:
    print(issue)
```

## Troubleshooting

### Problema 1: Lambda não consegue ler chave privada

**Erro:**
```
[ERROR] Unable to load private key
Could not deserialize key data
```

**Causa:**
- Chave criptografada com PBES#2 + ECC
- Runtime Lambda com OpenSSL antigo

**Solução:**
```bash
# Opção 1: Descriptografar
openssl ec -in encrypted.pem -out decrypted.pem

# Opção 2: Converter para RSA
openssl genrsa -out rsa_key.pem 2048
# Re-gerar certificado com nova chave

# Opção 3: Atualizar runtime
# Usar Python 3.11 ou 3.12
```

### Problema 2: ACM rejeita certificado importado

**Erro:**
```
InvalidParameterException: The private key is not valid
```

**Causa:**
- Chave privada com senha
- Formato incorreto
- Chave não corresponde ao certificado

**Solução:**
```bash
# Remover senha
openssl rsa -in encrypted.key -out decrypted.key
# ou para ECC:
openssl ec -in encrypted.key -out decrypted.key

# Verificar correspondência
openssl x509 -noout -modulus -in cert.pem | openssl md5
openssl rsa -noout -modulus -in key.pem | openssl md5
# MD5 deve ser igual

# Verificar formato PEM
head -n 1 cert.pem
# Deve ser: -----BEGIN CERTIFICATE-----
head -n 1 key.pem
# Deve ser: -----BEGIN RSA PRIVATE KEY----- ou -----BEGIN EC PRIVATE KEY-----
```

### Problema 3: MSK mTLS falha

**Erro:**
```
SSL handshake failed
```

**Verificações:**
```bash
# 1. Verificar certificado cliente
openssl x509 -in client-cert.pem -text -noout

# 2. Verificar cadeia de certificados
openssl verify -CAfile ca-cert.pem client-cert.pem

# 3. Testar conexão TLS
openssl s_client -connect kafka-broker:9094 \
    -cert client-cert.pem \
    -key client-key.pem \
    -CAfile ca-cert.pem

# 4. Verificar chave privada não tem senha
openssl rsa -in client-key.pem -noout -text
# Não deve pedir senha
```

### Problema 4: Algoritmo não suportado

**Erro:**
```
UnsupportedAlgorithmException: Algorithm not supported
```

**Diagnóstico:**
```bash
# Identificar algoritmo
openssl x509 -in cert.pem -text -noout | grep "Signature Algorithm"
openssl x509 -in cert.pem -text -noout | grep "Public Key Algorithm"

# Listar algoritmos suportados
openssl list -cipher-algorithms
openssl list -public-key-algorithms
```

**Solução:**
- Verificar matriz de compatibilidade acima
- Gerar novo certificado com algoritmo suportado
- Atualizar runtime/biblioteca

## Recursos de Aprendizado

### Documentação Oficial

- [ACM Key Algorithms](https://docs.aws.amazon.com/acm/latest/userguide/acm-certificate.html#algorithms)
- [AWS Cryptographic Services](https://docs.aws.amazon.com/crypto/latest/userguide/)
- [NIST Cryptographic Standards](https://csrc.nist.gov/projects/cryptographic-standards-and-guidelines)

### RFCs e Padrões

- [RFC 8017 - PKCS #1: RSA Cryptography](https://tools.ietf.org/html/rfc8017)
- [RFC 5208 - PKCS #8: Private-Key Information Syntax](https://tools.ietf.org/html/rfc5208)
- [RFC 8018 - PKCS #5: Password-Based Cryptography v2.1](https://tools.ietf.org/html/rfc8018)
- [RFC 5915 - Elliptic Curve Private Key Structure](https://tools.ietf.org/html/rfc5915)
- [RFC 6090 - Fundamental Elliptic Curve Cryptography Algorithms](https://tools.ietf.org/html/rfc6090)

### Ferramentas

- [OpenSSL Documentation](https://www.openssl.org/docs/)
- [ACM Certificate Checker](https://www.ssllabs.com/ssltest/)
- [Cipher Suite Info](https://ciphersuite.info/)

### Livros Recomendados

- **"Cryptography Engineering"** - Ferguson, Schneier, Kohno
- **"Serious Cryptography"** - Aumasson
- **"Applied Cryptography"** - Schneier

### Cursos Online

- [Coursera - Cryptography I](https://www.coursera.org/learn/crypto)
- [AWS Training - Security Engineering](https://aws.amazon.com/training/learn-about/security/)

## Conclusão

A escolha correta de algoritmos criptográficos é crucial para a segurança de aplicações na AWS. Principais takeaways:

### Recomendações Finais

**Certificados SSL/TLS:**
- ✅ Use **ECC P-256** para melhor performance
- ✅ Use **RSA 2048** para máxima compatibilidade
- ❌ Evite RSA 1024 e algoritmos depreciados

**Proteção de Chaves:**
- ✅ Use **PBES#2** com PBKDF2-HMAC-SHA256
- ✅ >= 100.000 iterações
- ❌ Nunca use PBES#1

**Lambda + MSK mTLS:**
- ✅ Use runtime Python 3.11+ ou 3.12
- ✅ Descriptografe chaves antes de usar
- ✅ Armazene chaves no Secrets Manager
- ✅ Teste compatibilidade antes de deploy

**Segurança Geral:**
- 🔄 Rotacione chaves regularmente
- 👁️ Monitore certificados próximos da expiração
- 📚 Mantenha-se atualizado com melhores práticas
- 🧪 Teste em ambiente de desenvolvimento primeiro

### Próximos Passos

1. Audite seus certificados atuais
2. Identifique algoritmos fracos ou depreciados
3. Planeje migração para algoritmos modernos
4. Implemente monitoramento e alertas
5. Documente suas decisões de segurança

A criptografia é um campo em constante evolução. Mantenha-se informado sobre novas recomendações e vulnerabilidades descobertas.
