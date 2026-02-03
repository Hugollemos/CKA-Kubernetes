# Kube Controller Manager - Guia Completo

## 1. O que é Kube Controller Manager?

**Kube Controller Manager** é um componente crítico do Kubernetes que funciona como o **"maestro"** do cluster. É responsável por monitorar continuamente o estado de vários componentes e garantir que o sistema sempre funcione no **estado desejado**.

### 1.1 Analogia

Imagine o Kube Controller Manager como um **maestro de orquestra**:

- 🎵 Cada músico (componente) é monitorado continuamente
- 📋 O maestro tem a partitura (estado desejado)
- 🔄 Se algum músico para de tocar, o maestro intervém
- 🎼 Garante que toda a orquestra toque em harmonia

---

## 2. Responsabilidades Principais

### 2.1 Monitoramento Contínuo

- Observa constantemente o estado dos recursos no cluster
- Detecta mudanças e desvios do estado desejado
- Reage automaticamente a problemas

### 2.2 Manutenção do Estado Desejado

- Garante que o estado atual **sempre corresponda** ao estado desejado
- Se há desviação, toma ações corretivas
- Funciona em **loops infinitos** (reconciliation loops)

### 2.3 Processamento e Orquestração

- Processa requisições de recursos
- Orquestra a criação e gerenciamento de recursos
- Comunica com Kube-API-Server para ler e atualizar estado

### 2.4 Automação

- Automatiza tarefas repetitivas
- Redimensiona deployments
- Gerencia replicação de pods
- Renova certificados automaticamente

---

## 3. Arquitetura: Controllers (Controladores)

O Kube Controller Manager **não é um único processo**, mas sim um **conjunto de controladores independentes** que trabalham em conjunto.

### 3.1 Controllers Principais

```
┌──────────────────────────────────────────────────────┐
│         Kube Controller Manager                      │
│  (Conjunto de Controllers)                           │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Deployment Controller                         │ │
│  │  - Monitora deployments                        │ │
│  │  - Garante número correto de replicas          │ │
│  │  - Gerencia rolling updates                    │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  ReplicaSet Controller                         │ │
│  │  - Gerencia replicasets                        │ │
│  │  - Mantém número desejado de pods              │ │
│  │  - Cria/deleta pods automaticamente            │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  StatefulSet Controller                        │ │
│  │  - Gerencia statefulsets                       │ │
│  │  - Mantém identidade estável                   │ │
│  │  - Gerencia ordem de criação                   │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  DaemonSet Controller                          │ │
│  │  - Garante um pod por node                     │ │
│  │  - Executa em todos os nós                     │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Job Controller                                │ │
│  │  - Gerencia jobs únicos                        │ │
│  │  - Garante conclusão bem-sucedida              │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  CronJob Controller                            │ │
│  │  - Schedula jobs periodicamente                │ │
│  │  - Gerencia agendamento                        │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Node Controller                               │ │
│  │  - Monitora saúde dos nós                      │ │
│  │  - Remove pods de nós indisponíveis            │ │
│  │  - Gerencia ciclo de vida dos nós              │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Service Account Controller                    │ │
│  │  - Cria service accounts padrão                │ │
│  │  - Gerencia credenciais                        │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Namespace Controller                          │ │
│  │  - Gerencia ciclo de vida de namespaces        │ │
│  │  - Limpa recursos ao deletar namespace         │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  PersistentVolume Controller                   │ │
│  │  - Gerencia volumes persistentes               │ │
│  │  - Controla ciclo de vida de PVs               │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  E muitos outros... (~30+ controllers)         │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
└──────────────────────────────────────────────────────┘

```

---

## 4. Como Funciona: Reconciliation Loop

Cada controller executa um **loop infinito de reconciliação**:

### 4.1 Fluxo do Reconciliation Loop

```
┌─────────────────────────────────────────────────┐
│      Reconciliation Loop (Contínuo)             │
│                                                 │
│  1. 👀 OBSERVAR                                 │
│     ├─ Monitorar estado atual                  │
│     ├─ Detectar mudanças                       │
│     └─ Ler eventos do cluster                  │
│                                                 │
│  2. 📋 COMPARAR                                 │
│     ├─ Comparar estado atual vs desejado       │
│     ├─ Identificar diferenças                  │
│     └─ Determinar ações necessárias            │
│                                                 │
│  3. ⚙️  AGIR                                    │
│     ├─ Se há diferença, tomar ação             │
│     ├─ Criar/atualizar/deletar recursos        │
│     ├─ Comunicar via Kube-API-Server           │
│     └─ Persistir mudanças no ETCD              │
│                                                 │
│  4. 🔄 REPETIR                                  │
│     └─ Volta ao passo 1 continuamente          │
│                                                 │
└─────────────────────────────────────────────────┘

```

### 4.2 Exemplo Prático: Deployment Controller

```yaml
# Estado Desejado (YAML que você criou)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 3  # ← Quer 3 pods
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest

```

**O que o Deployment Controller faz:**

```
Ciclo 1:
┌─────────────────────────┐
│ Observar                │
│ Pods atuais: 0          │  ← Menos do que o desejado (3)
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Comparar                │
│ Desejado: 3             │
│ Atual: 0                │
│ Ação: Criar 3 pods      │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Agir                    │
│ ✓ Criar pod-1           │
│ ✓ Criar pod-2           │
│ ✓ Criar pod-3           │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Estado após ação        │
│ Pods atuais: 3 ✓        │  ← Corresponde ao desejado
└─────────────────────────┘

Ciclo 2 (alguns minutos depois):
┌─────────────────────────┐
│ Observar                │
│ Pods atuais: 2          │  ← Um pod foi deletado! (problema)
│ Pod-2 está Down         │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Comparar                │
│ Desejado: 3             │
│ Atual: 2                │
│ Ação: Criar 1 pod       │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Agir                    │
│ ✓ Criar novo pod-4      │  ← Repõe o pod que saiu
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Estado após ação        │
│ Pods atuais: 3 ✓        │  ← Novamente no estado desejado
└─────────────────────────┘

```

---

## 5. Controllers Importantes Detalhados

### 5.1 Deployment Controller

**Responsabilidades:**

- Monitora deployments
- Cria/atualiza ReplicaSets
- Gerencia rolling updates
- Rollback automático em caso de erro

**Exemplo:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meu-app
spec:
  replicas: 3  # ← Deployment Controller garante isso
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Máximo 1 pod extra durante update
      maxUnavailable: 0  # Nunca deixar 0 pods disponíveis

```

### 5.2 ReplicaSet Controller

**Responsabilidades:**

- Mantém o número exato de pods
- Cria novos pods se alguns forem deletados
- Deleta pods extras

### 5.3 Node Controller

**Responsabilidades:**

- Monitora saúde dos nós
- Remove pods de nós indisponíveis
- Marca nós como não prontos/indisponíveis

**Exemplo de Ação:**

```
Nó se desconecta → Node Controller detecta
    ↓
Marca nó como "NotReady" → Pods começam a ser removidos
    ↓
ReplicaSet Controller detecta pods faltando
    ↓
Cria pods de reposição em nós saudáveis

```

### 5.4 Job Controller

**Responsabilidades:**

- Gerencia jobs que rodam uma única vez
- Garante conclusão bem-sucedida
- Faz retry em caso de falha

**Exemplo:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool:latest
  backoffLimit: 3  # ← Job Controller refaz até 3 vezes em caso de falha

```

### 5.5 CronJob Controller

**Responsabilidades:**

- Schedula jobs periodicamente
- Gerencia histórico de execução
- Limpa jobs antigos automaticamente

**Exemplo:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-job
spec:
  schedule: "0 2 * * *"  # ← Executa todo dia às 2 da manhã
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: cleanup-tool:latest

```

### 5.6 Namespace Controller

**Responsabilidades:**

- Gerencia ciclo de vida de namespaces
- Limpa recursos quando namespace é deletado
- Aplica políticas de finalização

---

## 6. Comunicação com Kube-API-Server

O Kube Controller Manager funciona **através do Kube-API-Server**:

### 6.1 Fluxo de Comunicação

```
┌──────────────────────────────┐
│  Kube Controller Manager     │
└────────────┬─────────────────┘
             │
             │ Requisições HTTPS
             │ (Autenticado)
             ▼
┌──────────────────────────────┐
│  Kube-API-Server             │
│                              │
│  1. Autentica requisição     │
│  2. Autoriza ação (RBAC)     │
│  3. Valida dados             │
│  4. Persiste no ETCD         │
│  5. Retorna confirmação      │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  ETCD (Banco de Dados)       │
└──────────────────────────────┘

```

### 6.2 Operações Típicas via API

```bash
# Controller monitora recursos (Watch)
kubectl get deployments --watch

# Controller cria novos pods
kubectl create pod pod-1 --image=nginx

# Controller atualiza status
kubectl patch pod pod-1 -p '{"status":{"phase":"Running"}}'

# Controller deleta pods
kubectl delete pod pod-2

```

---

## 7. Estado Desejado vs Estado Atual

### 7.1 Exemplo: Deployment com 3 Replicas

```
┌─────────────────────────┐
│  Estado Desejado        │
│  (Arquivo YAML)         │
│                         │
│  Deployment: nginx-app  │
│  Replicas: 3            │
│  Image: nginx:1.21      │
│  Labels: app=nginx      │
└────────────┬────────────┘
             │
             │ Salvo no ETCD
             ▼
┌─────────────────────────┐
│  Estado Atual           │
│  (No Cluster)           │
│                         │
│  ReplicaSet-1           │
│  ├─ pod-1 (Running)     │
│  ├─ pod-2 (Running)     │
│  └─ pod-3 (Failed)      │  ← DIFERENÇA! Pod falhou
└────────────┬────────────┘
             │
             │ Controller detecta
             │ e toma ação
             ▼
┌─────────────────────────┐
│  Ação do Controller     │
│                         │
│  ✓ Deletar pod-3        │
│  ✓ Criar pod-4          │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Novo Estado Atual      │
│                         │
│  ReplicaSet-1           │
│  ├─ pod-1 (Running)     │
│  ├─ pod-2 (Running)     │
│  └─ pod-4 (Running)     │  ← Novamente em equilíbrio
└─────────────────────────┘

```

---

## 8. Exemplos de Atuações do Controller Manager

### 8.1 Caso 1: Pod Deletado Acidentalmente

```
Estado Desejado: 3 pods nginx
                  ↓
[Pod-1] [Pod-2] [Pod-3]

Ação acidental: usuario deleta Pod-2
                  ↓
[Pod-1] [X] [Pod-3]

Controller detecta: Apenas 2 pods (menos que os 3 desejados)
                  ↓
Controller cria: Novo pod-4
                  ↓
[Pod-1] [Pod-4] [Pod-3]  ← Estado desejado restaurado!

```

### 8.2 Caso 2: Nó Indisponível

```
Estado Desejado: 3 pods nginx
                  ↓
Nó-A: [Pod-1] [Pod-2]
Nó-B: [Pod-3]

Nó-A fica offline (falha de hardware)
                  ↓
Nó-A: [X] [X] (inacessível)
Nó-B: [Pod-3]

Node Controller detecta: Nó-A offline
                  ↓
Remove pods do Nó-A
                  ↓
ReplicaSet Controller detecta: Apenas 1 pod (menos que 3)
                  ↓
Cria pods de reposição em Nó-B
                  ↓
Nó-B: [Pod-3] [Pod-4] [Pod-5]  ← Estado desejado restaurado!

```

### 8.3 Caso 3: Atualização de Imagem

```
Estado Desejado Anterior: 3 pods com nginx:1.20
                  ↓
[Pod-1:1.20] [Pod-2:1.20] [Pod-3:1.20]

Atualização: Mudar para nginx:1.21
                  ↓
Estado Desejado Novo: 3 pods com nginx:1.21

Deployment Controller inicia Rolling Update:
Ciclo 1: Criar pod-4:1.21, deletar pod-1:1.20
                  ↓
[X] [Pod-2:1.20] [Pod-3:1.20] [Pod-4:1.21]

Ciclo 2: Criar pod-5:1.21, deletar pod-2:1.20
                  ↓
[X] [X] [Pod-3:1.20] [Pod-4:1.21] [Pod-5:1.21]

Ciclo 3: Criar pod-6:1.21, deletar pod-3:1.20
                  ↓
[Pod-4:1.21] [Pod-5:1.21] [Pod-6:1.21]  ← Todos atualizados!

```

---

## 9. Configuração do Kube Controller Manager

```bash
kube-controller-manager \
  --kubeconfig=/etc/kubernetes/controller-manager.conf \
  --master=https://127.0.0.1:6443 \
  --leader-elect=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --node-monitor-grace-period=40s \
  --node-monitor-period=5s \
  --pod-eviction-timeout=5m0s \
  --v=2

```

**Flags importantes:**

- `-leader-elect`: Apenas uma instância processa (alta disponibilidade)
- `-controllers`: Quais controllers ativar
- `-node-monitor-grace-period`: Tempo antes de marcar nó como indisponível
- `-pod-eviction-timeout`: Tempo antes de remover pods de nó offline

---

## 10. Monitoramento e Observabilidade

### 10.1 Ver Logs do Controller Manager

```bash
# Em um cluster local
kubectl logs -n kube-system deployment/kube-controller-manager

# Ver eventos do cluster
kubectl get events -A --sort-by='.lastTimestamp'

# Ver status de um deployment
kubectl rollout status deployment/meu-app

# Ver histórico de rollouts
kubectl rollout history deployment/meu-app

```

### 10.2 Métricas

```bash
# Obter métricas do controller manager
curl localhost:8080/metrics

```

---

## 11. Troubleshooting

### Erro: Pods não estão sendo criados

```
Possível causa: Controller Manager offline
Solução: Verificar status do controller manager
kubectl get pod -n kube-system kube-controller-manager-master

```

### Erro: Deployment está travado

```
Possível causa: Imagem inválida, recursos insuficientes
Solução: Verificar eventos
kubectl describe deployment meu-app

```

### Erro: Pods em ErrImagePull

```
Possível causa: Imagem não encontrada
Solução: Controller não conseguiu criar pod com essa imagem
kubectl logs <pod-id> para ver erro

```

---

## 12. Resumo das Responsabilidades

| Responsabilidade | Controller | Descrição |
| --- | --- | --- |
| **Replicas** | ReplicaSet | Mantém número correto de pods |
| **Deployments** | Deployment | Gerencia rolling updates |
| **StatefulSets** | StatefulSet | Mantém identidade e ordem |
| **DaemonSets** | DaemonSet | Garante 1 pod por node |
| **Jobs** | Job | Garante conclusão bem-sucedida |
| **CronJobs** | CronJob | Executa jobs periodicamente |
| **Nós** | Node | Monitora saúde dos nós |
| **Namespaces** | Namespace | Limpa recursos de namespace deletado |

---

## 13. Diagrama Completo de Interação

```
┌────────────────────────────────────────────────────────┐
│         Usuário ou Aplicação                           │
│  (cria deployment, atualiza imagem, etc)               │
└───────────────┬──────────────────────────────────────┘
                │
                │ kubectl apply -f deployment.yaml
                │
                ▼
        ┌──────────────────┐
        │  Kube-API-Server │
        │  (persistir YAML)│
        └────────┬─────────┘
                 │
                 │ Salva em ETCD
                 │
                 ▼
        ┌──────────────────┐
        │ ETCD             │
        │ (Banco de Dados) │
        └────────┬─────────┘
                 │
                 │ Notifica mudança
                 │
                 ▼
    ┌────────────────────────────┐
    │ Kube Controller Manager    │
    │                            │
    │ Deployment Controller:     │
    │ ├─ Lê novo deployment      │
    │ ├─ Cria ReplicaSet         │
    │ └─ Comunica com API-Server │
    │                            │
    │ ReplicaSet Controller:     │
    │ ├─ Lê ReplicaSet           │
    │ ├─ Cria 3 pods             │
    │ └─ Comunica com API-Server │
    └────────┬───────────────────┘
             │
             │ Requisições create pod
             │
             ▼
    ┌──────────────────┐
    │  Kube-API-Server │
    │  (criar pods)    │
    └────────┬─────────┘
             │
             │ ETCD atualizado
             │
             ▼
    ┌────────────────────┐
    │ Scheduler          │
    │ (Coloca pods nos   │
    │  nós apropriados)  │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ Kubelet (cada nó)  │
    │ (executa pods)     │
    └────────────────────┘

```

---

## 14. Conclusão

- **Kube Controller Manager** é o "maestro" do Kubernetes
- Executa **loops infinitos de reconciliação** para cada controller
- **Monitora continuamente** o estado do cluster
- **Toma ações automáticas** para manter o estado desejado
- Comunica **apenas via Kube-API-Server**
- Garante **alta disponibilidade** e **auto-recuperação** do cluster
- É **essencial** para o funcionamento automático do Kubernetes

---

