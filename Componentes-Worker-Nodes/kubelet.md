# Kubelet - Guia Completo

## 1. O que é Kubelet?

**Kubelet** é um agente que roda em **cada nó do cluster Kubernetes**. É responsável por **executar e gerenciar os containers** nos nós, garantindo que os pods rodem conforme desejado.

### 1.1 Analogia

Imagine o Kubelet como um **gerente de um restaurante**:

- 👨‍🍳 Cada nó é um restaurante
- 📋 Kubelet recebe pedidos (pods) da central
- ⚙️ Executa os pedidos (containers)
- 👀 Monitora a qualidade (saúde dos containers)
- 🔧 Faz ajustes quando necessário

---

## 2. Responsabilidades Principais

### 2.1 Execução de Containers

- **Executar containers** em seu nó
- Trabalhar com **container runtime** (Docker, containerd, cri-o)
- Gerenciar **ciclo de vida** dos containers

### 2.2 Monitoramento de Pods

- Monitorar **saúde dos containers**
- Detectar **falhas e reiniciar** containers
- Relatar **status do pod** ao Kube-API-Server

### 2.3 Gerenciamento de Recursos

- Garantir **limites de CPU/memória**
- Gerenciar **volumes**
- Alocar **IPs aos pods**

### 2.4 Comunicação com API Server

- **Registrar o nó** no cluster
- **Informar status** do nó e pods
- **Receber instruções** (novos pods, deletar pods)

### 2.5 Health Checks

- Executar **liveness probes** (pod vivo?)
- Executar **readiness probes** (pronto para tráfego?)
- Executar **startup probes** (iniciado?)

### 2.6 Gerenciamento de Volumes

- **Montar volumes** em containers
- **Gerenciar storage** local

---

## 3. Arquitetura do Kubelet

### 3.1 Componentes Internos

```
┌──────────────────────────────────────────────────────────┐
│                    Kubelet (em cada nó)                  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  1. API Server Watcher                             │ │
│  │     - Monitora novo pods agendados para este nó    │ │
│  │     - Detecta pods para deletar                    │ │
│  │     - Recebe atualizações                          │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  2. Pod Manager                                   │ │
│  │     - Gerencia ciclo de vida do pod               │ │
│  │     - Rastreia pods em execução                   │ │
│  │     - Mapeia especificações ao runtime            │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  3. Volume Manager                                │ │
│  │     - Monta volumes                               │ │
│  │     - Gerencia armazenamento                      │ │
│  │     - Unmonta quando necessário                   │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  4. Container Runtime Interface (CRI)             │ │
│  │     - Comunica com runtime (Docker, containerd)   │ │
│  │     - Abstração para diferentes runtimes          │ │
│  │     - Gerencia containers                         │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  5. Probe Manager                                 │ │
│  │     - Executa liveness probes                     │ │
│  │     - Executa readiness probes                    │ │
│  │     - Executa startup probes                      │ │
│  │     - Reinicia containers que falharem            │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  6. CGROUP Manager                                │ │
│  │     - Aplica limites de recursos (CPU, memória)   │ │
│  │     - Isolamento de recursos                      │ │
│  │     - QoS (Quality of Service)                    │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  7. Status Manager                                │ │
│  │     - Rastreia status de pods/containers          │ │
│  │     - Reporta ao API-Server                       │ │
│  │     - Monitora saúde                              │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  8. Container Runtime                             │ │
│  │     - Docker/containerd/cri-o                     │ │
│  │     - Executa containers realmente                │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
└──────────────────────────────────────────────────────────┘

```

---

## 4. Ciclo de Vida de um Pod no Kubelet

### 4.1 Fluxo Completo

```
1. 👀 WATCH API-Server
   └─ Kubelet monitora novo pod agendado

2. 📥 RECEBE POD
   ├─ Nome, image, volumes, etc
   └─ Armazena em Pod Manager

3. 🔌 CRIA INFRA
   ├─ Cria namespace
   ├─ Configura rede (obtém IP)
   └─ Monta volumes

4. 📦 PULL IMAGE
   ├─ Baixa imagem do container
   └─ Armazena localmente

5. ⚙️  CRIA CONTAINER
   ├─ Comunica com CRI/Runtime
   ├─ Cria container com configs
   └─ Aplica limites de recursos

6. 🚀 INICIA CONTAINER
   └─ Container começa a rodar

7. 💚 MONITORA
   ├─ Executa health checks
   ├─ Valida readiness
   └─ Reinicia se necessário

8. 📊 RELATA STATUS
   ├─ Envia status ao API-Server
   └─ Atualiza ETCD

```

### 4.2 Exemplo Visual: Pod Sendo Executado

```
┌─────────────────────────────────────────────┐
│  Pod nginx-app (Agendado no Nó-1)          │
└────────────┬────────────────────────────────┘
             │
    ┌────────▼────────┐
    │  Kubelet (Nó-1) │
    └────────┬────────┘
             │
    1️⃣  Vê novo pod via API-Server
             │
    2️⃣  Pod Manager armazena especificação
             │
    3️⃣  Volume Manager monta volumes (se houver)
             │
    4️⃣  CRI/Runtime: Pull image nginx:latest
             │
             ▼
    ┌──────────────────┐
    │ Registry          │
    │ nginx:latest      │
    │ (200MB)          │
    └────────┬─────────┘
             │ Download
             ▼
    5️⃣  CRI/Runtime: Cria container
    ├─ CPU limit: 500m
    ├─ Memory limit: 256Mi
    ├─ Port 80
    └─ Mount /data
             │
    6️⃣  CRI/Runtime: Inicia container
             │
             ▼
    ┌──────────────────┐
    │ Container nginx  │
    │ PID: 1234        │
    │ IP: 10.0.0.5     │
    │ Status: Running  │
    └─────────────────┘
             │
    7️⃣  Probe Manager: Executa readiness probe
    ├─ curl http://localhost/health
    ├─ Sucesso: Container ready
    └─ Pod entra em estado Running
             │
    8️⃣  Status Manager: Reporta ao API-Server
    └─ Pod status: Running

```

---

## 5. Monitoramento e Health Checks

### 5.1 Tipos de Probes

Kubelet executa 3 tipos de health checks:

### Liveness Probe

**Pergunta**: O container ainda está vivo?
**Se falhar**: Kubelet **reinicia** o container
**Uso**: Detectar containers travados

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-liveness
spec:
  containers:
  - name: app
    image: app:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10  # Espera 10s antes de começar
      periodSeconds: 5          # Verifica a cada 5s
      failureThreshold: 3       # Falha após 3 tentativas

```

### Readiness Probe

**Pergunta**: O container está pronto para receber tráfego?
**Se falhar**: Pod é **removido do service** (não recebe tráfego)
**Uso**: Detectar quando container está inicializando

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-readiness
spec:
  containers:
  - name: app
    image: app:latest
    readinessProbe:
      exec:
        command:
        - /bin/check-health.sh
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 2

```

### Startup Probe

**Pergunta**: O container terminou de inicializar?
**Se falhar**: Kubelet **não executa** outras probes até sucesso
**Uso**: Apps que levam tempo para iniciar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-slow-startup
spec:
  containers:
  - name: app
    image: app:latest
    startupProbe:
      tcpSocket:
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 5

```

### 5.2 Tipos de Verificação

| Tipo | Descrição | Exemplo |
| --- | --- | --- |
| **httpGet** | Faz HTTP request | GET /health |
| **tcpSocket** | Tenta conectar TCP | Porta 8080 aberta? |
| **exec** | Executa comando | `/bin/check.sh` |
| **grpc** | Chamada gRPC | Check service |

### 5.3 Exemplo Prático: Ciclo de Probes

```
Container inicializa
         │
         ▼
    ┌─────────────────────┐
    │ Startup Probe       │ ← Verifica init
    │ (TCP 8080)          │
    └─────────────────────┘
         │ Sucesso após 3s
         ▼
    ┌─────────────────────┐
    │ Readiness Probe     │ ← Verifica pronto
    │ (GET /ready)        │
    └─────────────────────┘
         │ Sucesso
         ▼
    Pod entra em service (recebe tráfego)
         │
         ▼ (a cada 5s)
    ┌─────────────────────┐
    │ Liveness Probe      │ ← Verifica vivo
    │ (GET /health)       │
    └─────────────────────┘
         │ Falha 3 vezes
         ▼
    Container é reiniciado

```

---

## 6. Gerenciamento de Recursos

### 6.1 Requests vs Limits

Kubelet aplica restrições de recursos através de cgroups:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-limited-app
spec:
  containers:
  - name: app
    image: app:latest
    resources:
      # Quantidade GARANTIDA ao container
      requests:
        cpu: 250m        # 0.25 cores de CPU
        memory: 128Mi    # 128 MB de memória

      # Limite MÁXIMO que pode usar
      limits:
        cpu: 500m        # Máximo 0.5 cores
        memory: 256Mi    # Máximo 256 MB

```

### 6.2 QoS Classes (Quality of Service)

Kubelet classifica pods em 3 classes de QoS:

```
┌─────────────────────────────────────────────┐
│  Pod com requests == limits                 │
│  → QoS: Guaranteed (Máxima Prioridade)      │
│  → Nunca é evicted                          │
│                                             │
│  resources:                                 │
│    requests: {cpu: 500m, memory: 256Mi}     │
│    limits: {cpu: 500m, memory: 256Mi}       │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Pod com requests < limits                  │
│  → QoS: Burstable (Prioridade Média)        │
│  → Evicted se houver sobrecarga             │
│                                             │
│  resources:                                 │
│    requests: {cpu: 250m, memory: 128Mi}     │
│    limits: {cpu: 500m, memory: 256Mi}       │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Pod sem requests/limits                    │
│  → QoS: BestEffort (Menor Prioridade)       │
│  → Primeiro a ser evicted                   │
│                                             │
│  resources: {}                              │
│  (Nenhum recurso garantido)                 │
└─────────────────────────────────────────────┘

```

### 6.3 Eviction Policy (Quando Recursos Escassos)

Quando nó fica sem recursos, Kubelet **evita (deleta) pods** pela ordem:

```
Ordem de Eviction:
1. BestEffort pods (sem requests)
2. Burstable pods (usando mais que requests)
3. Guaranteed pods (só se realmente crítico)

```

---

## 7. Gerenciamento de Volumes

### 7.1 Tipos de Volumes Gerenciados

```
Kubelet gerencia:

┌────────────────────────────────┐
│ Volume Types                   │
├────────────────────────────────┤
│ • emptyDir (local, temporário) │
│ • configMap (configurações)    │
│ • secret (dados sensíveis)     │
│ • downwardAPI (metadata)       │
│ • hostPath (arquivo do host)   │
│ • persistentVolumeClaim (PVC)  │
│ • nfs, awsEBS, gcePersistentDisk
│ • E outros...                  │
└────────────────────────────────┘

```

### 7.2 Exemplo: Pod com Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-volumes
spec:
  containers:
  - name: app
    image: app:latest
    volumeMounts:
    - name: config          # ← Monta ConfigMap
      mountPath: /etc/config
    - name: logs            # ← Monta Volume Local
      mountPath: /var/log
    - name: secret-data     # ← Monta Secret
      mountPath: /secrets

  volumes:
  - name: config
    configMap:
      name: app-config

  - name: logs
    emptyDir: {}            # Volume vazio, deletado ao finalizar pod

  - name: secret-data
    secret:
      secretName: db-password

```

**O que Kubelet faz:**

1. Obtém ConfigMap `app-config` do API-Server
2. Cria diretório local `/var/log`
3. Obtém Secret `db-password` do API-Server
4. Monta tudo no container
5. Quando pod termina, remove volumes emptyDir

---

## 8. Registro do Nó

### 8.1 Heartbeat ao API-Server

Kubelet envia heartbeat regularmente:

```
Kubelet (Nó-1)
    │
    │ A cada 10s (default)
    │
    ▼
┌──────────────────────────────────────┐
│ Kube-API-Server                      │
│                                      │
│ Node status:                         │
│ - Name: nó-1                         │
│ - Status: Ready                      │
│ - CPU: 4 cores                       │
│ - Memory: 8Gi                        │
│ - Disk: 100Gi                        │
│ - Pods running: 15                   │
│ - Pods capacity: 110                 │
└──────────────────────────────────────┘
         │
         ▼ (Persiste em ETCD)

```

### 8.2 Node Readiness

Se Kubelet parar de enviar heartbeat:

```
Tempo 0s: Kubelet normal
    │ Heartbeat OK
    ▼
    Status: Ready

Tempo 40s: Kubelet desconecta
    │ Sem heartbeat
    ▼
    Status: NotReady (Node Controller detecta)

Tempo 5m: Continua sem heartbeat
    │ Node Controller toma ação
    ▼
    1. Mark node "NotReady"
    2. Node Controller evita pods
    3. ReplicaSet Controller cria pods em outro nó

```

---

## 9. Configuração do Kubelet

### 9.1 Arquivo de Configuração

```bash
kubelet \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --node-name=nó-1 \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --cgroup-driver=systemd \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --cluster-dns=10.0.0.10 \
  --cluster-domain=cluster.local \
  --allow-privileged=false \
  --max-pods=110 \
  --image-gc-high-threshold=85 \
  --image-gc-low-threshold=80 \
  --v=2

```

**Flags Importantes:**

- `-node-name` - Nome do nó no cluster
- `-container-runtime-endpoint` - Onde conectar ao runtime
- `-pod-manifest-path` - Pods estáticos (lê do disco)
- `-cluster-dns` - Servidor DNS do cluster
- `-max-pods` - Máximo de pods no nó
- `-image-gc-high-threshold` - Quando limpar imagens (85%)

### 9.2 KubeletConfiguration via YAML

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
nodeStatusUpdateFrequency: 10s
nodeStatusReportFrequency: 1m0s
address: 0.0.0.0
port: 10250
readOnlyPort: 0
serverTLSBootstrap: true
tlsCipherSuites:
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
maxPods: 110
podsPerCore: 0
resyncFrequency: 1m0s
fileCheckFrequency: 20s
healthzPort: 10248
healthzBindAddress: 127.0.0.1
makeIPTablesUtilChains: true
iptablesMasqueradeBit: 14
iptablesDropBit: 15
featureGates: {}
failSwapOn: true
memorySwap: {}
containerLogMaxSize: 10Mi
containerLogMaxFiles: 5
eventRecordQPS: 5
eventBurst: 10
kubeReserved: {}
systemReserved: {}
hardEvictionThresholds:
- memory.available<100Mi
- nodefs.available<10%
softEvictionThresholds:
- memory.available<500Mi
softEvictionGracePeriod:
- memory.available=1m30s
imagefs:
  inodesFree: 5%

```

---

## 10. Static Pods

### 10.1 O que são Static Pods?

Static Pods são definidos em arquivos YAML no nó e gerenciados **diretamente pelo Kubelet**, sem necessidade do API-Server.

```bash
# Arquivos estáticos em:
/etc/kubernetes/manifests/

# Kubelet monitora esta pasta e executa qualquer Pod YAML

```

### 10.2 Uso Comum: Control Plane

```
/etc/kubernetes/manifests/
├── etcd.yaml           ← Static Pod do ETCD
├── kube-apiserver.yaml ← Static Pod do API-Server
├── kube-controller-manager.yaml ← Static Pod do Controller Manager
└── kube-scheduler.yaml ← Static Pod do Scheduler

```

**Por que?** O control plane precisa rodar mesmo antes do cluster estar 100% funcional.

### 10.3 Exemplo: Static Pod

```yaml
# /etc/kubernetes/manifests/my-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-static
spec:
  containers:
  - name: app
    image: app:latest
    ports:
    - containerPort: 8080

```

Kubelet automaticamente:

1. Lê arquivo
2. Cria pod
3. Monitora
4. Reinicia se necessário
5. Se arquivo for deletado, pod é deletado

---

## 11. Container Runtime Interface (CRI)

### 11.1 Como Kubelet Comunica com Runtime

```
┌─────────────┐
│   Kubelet   │
│             │
│ Quer criar: │
│ Container   │
│ nginx:1.21  │
└──────┬──────┘
       │ gRPC (CRI)
       │
       ▼
┌─────────────────────────────────────┐
│  Container Runtime Interface (CRI)  │
│  Socket: unix:///run/containerd...  │
└─────────────────────────────────────┘
       │
       ▼ (Traduz para)
┌─────────────────────────┐
│  Container Runtime      │
│  • containerd           │
│  • cri-o                │
│  • Docker (via cri)     │
└─────────────────────────┘
       │
       ▼
    Cria container realmente

```

### 11.2 Operações CRI Comuns

```
kubelet chama:
├─ CreateContainer()    ← Cria novo container
├─ StartContainer()     ← Inicia container
├─ StopContainer()      ← Para container
├─ RemoveContainer()    ← Deleta container
├─ ListContainers()     ← Lista containers
├─ GetContainerStatus() ← Status do container
├─ ExecSync()          ← Executa comando no container
└─ PullImage()         ← Baixa imagem do registry

```

---

## 12. Monitoramento e Troubleshooting

### 12.1 Ver Status do Nó

```bash
# Ver status do nó
kubectl get nodes

# Ver detalhes do nó
kubectl describe node nó-1

# Ver recursos do nó
kubectl top nodes

# Ver pods no nó
kubectl get pods -o wide | grep nó-1

```

### 12.2 Ver Logs do Kubelet

```bash
# Logs do kubelet
sudo journalctl -u kubelet -f

# Logs em arquivo
tail -f /var/log/pods/*/*/kubelet.log

# Kubectl logs de pod
kubectl logs -n kube-system <pod-name>

```

### 12.3 Problemas Comuns

### Problema 1: Node is NotReady

```
$ kubectl get nodes
NAME    STATUS     ROLES   AGE
nó-1    NotReady   master  5d

```

**Causa possível**: Kubelet parou ou não consegue conectar ao API-Server
**Solução**:

```bash
# Verificar kubelet
sudo systemctl status kubelet

# Ver logs
sudo journalctl -u kubelet -n 50

# Reiniciar
sudo systemctl restart kubelet

```

### Problema 2: Pod Stuck in Pending

```
$ kubectl describe pod meu-pod
Status: Pending
Events:
  FailedScheduling: no nodes available

```

**Causa**: Kubelet indisponível ou falta de recursos
**Solução**: Verificar status de nós com `kubectl get nodes`

### Problema 3: Pod Stuck in ImagePullBackOff

```
$ kubectl describe pod meu-pod
Status: ImagePullBackOff
Events:
  Failed to pull image "app:typo": image not found

```

**Causa**: Imagem não existe ou typo no nome
**Solução**: Corrigir nome da imagem

### Problema 4: Container Restarting Continuously

```
$ kubectl get pods
NAME      RESTARTS
app-pod   5 (5m ago)

```

**Causa**: Liveness probe falhando, ou aplicação quebrando
**Solução**: Verificar logs com `kubectl logs <pod> --previous`

### 12.4 Comando Útil: exec (Debugar Container)

```bash
# Executar comando em container
kubectl exec -it meu-pod -- /bin/bash

# Executar comando específico
kubectl exec meu-pod -- ps aux

# Em pod com múltiplos containers
kubectl exec -it meu-pod -c container-name -- /bin/bash

```

---

## 13. Fluxo Completo: Pod Do Começo Ao Fim

### 13.1 Vida do Pod: Passo a Passo

```
1. Usuário cria Pod
   $ kubectl apply -f pod.yaml

   ↓

2. Kube-API-Server recebe e persiste no ETCD

   ↓

3. Kube Scheduler:
   ├─ Encontra pod sem nó
   ├─ Escolhe nó-1
   └─ Atribui pod ao nó-1

   ↓

4. Kubelet (nó-1):
   ├─ Watch API-Server detecta novo pod
   ├─ Pod Manager armazena especificação
   ├─ Volume Manager monta volumes (se houver)
   ├─ CRI: Pull image
   ├─ CRI: Create container
   ├─ CRI: Start container
   ├─ Pod status = ContainerCreating
   ├─ Startup Probe: Verifica se iniciou
   ├─ Pod status = Running
   ├─ Readiness Probe: Verifica se pronto
   ├─ Pod status = Ready (pode receber tráfego)
   └─ Status Manager: Reporta ao API-Server

   ↓

5. Kube Controller Manager:
   ├─ Detecta pod criado
   └─ Service Controller adiciona pod ao service

   ↓

6. Pod Recebendo Tráfego
   ├─ Liveness Probe executado a cada 5s
   ├─ Container monitora logs
   └─ Kubelet continua observando

   ↓

7. Usuário Deleta Pod
   $ kubectl delete pod meu-pod

   ↓

8. Kube-API-Server marca para deleção

   ↓

9. Kubelet (nó-1):
   ├─ Detecta marcação de deleção
   ├─ Inicia graceful shutdown (30s default)
   ├─ Container recebe SIGTERM
   ├─ Espera container finalizar
   ├─ Se não finalizar em 30s, SIGKILL
   ├─ CRI: Stop container
   ├─ CRI: Remove container
   ├─ Volume Manager unmount volumes
   └─ Status Manager: Reporta deleção

   ↓

10. Pod Removido do ETCD

```

---

## 14. Resumo das Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Execução** | Executar containers via CRI |
| **Monitoramento** | Health checks e status |
| **Recursos** | Limites de CPU/memória via cgroups |
| **Volumes** | Montar e gerenciar volumes |
| **Comunicação** | Relatar status ao API-Server |
| **Probes** | Liveness, readiness, startup |
| **Cleanup** | Remover containers quando necessário |
| **Node Reg** | Registrar-se no cluster |

---

## 15. Comparação: Kubelet vs Outros Componentes

```
┌──────────────────────────────────────────────────────┐
│  Componente        │  Localização  │  Função        │
├──────────────────────────────────────────────────────┤
│  Kube-API-Server   │  Master       │  Comunicação   │
│  Controller Manager│  Master       │  Orquestração  │
│  Kube-Scheduler    │  Master       │  Placement     │
│  Kubelet           │  Cada Nó      │  Execução      │ ← Aqui!
│  Kube-Proxy        │  Cada Nó      │  Networking    │
└──────────────────────────────────────────────────────┘

```

---

## 16. Diagrama Completo da Orquestração

```
┌────────────────────────────────────────────────────────┐
│  1. Usuário: kubectl apply -f pod.yaml                 │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  2. Kube-API-Server                                    │
│     └─ Salva Pod no ETCD                               │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  3. Kube-Scheduler                                     │
│     └─ Atribui pod ao nó-1                             │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  4. Kubelet (nó-1)  ← VOCÊ ESTÁ AQUI                  │
│                                                        │
│     Watch: Detecta novo pod                           │
│     ↓                                                  │
│     Volume Manager: Monta volumes                     │
│     ↓                                                  │
│     CRI: Pull image                                   │
│     ↓                                                  │
│     CRI: Create + Start container                     │
│     ↓                                                  │
│     Probe Manager: Executa health checks              │
│     ↓                                                  │
│     Status Manager: Reporta ao API-Server             │
│     ↓                                                  │
│     Continua monitorando container                    │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  5. Container Rodando                                  │
│     Application executando no nó                       │
└────────────────────────────────────────────────────────┘

```

---

## 17. Conclusão

- **Kubelet** é o agente que executa containers em cada nó
- **Responsável por monitorar** saúde e estado dos containers
- **Comunica com API-Server** para receber instruções e relatar status
- **Gerencia recursos** (CPU, memória) via cgroups
- **Executa health checks** (liveness, readiness, startup)
- **Gerencia volumes** e armazenamento
- É o **ponto de execução real** do Kubernetes

---

