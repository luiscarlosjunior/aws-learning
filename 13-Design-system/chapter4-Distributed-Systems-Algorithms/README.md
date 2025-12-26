# Capítulo 4: Algoritmos de Sistemas Distribuídos

Os algoritmos de sistemas distribuídos formam a espinha dorsal dos sistemas modernos de computação em larga escala. Estes algoritmos resolvem problemas fundamentais que surgem quando múltiplos processos independentes precisam coordenar suas ações, manter consistência de dados, tolerar falhas e alcançar objetivos comuns sem depender de uma autoridade centralizada. Como observa Leslie Lamport, pioneiro nesta área: "A distributed system is one in which the failure of a computer you didn't even know existed can render your own computer unusable" ("Um sistema distribuído é aquele no qual a falha de um computador que você nem sabia que existia pode tornar seu próprio computador inutilizável").

Este capítulo explora em profundidade os algoritmos fundamentais e avançados que possibilitam a construção de sistemas distribuídos robustos, escaláveis e confiáveis, com ênfase em suas implementações práticas na AWS e em sistemas de produção modernos.

---

## 1. Relógios, Ordenação de Eventos e Causalidade

### 1.1 Fundamentos Teóricos

Em sistemas distribuídos, a ausência de um relógio global sincronizado torna a ordenação de eventos um desafio fundamental. Diferentemente de sistemas centralizados, onde timestamps do sistema operacional fornecem uma ordem total de eventos, sistemas distribuídos requerem mecanismos sofisticados para estabelecer relações de causalidade entre eventos.

#### 1.1.1 O Problema do Tempo em Sistemas Distribuídos

**Por que timestamps físicos são insuficientes?**

```python
# Problema: Relógios físicos dessincronizados
class PhysicalClockProblem:
    """
    Demonstração do problema de dessincronização
    """
    def scenario_clock_skew(self):
        """
        Cenário: Dois servers com relógios diferentes
        """
        # Server A (relógio está 500ms adiantado)
        event_a_timestamp = time.time() + 0.5  # 1705334400.5
        message = {
            'event': 'user_login',
            'user_id': 'user123',
            'timestamp': event_a_timestamp
        }
        server_a.send_message(message, to='server_b')
        
        # Server B (relógio está correto)
        event_b_timestamp = time.time()  # 1705334400.0
        
        # Server B recebe mensagem de A
        received_message = server_b.receive_message()
        
        # Problema: Timestamp de A é "do futuro" relativo a B!
        if received_message['timestamp'] > event_b_timestamp:
            # Parece que mensagem veio do futuro
            # Ordenação de eventos está incorreta!
            print("Received message from the future!")
        
        # Consequências:
        # - Não podemos determinar ordem real dos eventos
        # - Logs ficam desordenados
        # - Debugging torna-se impossível
        # - Transações podem ser ordenadas incorretamente
```

**Fatores que causam dessincronização:**
- **Clock drift**: Osciladores de cristal variam (±50ppm típico)
- **Temperature effects**: Temperatura afeta frequência do cristal
- **Network delays**: Latência variável na sincronização (NTP)
- **Relatividade**: Em sistemas de alta precisão, até efeitos relativísticos importam

**Referência**: Lamport, L. (1978). "Time, Clocks, and the Ordering of Events in a Distributed System". *Communications of the ACM*, 21(7), 558-565.

#### 1.1.2 Relógios Lógicos de Lamport

Leslie Lamport introduziu o conceito de relógios lógicos que capturam relações de causalidade sem depender de sincronização de relógios físicos.

**Definição Formal:**

Para cada processo P_i, mantemos um contador lógico LC_i que segue estas regras:

1. **Regra R1 (Evento Local)**: Antes de executar um evento, P_i incrementa LC_i
2. **Regra R2 (Envio de Mensagem)**: Quando P_i envia mensagem m, anexa timestamp LC_i(m) = LC_i
3. **Regra R3 (Recebimento de Mensagem)**: Quando P_j recebe m com timestamp LC_i(m), atualiza: LC_j = max(LC_j, LC_i(m)) + 1

**Propriedade Fundamental**:
Se evento a → evento b (a happens-before b), então LC(a) < LC(b)

**Implementação Completa:**

```python
class LamportClock:
    """
    Implementação de Relógio Lógico de Lamport
    """
    def __init__(self, process_id):
        self.process_id = process_id
        self.counter = 0
        self.lock = threading.Lock()
    
    def tick(self):
        """
        Incrementa relógio em evento local (Regra R1)
        """
        with self.lock:
            self.counter += 1
            return self.counter
    
    def send_timestamp(self):
        """
        Retorna timestamp para anexar em mensagem (Regra R2)
        """
        with self.lock:
            self.counter += 1
            timestamp = self.counter
            logger.debug(f"[P{self.process_id}] Send with timestamp {timestamp}")
            return timestamp
    
    def receive_timestamp(self, received_timestamp):
        """
        Atualiza relógio ao receber mensagem (Regra R3)
        """
        with self.lock:
            self.counter = max(self.counter, received_timestamp) + 1
            logger.debug(
                f"[P{self.process_id}] Received timestamp {received_timestamp}, "
                f"updated to {self.counter}"
            )
            return self.counter


# Exemplo de uso prático
class DistributedBankingSystem:
    """
    Sistema bancário com ordenação de eventos
    """
    def __init__(self, process_id):
        self.clock = LamportClock(process_id)
        self.event_log = []
    
    def transfer_money(self, from_account, to_account, amount):
        """
        Transferência com timestamp lógico
        """
        # Evento local: iniciar transferência
        timestamp = self.clock.tick()
        
        # Log evento com timestamp
        event = {
            'type': 'TRANSFER_INITIATED',
            'from': from_account,
            'to': to_account,
            'amount': amount,
            'timestamp': timestamp,
            'process': self.clock.process_id
        }
        self.event_log.append(event)
        
        # Enviar mensagem para outro processo
        message = {
            'event': event,
            'logical_timestamp': self.clock.send_timestamp()
        }
        
        self.send_to_replica(message)
        
        return timestamp
    
    def receive_transfer(self, message):
        """
        Receber notificação de transferência
        """
        # Atualizar relógio lógico
        self.clock.receive_timestamp(message['logical_timestamp'])
        
        # Processar evento
        event = message['event']
        local_timestamp = self.clock.tick()
        
        # Log evento recebido
        received_event = {
            'type': 'TRANSFER_RECEIVED',
            'original_event': event,
            'received_at': local_timestamp,
            'process': self.clock.process_id
        }
        self.event_log.append(received_event)
        
        # Eventos agora estão ordenados corretamente por timestamp lógico
        self.event_log.sort(key=lambda e: e.get('timestamp', e.get('received_at')))
```

**Exemplo de Execução:**

```python
# Cenário: Três processos executando transferências concorrentes
"""
Processo A:
  e1: transfer(account1, account2, $100)  LC_A = 1
  e2: send_message(to=B)                   LC_A = 2
  e3: local_operation()                    LC_A = 3

Processo B:
  e4: receive(from=A, LC=2)                LC_B = max(0, 2) + 1 = 3
  e5: transfer(account2, account3, $50)    LC_B = 4
  e6: send_message(to=C)                   LC_B = 5

Processo C:
  e7: local_operation()                    LC_C = 1
  e8: receive(from=B, LC=5)                LC_C = max(1, 5) + 1 = 6
  e9: transfer(account3, account1, $25)    LC_C = 7

Ordem total resultante: e1 < e2 < e3 = e4 < e5 < e6 < e7 < e8 < e9

Isso captura corretamente a causalidade:
- e1 aconteceu antes de e4 (mensagem enviada)
- e5 aconteceu depois de e4 (porque recebeu mensagem de A)
- e8 aconteceu depois de e6 (mensagem enviada de B para C)
"""
```

**Limitações dos Relógios de Lamport:**

O inverso não é verdade: LC(a) < LC(b) não implica necessariamente que a → b. Eventos podem ser concorrentes.

```python
def demonstrate_lamport_limitation():
    """
    Eventos concorrentes podem ter timestamps diferentes
    mas não são causalmente relacionados
    """
    # Processo A
    clock_a = LamportClock('A')
    timestamp_a = clock_a.tick()  # timestamp_a = 1
    
    # Processo B (executando concorrentemente, sem comunicação com A)
    clock_b = LamportClock('B')
    timestamp_b1 = clock_b.tick()  # timestamp_b1 = 1
    timestamp_b2 = clock_b.tick()  # timestamp_b2 = 2
    
    # Análise:
    # timestamp_a (1) < timestamp_b2 (2)
    # MAS eventos não são causalmente relacionados!
    # São eventos CONCORRENTES (concurrent)
    
    # Para detectar concorrência, precisamos de Vector Clocks
```

#### 1.1.3 Vector Clocks (Relógios Vetoriais)

Vector Clocks estendem Lamport Clocks para capturar completamente relações de causalidade e detectar concorrência.

**Definição Formal:**

Para sistema com N processos, cada processo P_i mantém vetor V_i de N elementos:
- V_i[i] = número de eventos executados por P_i
- V_i[j] = conhecimento de P_i sobre eventos de P_j

**Regras de Atualização:**

1. **Evento Local**: Antes de executar evento, P_i incrementa V_i[i]
2. **Envio**: P_i envia mensagem com V_i anexado
3. **Recebimento**: P_j recebe mensagem de P_i com vetor V_msg:
   - Para cada k: V_j[k] = max(V_j[k], V_msg[k])
   - Incrementa V_j[j]

**Comparação de Vetores:**

```python
def compare_vector_clocks(v1, v2):
    """
    Compara dois vector clocks
    Retorna: 'HAPPENS_BEFORE', 'HAPPENS_AFTER', 'CONCURRENT'
    """
    # v1 <= v2 se para todo i: v1[i] <= v2[i]
    v1_less_eq = all(v1[i] <= v2[i] for i in range(len(v1)))
    v2_less_eq = all(v2[i] <= v1[i] for i in range(len(v2)))
    
    # v1 < v2 se v1 <= v2 e v1 != v2
    v1_less = v1_less_eq and (v1 != v2)
    v2_less = v2_less_eq and (v1 != v2)
    
    if v1_less:
        return 'HAPPENS_BEFORE'  # v1 → v2
    elif v2_less:
        return 'HAPPENS_AFTER'   # v2 → v1
    else:
        return 'CONCURRENT'       # v1 || v2 (concurrent)
```

**Implementação Completa:**

```python
import copy

class VectorClock:
    """
    Implementação de Vector Clock
    """
    def __init__(self, process_id, num_processes):
        self.process_id = process_id
        self.num_processes = num_processes
        # Inicializa vetor com zeros
        self.vector = [0] * num_processes
        self.lock = threading.Lock()
    
    def tick(self):
        """
        Incrementa contador local em evento
        """
        with self.lock:
            self.vector[self.process_id] += 1
            return copy.deepcopy(self.vector)
    
    def get_timestamp(self):
        """
        Retorna cópia do vetor atual para enviar
        """
        with self.lock:
            self.vector[self.process_id] += 1
            return copy.deepcopy(self.vector)
    
    def update(self, received_vector):
        """
        Atualiza vetor ao receber mensagem
        """
        with self.lock:
            # Merge: max component-wise
            for i in range(self.num_processes):
                self.vector[i] = max(self.vector[i], received_vector[i])
            
            # Incrementa contador local
            self.vector[self.process_id] += 1
            
            return copy.deepcopy(self.vector)
    
    def happens_before(self, other_vector):
        """
        Verifica se self aconteceu antes de other_vector
        """
        with self.lock:
            # self < other se:
            # 1. Para todo i: self[i] <= other[i]
            # 2. Existe j tal que: self[j] < other[j]
            all_less_eq = all(
                self.vector[i] <= other_vector[i]
                for i in range(self.num_processes)
            )
            some_strictly_less = any(
                self.vector[i] < other_vector[i]
                for i in range(self.num_processes)
            )
            return all_less_eq and some_strictly_less
    
    def is_concurrent(self, other_vector):
        """
        Verifica se eventos são concorrentes
        """
        with self.lock:
            return (
                not self.happens_before(other_vector) and
                not VectorClock._static_happens_before(other_vector, self.vector)
            )
    
    @staticmethod
    def _static_happens_before(v1, v2):
        """
        Versão estática de happens_before
        """
        all_less_eq = all(v1[i] <= v2[i] for i in range(len(v1)))
        some_strictly_less = any(v1[i] < v2[i] for i in range(len(v1)))
        return all_less_eq and some_strictly_less
    
    def __repr__(self):
        return f"VC[P{self.process_id}]: {self.vector}"


# Aplicação Prática: Sistema de Versionamento Distribuído
class DistributedVersionControl:
    """
    Sistema similar ao Git usando Vector Clocks
    """
    def __init__(self, replica_id, num_replicas):
        self.replica_id = replica_id
        self.clock = VectorClock(replica_id, num_replicas)
        self.documents = {}  # doc_id -> versões
    
    def create_document(self, doc_id, content):
        """
        Cria novo documento com versão inicial
        """
        version_vector = self.clock.tick()
        
        document = {
            'doc_id': doc_id,
            'content': content,
            'version': version_vector,
            'created_by': self.replica_id,
            'history': []
        }
        
        self.documents[doc_id] = [document]
        
        return version_vector
    
    def update_document(self, doc_id, new_content, base_version):
        """
        Atualiza documento criando nova versão
        """
        if doc_id not in self.documents:
            raise ValueError(f"Document {doc_id} not found")
        
        # Gera novo vetor de versão
        new_version = self.clock.tick()
        
        # Cria nova versão do documento
        new_doc = {
            'doc_id': doc_id,
            'content': new_content,
            'version': new_version,
            'base_version': base_version,
            'updated_by': self.replica_id
        }
        
        self.documents[doc_id].append(new_doc)
        
        return new_version
    
    def merge_versions(self, doc_id, remote_versions):
        """
        Merge de versões de diferentes réplicas
        """
        local_versions = self.documents.get(doc_id, [])
        
        conflicts = []
        
        for remote_version in remote_versions:
            remote_vec = remote_version['version']
            
            # Verifica conflitos
            is_concurrent = False
            for local_version in local_versions:
                local_vec = local_version['version']
                
                if VectorClock._static_happens_before(remote_vec, local_vec):
                    # Remote é anterior, ignora
                    break
                elif VectorClock._static_happens_before(local_vec, remote_vec):
                    # Local é anterior, aceita remote
                    self.clock.update(remote_vec)
                    self.documents[doc_id].append(remote_version)
                    break
                else:
                    # Concorrentes! Conflito detectado
                    is_concurrent = True
                    conflicts.append({
                        'local': local_version,
                        'remote': remote_version,
                        'reason': 'concurrent_modifications'
                    })
        
        return {
            'conflicts': conflicts,
            'merged': len(conflicts) == 0
        }


# Exemplo de execução com conflitos
def demonstrate_version_conflict():
    """
    Demonstra detecção de conflitos com Vector Clocks
    """
    # Três réplicas
    replica_a = DistributedVersionControl(replica_id=0, num_replicas=3)
    replica_b = DistributedVersionControl(replica_id=1, num_replicas=3)
    replica_c = DistributedVersionControl(replica_id=2, num_replicas=3)
    
    # Replica A cria documento
    v1 = replica_a.create_document('doc1', 'Initial content')
    print(f"Version 1: {v1}")  # [1, 0, 0]
    
    # Sincroniza para B e C
    replica_b.clock.update(v1)
    replica_c.clock.update(v1)
    
    # Edições concorrentes (sem sincronização)
    # Replica B edita
    v2 = replica_b.update_document('doc1', 'Content edited by B', v1)
    print(f"Version 2 (B): {v2}")  # [1, 1, 0]
    
    # Replica C edita (concorrente, sem ver edição de B)
    v3 = replica_c.update_document('doc1', 'Content edited by C', v1)
    print(f"Version 3 (C): {v3}")  # [1, 0, 1]
    
    # Análise de causalidade
    print(f"\nv2 happens before v3? {VectorClock._static_happens_before(v2, v3)}")  # False
    print(f"v3 happens before v2? {VectorClock._static_happens_before(v3, v2)}")  # False
    print(f"v2 and v3 concurrent? {not (VectorClock._static_happens_before(v2, v3) or VectorClock._static_happens_before(v3, v2))}")  # True
    
    # Conflito detectado! Sistema pode agora:
    # 1. Apresentar ambas versões ao usuário para resolução manual
    # 2. Aplicar estratégia automática (last-writer-wins, merge, CRDT)
    # 3. Manter ambas versões (multi-version concurrency control)
```

**Uso em Sistemas Reais:**

**Amazon DynamoDB:**
Utiliza variação de vector clocks para versionamento de items e resolução de conflitos em escritas concorrentes.

```python
# DynamoDB usa versioning similar para detectar conflitos
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Documents')

def write_with_version_check(doc_id, new_content, expected_version):
    """
    Write condicional verificando versão
    """
    try:
        table.update_item(
            Key={'doc_id': doc_id},
            UpdateExpression='SET content = :new_content, version = :new_version',
            ConditionExpression='version = :expected_version',
            ExpressionAttributeValues={
                ':new_content': new_content,
                ':expected_version': expected_version,
                ':new_version': expected_version + 1
            }
        )
        return {'success': True}
    except ClientError as e:
        if e.response['Error']['Code'] == 'ConditionalCheckFailedException':
            # Versão mudou! Conflito detectado
            return {'success': False, 'reason': 'version_conflict'}
        raise
```

**Apache Cassandra:**
Usa timestamps (similar a Lamport clocks) para resolver conflitos, mas com limitações em detectar concorrência.

**Riak:**
Implementa vector clocks completos para detectar conflitos e permitir resolução por siblings.

#### 1.1.4 Hybrid Logical Clocks (HLC)

Hybrid Logical Clocks combinam benefícios de physical timestamps (interpretabilidade) com logical clocks (captura de causalidade).

**Estrutura:**

```python
class HybridLogicalClock:
    """
    Hybrid Logical Clock (HLC)
    Combina physical time com logical counter
    """
    def __init__(self, process_id):
        self.process_id = process_id
        self.physical_time = 0
        self.logical_counter = 0
        self.lock = threading.Lock()
    
    def now(self):
        """
        Retorna timestamp HLC atual
        """
        with self.lock:
            # Pega physical time atual
            current_physical = int(time.time() * 1e9)  # Nanosegundos
            
            if current_physical > self.physical_time:
                # Physical time avançou
                self.physical_time = current_physical
                self.logical_counter = 0
            else:
                # Physical time não avançou (dentro do mesmo tick)
                # ou regrediu (clock adjustment)
                # Incrementa logical counter
                self.logical_counter += 1
            
            return {
                'physical': self.physical_time,
                'logical': self.logical_counter,
                'process': self.process_id
            }
    
    def update(self, received_hlc):
        """
        Atualiza HLC ao receber mensagem
        """
        with self.lock:
            current_physical = int(time.time() * 1e9)
            received_physical = received_hlc['physical']
            received_logical = received_hlc['logical']
            
            # Máximo entre physical times
            max_physical = max(current_physical, self.physical_time, received_physical)
            
            if max_physical == current_physical and max_physical == self.physical_time:
                # Current physical == local physical
                if self.physical_time == received_physical:
                    # Todos iguais: max dos logical counters + 1
                    self.logical_counter = max(self.logical_counter, received_logical) + 1
                else:
                    # Current > received: apenas increment logical
                    self.logical_counter += 1
            
            elif max_physical == received_physical:
                # Received physical é maior
                self.physical_time = received_physical
                self.logical_counter = received_logical + 1
            
            else:
                # Current physical é maior
                self.physical_time = max_physical
                self.logical_counter = 0
            
            return {
                'physical': self.physical_time,
                'logical': self.logical_counter,
                'process': self.process_id
            }
    
    def compare(self, hlc1, hlc2):
        """
        Compara dois timestamps HLC
        """
        # Primeiro compara physical time
        if hlc1['physical'] < hlc2['physical']:
            return -1  # hlc1 < hlc2
        elif hlc1['physical'] > hlc2['physical']:
            return 1   # hlc1 > hlc2
        
        # Physical times iguais: compara logical counter
        if hlc1['logical'] < hlc2['logical']:
            return -1
        elif hlc1['logical'] > hlc2['logical']:
            return 1
        
        # Ambos iguais: compara process ID (tiebreaker)
        if hlc1['process'] < hlc2['process']:
            return -1
        elif hlc1['process'] > hlc2['process']:
            return 1
        
        return 0  # Completamente iguais


# Uso em Banco de Dados Distribuído
class DistributedDatabase:
    """
    Database usando HLC para ordenação e versionamento
    """
    def __init__(self, node_id):
        self.node_id = node_id
        self.hlc = HybridLogicalClock(node_id)
        self.data = {}  # key -> [(hlc, value)]
    
    def write(self, key, value):
        """
        Escreve valor com timestamp HLC
        """
        timestamp = self.hlc.now()
        
        if key not in self.data:
            self.data[key] = []
        
        self.data[key].append((timestamp, value))
        
        # Mantém apenas versões recentes (garbage collection)
        self.data[key] = sorted(
            self.data[key],
            key=lambda x: (x[0]['physical'], x[0]['logical'])
        )[-10:]  # Últimas 10 versões
        
        return timestamp
    
    def read(self, key, as_of_timestamp=None):
        """
        Lê valor mais recente ou em timestamp específico
        """
        if key not in self.data:
            return None
        
        versions = self.data[key]
        
        if as_of_timestamp is None:
            # Retorna versão mais recente
            return versions[-1][1]
        
        # Retorna versão válida em as_of_timestamp
        for timestamp, value in reversed(versions):
            if self.hlc.compare(timestamp, as_of_timestamp) <= 0:
                return value
        
        return None  # Nenhuma versão antes de as_of_timestamp
    
    def replicate_write(self, remote_timestamp, key, value):
        """
        Recebe write de outro node
        """
        # Atualiza HLC local
        self.hlc.update(remote_timestamp)
        
        # Aplica write
        if key not in self.data:
            self.data[key] = []
        
        self.data[key].append((remote_timestamp, value))
        
        # Ordena por timestamp
        self.data[key] = sorted(
            self.data[key],
            key=lambda x: (x[0]['physical'], x[0]['logical'])
        )


# Exemplo: CockroachDB usa HLC para MVCC
class CockroachDBStyleMVCC:
    """
    Multi-Version Concurrency Control usando HLC
    Similar ao CockroachDB
    """
    def __init__(self, node_id):
        self.hlc = HybridLogicalClock(node_id)
        self.mvcc_data = {}  # key -> [(hlc_timestamp, value, deleted)]
    
    def mvcc_write(self, key, value):
        """
        Write MVCC: adiciona nova versão
        """
        write_timestamp = self.hlc.now()
        
        if key not in self.mvcc_data:
            self.mvcc_data[key] = []
        
        # Adiciona nova versão
        self.mvcc_data[key].append((write_timestamp, value, False))
        
        return write_timestamp
    
    def mvcc_read(self, key, read_timestamp=None):
        """
        Read MVCC: retorna versão visível no read_timestamp
        """
        if read_timestamp is None:
            read_timestamp = self.hlc.now()
        
        if key not in self.mvcc_data:
            return None
        
        # Encontra versão mais recente <= read_timestamp
        visible_version = None
        
        for version_ts, value, deleted in self.mvcc_data[key]:
            if self.hlc.compare(version_ts, read_timestamp) <= 0:
                if not deleted:
                    visible_version = value
        
        return visible_version
    
    def mvcc_delete(self, key):
        """
        Delete MVCC: marca versão como deletada
        """
        delete_timestamp = self.hlc.now()
        
        if key not in self.mvcc_data:
            self.mvcc_data[key] = []
        
        # Adiciona tombstone (marcador de delete)
        self.mvcc_data[key].append((delete_timestamp, None, True))
        
        return delete_timestamp
    
    def garbage_collect(self, safe_timestamp):
        """
        Remove versões antigas que não são mais necessárias
        """
        for key in list(self.mvcc_data.keys()):
            # Mantém apenas versões >= safe_timestamp
            self.mvcc_data[key] = [
                version for version in self.mvcc_data[key]
                if self.hlc.compare(version[0], safe_timestamp) >= 0
            ]
            
            # Remove key se vazio
            if not self.mvcc_data[key]:
                del self.mvcc_data[key]
```

**Vantagens do HLC:**

1. **Human-readable**: Physical time component é interpretável
2. **Causality**: Mantém propriedades de logical clocks
3. **Boundedness**: Clock drift limitado (diferença entre physical e logical é bounded)
4. **Compatibility**: Pode ser comparado com timestamps físicos

**Sistemas que usam HLC:**

- **CockroachDB**: Transações distribuídas e MVCC
- **YugabyteDB**: Consistência de transações
- **Apache Pulsar**: Ordenação de mensagens
- **MongoDB**: Causal consistency em transações (variação de HLC)

**Referências:**

- Lamport, L. (1978). "Time, Clocks, and the Ordering of Events in a Distributed System". *Communications of the ACM*.
- Mattern, F. (1988). "Virtual Time and Global States of Distributed Systems". *Parallel and Distributed Algorithms*.
- Kulkarni, S. et al. (2014). "Logical Physical Clocks". *OPODIS*.

---


## 2. Detecção de Falhas e Membership

### 2.1 O Problema da Detecção de Falhas

Em sistemas distribuídos, distinguir entre um processo que falhou e um processo que está simplesmente lento é um problema fundamental. Esta seção explora algoritmos para detecção de falhas e gerenciamento de membership de grupos.

#### 2.1.1 Impossibilidade da Detecção Perfeita

**Teorema**: Em sistemas assíncronos (sem bounds de tempo conhecidos), é impossível implementar um detector de falhas perfeito.

**Por quê?** Não há como distinguir entre:
- Processo crashed
- Processo lento
- Rede com alta latência
- Mensagem perdida

```python
# Problema fundamental da detecção de falhas
def problematic_failure_detection():
    """
    Por que detecção perfeita é impossível
    """
    # Cenário: Enviamos heartbeat e esperamos resposta
    send_heartbeat(to='process_b')
    
    # Esperamos timeout
    response = wait_for_response(timeout=5_seconds)
    
    if response is None:
        # O que aconteceu?
        # A) Process B crashed? 
        # B) Process B está sobrecarregado?
        # C) Rede está congestionada?
        # D) Nossa mensagem se perdeu?
        # E) Resposta de B se perdeu?
        
        # IMPOSSÍVEL determinar com certeza!
        # Qualquer decisão pode ser incorreta
        mark_as_failed('process_b')  # Pode ser falso positivo!
```

#### 2.1.2 Detectores de Falhas Não-Perfeitos

Como detecção perfeita é impossível, usamos detectores que podem cometer erros mas com propriedades úteis.

**Propriedades Desejadas:**

- **Completeness (Completude)**: Todo processo que falha é eventualmente detectado
  - **Strong Completeness**: Eventualmente detectado por TODOS os processos corretos
  - **Weak Completeness**: Eventualmente detectado por ALGUM processo correto

- **Accuracy (Precisão)**: Minimizar falsos positivos
  - **Strong Accuracy**: Nenhum processo correto é suspeitado (nunca)
  - **Weak Accuracy**: Algum processo correto nunca é suspeitado
  - **Eventual Strong Accuracy**: Após algum tempo, nenhum correto é suspeitado
  - **Eventual Weak Accuracy**: Após algum tempo, algum correto nunca é suspeitado

**Classes de Detectores:**

```python
from enum import Enum
from typing import Set

class FailureDetectorClass(Enum):
    PERFECT = "P"  # Strong completeness + Strong accuracy (impossível assíncrono)
    EVENTUALLY_PERFECT = "◇P"  # Strong completeness + Eventual strong accuracy
    STRONG = "S"  # Strong completeness + Weak accuracy
    EVENTUALLY_STRONG = "◇S"  # Strong completeness + Eventual weak accuracy
    WEAK = "W"  # Weak completeness + Weak accuracy

class FailureDetector:
    """
    Implementação de detector de falhas Eventually Perfect (◇P)
    """
    def __init__(self, process_id, all_processes, initial_timeout=1.0):
        self.process_id = process_id
        self.all_processes = set(all_processes)
        self.suspected = set()  # Processos suspeitos
        self.alive = set(all_processes)  # Processos vivos
        
        # Timeouts adaptativos
        self.timeout = {p: initial_timeout for p in all_processes}
        self.last_heartbeat = {p: time.time() for p in all_processes}
        
        # Para calcular RTT e ajustar timeouts
        self.rtt_samples = {p: [] for p in all_processes}
        
        self.lock = threading.Lock()
    
    def heartbeat_received(self, from_process, send_time):
        """
        Processa heartbeat recebido
        """
        with self.lock:
            receive_time = time.time()
            
            # Atualiza last heartbeat
            self.last_heartbeat[from_process] = receive_time
            
            # Remove de suspected se estava
            if from_process in self.suspected:
                self.suspected.remove(from_process)
                self.alive.add(from_process)
                print(f"[FD] Process {from_process} recovered")
            
            # Calcula RTT
            rtt = receive_time - send_time
            self.rtt_samples[from_process].append(rtt)
            
            # Mantém apenas últimas 100 samples
            self.rtt_samples[from_process] = self.rtt_samples[from_process][-100:]
            
            # Adapta timeout baseado em RTT
            self._adapt_timeout(from_process)
    
    def _adapt_timeout(self, process):
        """
        Adapta timeout baseado em estatísticas de RTT
        Chen-Wei Algorithm
        """
        samples = self.rtt_samples[process]
        if len(samples) < 10:
            return  # Não adapta com poucas samples
        
        # Calcula média e desvio padrão
        mean_rtt = statistics.mean(samples)
        std_rtt = statistics.stdev(samples)
        
        # Timeout = mean + 4 * std (cobre ~99.99% dos casos se normal distribution)
        safety_margin = 4.0
        new_timeout = mean_rtt + safety_margin * std_rtt
        
        # Bounds: [100ms, 60s]
        new_timeout = max(0.1, min(60.0, new_timeout))
        
        self.timeout[process] = new_timeout
    
    def check_failures(self):
        """
        Verifica timeouts e suspeita processos
        Chamado periodicamente
        """
        with self.lock:
            current_time = time.time()
            
            for process in self.all_processes:
                if process == self.process_id:
                    continue  # Não verificamos nós mesmos
                
                time_since_heartbeat = current_time - self.last_heartbeat[process]
                
                if time_since_heartbeat > self.timeout[process]:
                    if process not in self.suspected:
                        # Timeout! Suspeitar processo
                        self.suspected.add(process)
                        self.alive.discard(process)
                        print(f"[FD] Suspecting process {process} "
                              f"(timeout: {self.timeout[process]:.2f}s, "
                              f"elapsed: {time_since_heartbeat:.2f}s)")
    
    def get_alive_processes(self) -> Set[int]:
        """
        Retorna conjunto de processos considerados vivos
        """
        with self.lock:
            return self.alive.copy()
    
    def is_suspected(self, process) -> bool:
        """
        Verifica se processo é suspeito
        """
        with self.lock:
            return process in self.suspected


# Sistema completo com heartbeats
class HeartbeatSystem:
    """
    Sistema de heartbeats para detecção de falhas
    """
    def __init__(self, process_id, all_processes):
        self.process_id = process_id
        self.detector = FailureDetector(process_id, all_processes)
        self.running = False
        
        # Network simulation
        self.network = NetworkSimulator()
    
    def start(self):
        """
        Inicia threads de heartbeat e failure checking
        """
        self.running = True
        
        # Thread para enviar heartbeats
        self.heartbeat_thread = threading.Thread(
            target=self._send_heartbeats_loop,
            daemon=True
        )
        self.heartbeat_thread.start()
        
        # Thread para check failures
        self.check_thread = threading.Thread(
            target=self._check_failures_loop,
            daemon=True
        )
        self.check_thread.start()
    
    def stop(self):
        """
        Para sistema
        """
        self.running = False
    
    def _send_heartbeats_loop(self):
        """
        Envia heartbeats periodicamente
        """
        while self.running:
            for process in self.detector.all_processes:
                if process != self.process_id:
                    self._send_heartbeat(process)
            
            time.sleep(0.5)  # Heartbeat interval: 500ms
    
    def _send_heartbeat(self, to_process):
        """
        Envia heartbeat para processo
        """
        message = {
            'type': 'HEARTBEAT',
            'from': self.process_id,
            'send_time': time.time()
        }
        
        self.network.send(message, to=to_process)
    
    def _check_failures_loop(self):
        """
        Verifica timeouts periodicamente
        """
        while self.running:
            self.detector.check_failures()
            time.sleep(0.1)  # Check interval: 100ms
    
    def receive_heartbeat(self, message):
        """
        Processa heartbeat recebido
        """
        from_process = message['from']
        send_time = message['send_time']
        
        self.detector.heartbeat_received(from_process, send_time)
```

#### 2.1.3 Algoritmo SWIM (Scalable Weakly-consistent Infection-style Process Group Membership)

SWIM é um protocolo moderno para detecção de falhas e membership, usado em sistemas como HashiCorp Consul, Memberlist e Amazon DynamoDB.

**Características:**

- **Escalável**: Overhead constante por processo
- **Robusto**: Reduce falsos positivos através de indirect probing
- **Rápido**: Detecção em tempo limitado

**Algoritmo:**

```python
import random
from dataclasses import dataclass
from typing import List, Set

@dataclass
class MemberInfo:
    """Informação sobre membro do grupo"""
    process_id: int
    address: str
    status: str  # 'alive', 'suspect', 'dead'
    incarnation: int  # Para refutar suspeitas
    timestamp: float

class SWIMProtocol:
    """
    Implementação do protocolo SWIM
    """
    def __init__(self, process_id, address, seed_members):
        self.process_id = process_id
        self.address = address
        
        # Membership list
        self.members = {}  # process_id -> MemberInfo
        for member_id, member_addr in seed_members:
            self.members[member_id] = MemberInfo(
                process_id=member_id,
                address=member_addr,
                status='alive',
                incarnation=0,
                timestamp=time.time()
            )
        
        # Own incarnation number (para refutar suspeitas)
        self.incarnation = 0
        
        # Configuração
        self.protocol_period = 1.0  # 1 segundo
        self.timeout = 0.5  # Timeout para ping
        self.k_indirect = 3  # Número de indirect probes
        
        self.lock = threading.Lock()
        self.running = False
    
    def start(self):
        """Inicia protocolo SWIM"""
        self.running = True
        self.protocol_thread = threading.Thread(
            target=self._protocol_loop,
            daemon=True
        )
        self.protocol_thread.start()
    
    def _protocol_loop(self):
        """
        Loop principal do protocolo SWIM
        """
        while self.running:
            # Seleciona membro aleatório para probe
            target = self._select_random_member()
            
            if target:
                # Tenta direct ping
                if not self._direct_ping(target):
                    # Falhou! Tenta indirect ping através de outros
                    if not self._indirect_ping(target):
                        # Indirect também falhou! Marcar como suspect
                        self._mark_suspect(target)
            
            # Dissemina updates via gossip
            self._gossip_membership_updates()
            
            time.sleep(self.protocol_period)
    
    def _select_random_member(self):
        """
        Seleciona membro aleatório (exceto self)
        """
        with self.lock:
            alive_members = [
                m for m in self.members.values()
                if m.status == 'alive' and m.process_id != self.process_id
            ]
            
            if not alive_members:
                return None
            
            return random.choice(alive_members)
    
    def _direct_ping(self, target: MemberInfo) -> bool:
        """
        Envia PING direto e espera ACK
        """
        message = {
            'type': 'PING',
            'from': self.process_id,
            'seq': random.randint(0, 1000000)
        }
        
        try:
            # Envia ping com timeout
            response = self._send_with_timeout(
                message,
                to=target.address,
                timeout=self.timeout
            )
            
            if response and response['type'] == 'ACK':
                # Success! Target está vivo
                self._update_member_alive(target.process_id)
                return True
        
        except TimeoutError:
            pass
        
        return False
    
    def _indirect_ping(self, target: MemberInfo) -> bool:
        """
        PING através de K membros intermediários
        Se algum receber ACK do target, está vivo
        """
        with self.lock:
            # Seleciona K membros aleatórios (exceto target e self)
            intermediaries = [
                m for m in self.members.values()
                if m.status == 'alive' 
                and m.process_id != self.process_id
                and m.process_id != target.process_id
            ]
            
            if not intermediaries:
                return False
            
            k_intermediaries = random.sample(
                intermediaries,
                min(self.k_indirect, len(intermediaries))
            )
        
        # Envia PING-REQ para cada intermediário
        responses = []
        threads = []
        
        def ping_via_intermediary(intermediary):
            message = {
                'type': 'PING-REQ',
                'from': self.process_id,
                'target': target.process_id,
                'target_address': target.address,
                'seq': random.randint(0, 1000000)
            }
            
            try:
                response = self._send_with_timeout(
                    message,
                    to=intermediary.address,
                    timeout=self.timeout
                )
                
                if response and response['type'] == 'ACK':
                    responses.append(True)
            except TimeoutError:
                pass
        
        for intermediary in k_intermediaries:
            t = threading.Thread(target=ping_via_intermediary, args=(intermediary,))
            t.start()
            threads.append(t)
        
        # Espera todos terminarem
        for t in threads:
            t.join()
        
        # Se algum teve sucesso, target está vivo
        if any(responses):
            self._update_member_alive(target.process_id)
            return True
        
        return False
    
    def _mark_suspect(self, target: MemberInfo):
        """
        Marca membro como suspect
        """
        with self.lock:
            if target.process_id in self.members:
                member = self.members[target.process_id]
                
                if member.status == 'alive':
                    print(f"[SWIM] Marking {target.process_id} as SUSPECT")
                    member.status = 'suspect'
                    member.timestamp = time.time()
                    
                    # Agenda para marcar como dead se não refutar
                    threading.Timer(
                        self.protocol_period * 3,
                        self._confirm_death,
                        args=(target.process_id, member.incarnation)
                    ).start()
    
    def _confirm_death(self, process_id, incarnation_at_suspicion):
        """
        Confirma morte se suspeita não foi refutada
        """
        with self.lock:
            if process_id in self.members:
                member = self.members[process_id]
                
                # Se incarnation number não aumentou, confirma morte
                if (member.status == 'suspect' and 
                    member.incarnation == incarnation_at_suspicion):
                    
                    print(f"[SWIM] Confirmed {process_id} as DEAD")
                    member.status = 'dead'
                    member.timestamp = time.time()
    
    def _update_member_alive(self, process_id):
        """
        Atualiza membro como alive
        """
        with self.lock:
            if process_id in self.members:
                member = self.members[process_id]
                if member.status != 'alive':
                    print(f"[SWIM] {process_id} is ALIVE again")
                member.status = 'alive'
                member.timestamp = time.time()
    
    def handle_suspicion(self, about_process, incarnation):
        """
        Recebe notificação de suspeita sobre processo
        """
        if about_process == self.process_id:
            # Suspeita sobre NÓS! Refutar
            self.incarnation = max(self.incarnation, incarnation) + 1
            
            print(f"[SWIM] Refuting suspicion, new incarnation: {self.incarnation}")
            
            # Dissemina refutação via gossip
            self._gossip_alive_message()
        else:
            # Suspeita sobre outro processo
            with self.lock:
                if about_process in self.members:
                    member = self.members[about_process]
                    
                    # Atualiza apenas se incarnation é mais recente
                    if incarnation > member.incarnation:
                        member.status = 'suspect'
                        member.incarnation = incarnation
                        member.timestamp = time.time()
    
    def _gossip_membership_updates(self):
        """
        Dissemina updates sobre membership via gossip
        """
        with self.lock:
            # Seleciona membros aleatórios para enviar gossip
            targets = random.sample(
                list(self.members.values()),
                min(3, len(self.members))
            )
        
        # Prepara updates para disseminar
        updates = self._get_recent_updates()
        
        for target in targets:
            if target.process_id != self.process_id and target.status == 'alive':
                message = {
                    'type': 'GOSSIP',
                    'from': self.process_id,
                    'updates': updates
                }
                
                try:
                    self._send_async(message, to=target.address)
                except Exception as e:
                    print(f"[SWIM] Failed to gossip to {target.process_id}: {e}")
    
    def handle_gossip(self, updates):
        """
        Processa updates recebidos via gossip
        """
        with self.lock:
            for update in updates:
                process_id = update['process_id']
                status = update['status']
                incarnation = update['incarnation']
                
                if process_id == self.process_id:
                    # Update sobre nós mesmos
                    if status == 'suspect' and incarnation >= self.incarnation:
                        # Refutar!
                        self.handle_suspicion(process_id, incarnation)
                    continue
                
                # Update sobre outro processo
                if process_id not in self.members:
                    # Novo membro descoberto!
                    self.members[process_id] = MemberInfo(
                        process_id=process_id,
                        address=update['address'],
                        status=status,
                        incarnation=incarnation,
                        timestamp=time.time()
                    )
                    print(f"[SWIM] Discovered new member: {process_id}")
                else:
                    member = self.members[process_id]
                    
                    # Atualiza apenas se informação é mais recente
                    if incarnation > member.incarnation or (
                        incarnation == member.incarnation and
                        status == 'dead' and member.status != 'dead'
                    ):
                        member.status = status
                        member.incarnation = incarnation
                        member.timestamp = time.time()
```

**Propriedades do SWIM:**

1. **Detection Time**: O(log N) períodos de protocolo
2. **False Positive Rate**: Muito baixa devido a indirect probing
3. **Message Load**: Constante por processo (não cresce com N)
4. **Completeness**: Eventual (todos eventualmente detectam falhas)

**Uso em Sistemas Reais:**

- **HashiCorp Consul**: Service discovery e health checking
- **HashiCorp Serf**: Cluster membership
- **Amazon DynamoDB**: Membership de anéis de consistent hashing
- **Apache Cassandra**: Gossip protocol baseado em SWIM

#### 2.1.4 Phi Accrual Failure Detector

Usado em Apache Cassandra, fornece suspeita gradual ao invés de binária.

```python
import math
from collections import deque

class PhiAccrualFailureDetector:
    """
    Phi Accrual Failure Detector
    Retorna valor contínuo de suspeita ao invés de binário
    """
    def __init__(self, threshold=8.0, max_sample_size=1000, min_std_dev=0.5):
        self.threshold = threshold  # Phi threshold para considerar down
        self.max_sample_size = max_sample_size
        self.min_std_dev = min_std_dev
        
        # Histórico de intervalos entre heartbeats
        self.intervals = deque(maxlen=max_sample_size)
        self.last_heartbeat_time = None
    
    def heartbeat(self):
        """
        Registra heartbeat recebido
        """
        now = time.time()
        
        if self.last_heartbeat_time is not None:
            interval = now - self.last_heartbeat_time
            self.intervals.append(interval)
        
        self.last_heartbeat_time = now
    
    def phi(self):
        """
        Calcula valor Phi atual (nível de suspeita)
        
        Phi representa: -log10(P(alive))
        - Phi = 1: 90% de confiança que falhou
        - Phi = 2: 99% de confiança que falhou
        - Phi = 3: 99.9% de confiança que falhou
        """
        if self.last_heartbeat_time is None or len(self.intervals) < 2:
            return 0.0
        
        now = time.time()
        time_since_last_heartbeat = now - self.last_heartbeat_time
        
        # Calcula média e desvio padrão dos intervalos
        mean_interval = statistics.mean(self.intervals)
        std_dev = statistics.stdev(self.intervals)
        std_dev = max(std_dev, self.min_std_dev)  # Evita divisão por zero
        
        # Normaliza time_since_last_heartbeat
        y = time_since_last_heartbeat
        
        # Calcula CDF da distribuição normal
        # P(X > y) onde X ~ N(mean, std_dev^2)
        z = (y - mean_interval) / std_dev
        probability_alive = 1.0 - self._normal_cdf(z)
        
        # Evita log(0)
        probability_alive = max(probability_alive, 1e-10)
        
        # Phi = -log10(P(alive))
        phi_value = -math.log10(probability_alive)
        
        return phi_value
    
    def is_available(self):
        """
        Determina se processo está disponível
        """
        return self.phi() < self.threshold
    
    @staticmethod
    def _normal_cdf(z):
        """
        Aproximação da CDF da distribuição normal padrão
        """
        # Aproximação usando função de erro
        return 0.5 * (1.0 + math.erf(z / math.sqrt(2.0)))


# Exemplo de uso
def demonstrate_phi_accrual():
    """
    Demonstra como Phi Accrual funciona
    """
    detector = PhiAccrualFailureDetector(threshold=8.0)
    
    # Simula heartbeats regulares (1 por segundo)
    for i in range(10):
        detector.heartbeat()
        time.sleep(1.0)
        phi = detector.phi()
        print(f"t={i+1}s: Phi = {phi:.2f}, Available = {detector.is_available()}")
    
    # Simula falha (para de enviar heartbeats)
    print("\n--- Process stopped sending heartbeats ---")
    for i in range(10):
        time.sleep(1.0)
        phi = detector.phi()
        print(f"t={11+i}s: Phi = {phi:.2f}, Available = {detector.is_available()}")
    
    # Output esperado:
    # Primeiros 10s: Phi baixo (<1), Available=True
    # Após falha: Phi cresce gradualmente
    # Quando Phi > 8.0: Available=False (detectado como down)
```

**Vantagens sobre detectores binários:**

1. **Adaptativo**: Ajusta-se automaticamente a variações de rede
2. **Gradual**: Permite políticas mais sofisticadas (ex: "wait mais antes de remover da pool")
3. **Configurável**: Threshold pode ser ajustado para balancear false positives vs detection time

---

