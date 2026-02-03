# Kube Scheduler - Guia Completo

## 1. O que é Kube Scheduler?

**Kube Scheduler** é um componente do Kubernetes responsável por **decidir em qual nó (máquina) cada pod será executado**. É o "distribuidor inteligente" do cluster, colocando os pods nos lugares certos.

### 1.1 Analogia

Imagine o Kube Scheduler como um **gerente de recursos de um hotel**:

- 🏨 Tem vários quartos (nós) disponíveis
- 👥 Chegam novos hóspedes (pods) para alojar
- 🔍 Analisa qual quarto é melhor para cada hóspede
- ✅ Aloca o hóspede ao quarto mais apropriado

---

## 2. Responsabilidades Principais

### 2.1 Seleção de Nós

- **Escolher qual nó** executará cada pod
- Considerar **recursos disponíveis** (CPU, memória)
- Levar em conta **restrições e preferências** do pod

### 2.2 Análise de Restrições

- **Verificar se o nó atende aos requisitos** do pod
- Considerar afinidade (node affinity)
- Considerar repulsão (pod affinity/anti-affinity)
- Validar tolerâncias (tolerations)

### 2.3 Distribuição Eficiente

- **Distribuir pods** de forma equilibrada
- Evitar **sobrecarga** em nós específicos
- Melhorar **utilização de recursos** do cluster
- Permitir **escalabilidade** do sistema

### 2.4 Otimização

- Considerar **locais onde dados já existem** (data locality)
- **Agrupar ou separar** pods conforme necessário
- Balancear **performance e utilização**

---

## 3. Fluxo do Processo de Scheduling

### 3.1 Quando o Scheduler Entra em Ação

```
1. 👤 Usuário cria um Pod/Deployment
   ↓
2. 📝 Kube-API-Server recebe e persiste no ETCD
   ↓
3. 🔍 Kube Controller Manager cria pods (se for deployment)
   ↓
4. 📌 Pod fica em estado "Pending" (aguardando scheduler)
   ↓
5. 🎯 Kube Scheduler detecta pod sem nó atribuído
   ↓
6. ⚙️  Scheduler analisa nós e escolhe o melhor
   ↓
7. ✅ Scheduler atribui o pod a um nó (via Kube-API-Server)
   ↓
8. 🚀 Kubelet no nó vê o pod e começa a executar
   ↓
9. 📦 Pod entra em estado "Running"

```

### 3.2 Estados do Pod no Processo

```
Pod Life Cycle:

         ┌─────────────┐
         │  Pending    │ ← Pod criado, aguardando scheduler
         └──────┬──────┘
                │ (Scheduler escolhe nó)
         ┌──────▼──────┐
         │  Container  │ ← Kubelet começa a executar
         │  Creating   │
         └──────┬──────┘
                │ (Container iniciado)
         ┌──────▼──────┐
         │  Running    │ ← Pod está executando
         └──────┬──────┘
                │ (Pod termina)
         ┌──────▼──────┐
         │  Succeeded  │ ← Completado com sucesso
         │  ou Failed  │   ou Falhou
         └─────────────┘

```

---

## 4. Arquitetura do Scheduler

### 4.1 Componentes Internos

```
┌──────────────────────────────────────────────────────┐
│         Kube Scheduler                               │
│                                                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  1. Informação do Pod                       │   │
│  │     - Requisitos de recursos (CPU, memória)│   │
│  │     - Restrições (nodeSelector)            │   │
│  │     - Tolerâncias (tolerations)            │   │
│  │     - Afinidade (affinity)                 │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  2. Filtragem (Filter Phase)               │   │
│  │     - Remove nós que NÃO podem executar    │   │
│  │     - Verifica recursos mínimos            │   │
│  │     - Valida tolerâncias                   │   │
│  │     - Verifica taints do nó                │   │
│  │     Result: Lista de nós "candidatos"     │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  3. Scoring (Scoring Phase)                │   │
│  │     - Avalia cada nó candidato             │   │
│  │     - Calcula score para cada nó           │   │
│  │     - Considera múltiplos fatores          │   │
│  │     - Plugins de priorização               │   │
│  │     Result: Nós ranqueados                │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  4. Seleção (Selection)                    │   │
│  │     - Escolhe nó com maior score           │   │
│  │     - Em caso de empate, escolhe aleatório │   │
│  │     Result: Nó final escolhido             │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  5. Binding (Vinculação)                   │   │
│  │     - Comunica com Kube-API-Server         │   │
│  │     - Atribui pod ao nó via ETCD           │   │
│  │     Result: Pod vinculado ao nó            │   │
│  └────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘

```

---

## 5. Filtragem (Filter Phase)

### 5.1 O que é Filtragem?

Na fase de filtragem, o scheduler **remove nós que NÃO podem executar o pod**.

### 5.2 Critérios de Filtragem

```
┌─────────────────────────────────────┐
│    Pod a Agendar                    │
│                                     │
│  Requisitos:                        │
│  - CPU: 500m                        │
│  - Memória: 256Mi                   │
│  - NodeSelector: disktype=ssd       │
│  - Tolerations: none                │
│  - Affinity: none                   │
└─────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────────┐
│  Fase de Filtragem                       │
│                                          │
│  Nó-1:                                   │
│  ├─ CPU disponível: 100m  ❌             │
│  │  (Insuficiente: 500m > 100m)          │
│  └─ ELIMINADO                            │
│                                          │
│  Nó-2:                                   │
│  ├─ CPU disponível: 1000m ✓              │
│  ├─ Memória disponível: 512Mi ✓          │
│  ├─ Disco: SSD ✓                         │
│  ├─ Não tainted ✓                        │
│  └─ CANDIDATO                            │
│                                          │
│  Nó-3:                                   │
│  ├─ CPU disponível: 800m ✓               │
│  ├─ Memória disponível: 512Mi ✓          │
│  ├─ Disco: HDD ❌                        │
│  │  (NodeSelector requer SSD)            │
│  └─ ELIMINADO                            │
│                                          │
│  Nó-4:                                   │
│  ├─ CPU disponível: 1500m ✓              │
│  ├─ Memória disponível: 1Gi ✓            │
│  ├─ Disco: SSD ✓                         │
│  ├─ Taint: NoExecute ❌                  │
│  │  (Sem tolerância para este taint)     │
│  └─ ELIMINADO                            │
│                                          │
│  Resultado: Apenas Nó-2 é candidato    │
└──────────────────────────────────────────┘

```

### 5.3 Predicados Comuns (Filtros)

| Filtro | Descrição | Exemplo |
| --- | --- | --- |
| **PodFitsResources** | Verifica CPU/Memória | Nó tem 512Mi, pod precisa 256Mi ✓ |
| **PodFitsHost** | Verifica port bindings | Porta já em uso ✗ |
| **PodSelectorMatches** | Verifica nodeSelector | nodeSelector: gpu=true |
| **NoDiskConflict** | Verifica volumes | Volumes não em conflito |
| **PodToleratesNodeTaints** | Verifica tolerações | Tolerations vs Taints |
| **NodeAffinity** | Verifica afinidade | Nó atende requerimentos |

---

## 6. Scoring (Scoring Phase)

### 6.1 O que é Scoring?

Na fase de scoring, o scheduler **avalia os nós candidatos** e dá uma pontuação a cada um. O nó com maior pontuação é escolhido.

### 6.2 Fatores de Scoring

```
Após filtragem, restam:
├─ Nó-2 ✓
├─ Nó-5 ✓
└─ Nó-7 ✓

Fase de Scoring:

┌────────────────────────────────┐
│  Nó-2                          │
├────────────────────────────────┤
│ • Recursos disponíveis: Alto   │ +80
│ • CPU livre: 500m de 1000m     │
│ • Preferência de zona: Mesma   │ +20
│ • Localidade de dados: Ótima   │ +30
├────────────────────────────────┤
│ TOTAL: 130 pontos              │
└────────────────────────────────┘

┌────────────────────────────────┐
│  Nó-5                          │
├────────────────────────────────┤
│ • Recursos disponíveis: Médio  │ +50
│ • CPU livre: 200m de 1000m     │
│ • Preferência de zona: Distante│ +10
│ • Localidade de dados: Ruim    │ +5
├────────────────────────────────┤
│ TOTAL: 65 pontos               │
└────────────────────────────────┘

┌────────────────────────────────┐
│  Nó-7                          │
├────────────────────────────────┤
│ • Recursos disponíveis: Médio  │ +50
│ • CPU livre: 100m de 1000m     │
│ • Preferência de zona: Mesma   │ +20
│ • Localidade de dados: Média   │ +15
├────────────────────────────────┤
│ TOTAL: 85 pontos               │
└────────────────────────────────┘

Resultado: Nó-2 vencedor (130 pontos) ✓

```

### 6.3 Plugins de Priorização Comuns

| Plugin | Descrição | Objetivo |
| --- | --- | --- |
| **LeastRequested** | Nós com menos recursos usados | Distribuir carga |
| **BalancedResourceAllocation** | Balancear CPU e memória | Uso uniforme |
| **NodePreferAvoidPods** | Evitar nós marcados | Graceful shutdown |
| **TaintToleration** | Preferir nós com taints compatíveis | Isolamento |
| **NodeAffinity** | Nós que batem afinidade | Controle fino |
| **PodAffinity** | Agrupar pods relacionados | Localidade |
| **PodAntiAffinity** | Separar pods diferentes | Disponibilidade |

---

## 7. Restrições e Preferências

### 7.1 NodeSelector (Simples)

**NodeSelector** é a forma mais simples de restringir em quais nós um pod pode rodar.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd
spec:
  nodeSelector:
    disktype: ssd  # ← Pod DEVE rodar em nó com esta label
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        cpu: 500m
        memory: 256Mi

```

**Como preparar o nó:**

```bash
# Adicionar label ao nó
kubectl label nodes nó-2 disktype=ssd

# Verificar
kubectl get nodes --show-labels

```

### 7.2 Node Affinity (Avançado)

**Node Affinity** oferece controle mais fino sobre placement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - nó-2
            - nó-3
            - nó-4
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: webapp
    image: webapp:latest

```

**Tipos:**

- `requiredDuringSchedulingIgnoredDuringExecution` - **DEVE** atender (hard)
- `preferredDuringSchedulingIgnoredDuringExecution` - **PREFERE** atender (soft)

### 7.3 Taints e Tolerations

**Taints** marcam nós como indisponíveis para certos pods. **Tolerations** permitem que pods ignorem taints.

```yaml
# Marcar nó com taint (via CLI)
kubectl taint nodes nó-gpu gpu=true:NoSchedule

# Pod com toleração
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: gpu-worker
    image: gpu-worker:latest

```

**Efeitos de Taints:**

- `NoSchedule` - Não agenda novos pods (existentes continuam)
- `PreferNoSchedule` - Prefere não agendar (soft)
- `NoExecute` - Remove pods existentes (hard)

### 7.4 Pod Affinity / Anti-Affinity

Controla se pods devem estar **juntos ou separados**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - database
        topologyKey: kubernetes.io/hostname
  containers:
  - name: frontend
    image: frontend:latest

```

**Interpretação:**

- Este frontend **DEVE** rodar no **mesmo nó** que o pod com label `app=database`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-cache
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - database
          topologyKey: kubernetes.io/hostname
  containers:
  - name: cache
    image: cache:latest

```

**Interpretação:**

- Este cache **PREFERE** rodar em **nó diferente** do pod com label `app=database`

---

## 8. Exemplo Prático Completo

### 8.1 Cenário: Deployment Web

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      # 1. Requisitos de recursos
      containers:
      - name: web
        image: nginx:latest
        resources:
          requests:
            cpu: 250m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi

      # 2. Afinidade de nó (preferir nós SSD)
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd

      # 3. Anti-afinidade de pod (pods em nós diferentes)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web
              topologyKey: kubernetes.io/hostname

```

### 8.2 Processo de Scheduling para Este Deployment

```
1. Replicaset Controller cria 3 pods
   Pod-1, Pod-2, Pod-3 (todos com Pending)

2. Scheduler processa Pod-1:

   Filtragem:
   ├─ Nó-1: 250m + 128Mi de recursos ✓
   ├─ Nó-2: 250m + 128Mi de recursos ✓
   ├─ Nó-3: Indisponível (Nó offline) ✗
   └─ Nó-4: 250m + 128Mi de recursos ✓

   Scoring:
   ├─ Nó-1: disktype=hdd, score=50
   ├─ Nó-2: disktype=ssd, score=100
   └─ Nó-4: disktype=hdd, score=50

   Vencedor: Nó-2 ✓

3. Scheduler processa Pod-2:

   Filtragem: Nó-1 ✓, Nó-2 ✓, Nó-4 ✓

   Scoring:
   ├─ Nó-1: disktype=hdd (50) + não tem web (100) = 150
   ├─ Nó-2: disktype=ssd (100) + já tem web (0) = 100 ❌ Anti-afinidade
   └─ Nó-4: disktype=hdd (50) + não tem web (100) = 150

   Vencedor: Nó-1 (ou Nó-4, aleatório entre 150) ✓

4. Scheduler processa Pod-3:

   Filtragem: Nó-2 ✓, Nó-4 ✓

   Scoring:
   ├─ Nó-2: disktype=ssd (100) + já tem web (0) = 100
   └─ Nó-4: disktype=hdd (50) + não tem web (100) = 150

   Vencedor: Nó-4 ✓

Resultado Final:
├─ Pod-1: Nó-2 ✓ (SSD preferido)
├─ Pod-2: Nó-1 ✓ (Separado de Pod-1)
└─ Pod-3: Nó-4 ✓ (Separado dos outros)

Todos os 3 pods em nós diferentes (Anti-affinity respeitada)!

```

---

## 9. Algoritmos de Scheduling

### 9.1 Scheduler Framework (Padrão Atual)

```
Pod → Filter Plugins → Scoring Plugins → Nó Final

┌────────────────────────────────────────────┐
│  Pod com requisitos de scheduling          │
└────────────┬───────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────┐
│  Filter Plugins                            │
│  ├─ ResourceFit                            │
│  ├─ NodeUnschedulable                      │
│  ├─ NodeName                               │
│  ├─ TaintToleration                        │
│  ├─ NodeAffinity                           │
│  └─ PodAffinity (filtragem prévia)         │
└────────────┬───────────────────────────────┘
             │ (remove nós inadequados)
             ▼
┌────────────────────────────────────────────┐
│  Scoring Plugins                           │
│  ├─ NodeResourcesFit                       │
│  ├─ ImageLocality                          │
│  ├─ InterPodAffinity                       │
│  ├─ NodeAffinity                           │
│  ├─ TaintToleration                        │
│  └─ PodTopologySpread                      │
└────────────┬───────────────────────────────┘
             │ (ordena nós por score)
             ▼
┌────────────────────────────────────────────┐
│  Seleção Final                             │
│  (Nó com maior score)                      │
└────────────┬───────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────┐
│  Binding                                   │
│  (Vincular pod ao nó via API-Server)       │
└────────────────────────────────────────────┘

```

---

## 10. Configuração do Scheduler

### 10.1 Configuração Básica

```bash
kube-scheduler \
  --kubeconfig=/etc/kubernetes/scheduler.conf \
  --leader-elect=true \
  --v=2

```

### 10.2 Configuração com Arquivo Config

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- name: default
  plugins:
    preFilter:
      enabled:
      - name: DefaultPreemption
    filter:
      enabled:
      - name: NodeUnschedulable
      - name: NodeAffinity
      - name: TaintToleration
    postFilter:
      enabled:
      - name: DefaultPreemption
    preScore:
      enabled:
      - name: NodeAffinity
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: ImageLocality
        weight: 1
      - name: NodeAffinity
        weight: 1
    reserve:
      enabled:
      - name: VolumeBinding

```

---

## 11. Monitoramento e Troubleshooting

### 11.1 Ver Status de Pods Pendentes

```bash
# Ver pods não agendados
kubectl get pods -A --field-selector=status.phase=Pending

# Ver detalhes do pod (eventos)
kubectl describe pod meu-pod -n meu-namespace

# Ver logs do scheduler
kubectl logs -n kube-system -l component=kube-scheduler

```

### 11.2 Exemplos de Problemas

### Problema 1: Pod Não Consegue Ser Agendado

```
$ kubectl describe pod app-pod

Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  2m                 default-scheduler  no nodes available to schedule pods

```

**Causa possível**: Insuficiente recursos
**Solução**: Aumentar recursos do nó ou reduzir requisitos do pod

### Problema 2: Pod Aguardando NodeSelector

```
$ kubectl describe pod gpu-pod

Events:
  Warning  FailedScheduling  2m  default-scheduler  0/5 nodes are available: 5 node(s) didn't match Pod's node selector.

```

**Causa**: Label `gpu=true` não existe em nó algum
**Solução**: `kubectl label nodes nó-1 gpu=true`

### Problema 3: Pod com Taint Incompatível

```
$ kubectl describe pod normal-pod

Events:
  Warning  FailedScheduling  2m  default-scheduler  0/3 nodes are available: 3 node(s) have taints that the pod does not tolerate: {gpu=true:NoSchedule}

```

**Causa**: Nó tem taint, pod não tem toleração
**Solução**: Adicionar toleração ao pod ou remover taint

### 11.3 Comandos Úteis

```bash
# Ver recursos do nó
kubectl top nodes

# Ver recursos do pod
kubectl top pods

# Ver labels dos nós
kubectl get nodes --show-labels

# Ver taints dos nós
kubectl describe node nó-1 | grep Taints

# Simular scheduling (dry-run)
kubectl create pod test --image=nginx --dry-run=client -o yaml | kubectl apply -f - --dry-run=client

```

---

## 12. Otimizações Avançadas

### 12.1 Pod Topology Spread

Espalhar pods entre diferentes zonas/domínios:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-spread
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  containers:
  - name: web
    image: nginx:latest

```

**Interpretação:**

- Máximo 1 pod por zona (distribuído uniformemente)
- Se não conseguir distribuir, não agenda

### 12.2 Priority Classes

Definir prioridade de scheduling:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "Alto priority para apps críticas"
---
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: app:latest

```

---

## 13. Resumo de Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Filtragem** | Remove nós que não podem executar o pod |
| **Scoring** | Avalia nós candidatos e ordena por score |
| **Seleção** | Escolhe nó com maior score |
| **Binding** | Vincula pod ao nó via API-Server |
| **Otimização** | Distribui pods de forma eficiente |

---

## 14. Fluxo Completo do Cluster

```
┌─────────────────────────────────────────────────────────────┐
│  1. Usuário cria Deployment                                │
│     kubectl create deployment app --image=nginx             │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  2. Kube-API-Server recebe e persiste no ETCD              │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  3. Kube Controller Manager                                │
│     └─ ReplicaSet Controller cria 3 pods (Pending)         │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  4. Kube Scheduler                                          │
│     ├─ Filtra nós candidatos                               │
│     ├─ Scored nós                                          │
│     ├─ Escolhe melhor nó para cada pod                     │
│     └─ Vincula pod ao nó                                   │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  5. Kubelet (em cada nó)                                   │
│     ├─ Detecta novo pod agendado                           │
│     ├─ Inicia containers                                   │
│     └─ Pod entra em estado Running                         │
└─────────────────────────────────────────────────────────────┘

```

---

## 15. Comparação com Outros Schedulers

| Scheduler | Caso de Uso | Características |
| --- | --- | --- |
| **Default Scheduler** | Propósito geral | Equilibrado, plugin-based |
| **Custom Scheduler** | Casos específicos | Customizado, lógica própria |
| **Karpenter** | Escalamento eficiente | Otimizado para custo/performance |
| **Crane** | Economia de recursos | Bin-packing agressivo |

---

## 16. Conclusão

- **Kube Scheduler** é o "distribuidor inteligente" do cluster
- **Processa em 2 fases**: Filtragem e Scoring
- **Garante distribuição eficiente** de pods
- **Considera múltiplos fatores**: Recursos, afinidade, taints, etc.
- **Altamente configurável** via restrições e preferências
- É **essencial** para o funcionamento ótimo do Kubernetes

---

