# Scheduling

# Scheduling Manual no Kubernetes/EKS - Resumo para Estudos CKA

## O que é Scheduling Manual?

**Scheduling** é o processo de decidir em qual **nó** um Pod vai rodar. Normalmente o Kubernetes faz isso automaticamente, mas você pode forçar um Pod a rodar em um nó específico.

## Por Que Precisa?

```
Cenários comuns:
- Pod precisa de GPU (machine learning)
- Pod precisa de muita CPU/RAM
- Pod precisa estar em nó específico (SSD rápido)
- App precisa estar junto/separada de outra app
- Hardware especial (TPU, FPGA, etc)

```

---

## Formas de Fazer Scheduling Manual

### 1. nodeSelector (Mais Simples) ⭐

Força o Pod a rodar em nós com labels específicas.

### Passo 1: Etiquetar o Nó

```bash
# Adicionar label ao nó
kubectl label nodes node-1 tipo=gpu-ready

# Ver labels do nó
kubectl get nodes --show-labels
kubectl label nodes node-1 --list

```

### Passo 2: Usar nodeSelector no Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-pod
spec:
  nodeSelector:
    tipo: gpu-ready    # Pod roda APENAS em nós com essa label
  containers:
  - name: tensorflow
    image: tensorflow:latest
    resources:
      requests:
        nvidia.com/gpu: 1  # precisa de 1 GPU

```

### Criar via kubectl

```bash
kubectl run gpu-app --image=tensorflow --dry-run=client -o yaml > pod.yaml
# Editar e adicionar nodeSelector
kubectl create -f pod.yaml

```

### Verificar

```bash
# Ver em qual nó o Pod está rodando
kubectl get pods -o wide

# Descrever Pod para ver se foi agendado
kubectl describe pod ml-pod
# Procure por "Node:" e "Events"

```

---

### 2. nodeAffinity (Mais Poderoso) ⭐⭐

Mais flexível que nodeSelector. Permite múltiplas condições e preferências.

### Affinity Obrigatória (requiredDuringSchedulingIgnoredDuringExecution)

Pod **NÃO inicia** se a condição não for atendida.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
            - nvme
          - key: zona
            operator: In
            values:
            - us-east-1a
  containers:
  - name: web
    image: nginx:latest

```

**Operadores disponíveis:**

- `In`: valor está na lista
- `NotIn`: valor não está na lista
- `Exists`: key existe (não verifica valor)
- `DoesNotExist`: key não existe
- `Gt`: valor > número
- `Lt`: valor < número

### Affinity com Preferência (preferredDuringSchedulingIgnoredDuringExecution)

Pod **prefere** rodar nesse nó, mas pode rodar em outro se necessário.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100          # peso 1-100 (maior = mais preferência)
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 50
        preference:
          matchExpressions:
          - key: zona
            operator: In
            values:
            - us-east-1a
  containers:
  - name: redis
    image: redis:latest

```

### Combinando Obrigatória + Preferência

```yaml
spec:
  affinity:
    nodeAffinity:
      # Tem que cumprir ISSO
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tipo
            operator: In
            values:
            - producao
      # E prefere ISSO
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd

```

---

### 3. Pod Affinity/Anti-Affinity

Força um Pod estar junto com outro Pod (ou afastado).

### Pod Affinity (Junto)

Pod A prefere/requer estar no mesmo nó/zona que Pod B.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname  # mesmo nó
  containers:
  - name: app
    image: app:latest

```

**topologyKey:**

- `kubernetes.io/hostname`: mesmo nó
- `topology.kubernetes.io/zone`: mesma zona/AZ
- `topology.kubernetes.io/region`: mesma região

### Pod Anti-Affinity (Separado)

Pod A requer estar em nó diferente de Pod B.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-1
  labels:
    db: postgresql
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: db
            operator: In
            values:
            - postgresql
        topologyKey: kubernetes.io/hostname  # nó diferente
  containers:
  - name: postgres
    image: postgres:latest

```

**Use quando:**

- Não quer dois databases no mesmo nó
- Quer alta disponibilidade (distribuir entre nós)
- Quer latência baixa (próximo)

---

### 4. Taints e Tolerations

Força certos Pods a NÃO rodarem em um nó (ou só Pods específicos).

### Taint no Nó

```bash
# Adicionar taint ao nó
kubectl taint nodes node-1 tipo=gpu:NoSchedule
kubectl taint nodes node-1 tipo=gpu:NoExecute
kubectl taint nodes node-1 tipo=gpu:PreferNoSchedule

# Ver taints
kubectl describe node node-1 | grep Taints

# Remover taint
kubectl taint nodes node-1 tipo=gpu:NoSchedule-

```

**Efeitos:**

- `NoSchedule`: Não coloca Pod novo (mas Pods existentes continuam)
- `NoExecute`: Remove Pod imediatamente se não tolera
- `PreferNoSchedule`: Evita colocar, mas pode se necessário

### Toleration no Pod

Pod que **tolera** (aguenta) o taint pode rodar lá.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-task
spec:
  tolerations:
  - key: tipo
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: gpu-app
    image: gpu-app:latest

```

### Exemplo Prático

```bash
# 1. Marcar nó como GPU-only
kubectl taint nodes node-gpu tipo=gpu:NoSchedule

# 2. Pod normal tenta rodar
kubectl run normal-app --image=nginx
# Fica Pending (não consegue scheduling)

# 3. Pod com toleration consegue
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  tolerations:
  - key: tipo
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: app
    image: gpu-app:latest
EOF
# Roda normalmente

```

---

## Resumo Rápido: Quando Usar Cada Um

| Método | Complexidade | Uso | Quando |
| --- | --- | --- | --- |
| **nodeSelector** | Simples | Label simples no nó | Labels 1-2 condições |
| **nodeAffinity** | Médio | Multiple labels, operadores | Múltiplas condições, preferências |
| **Pod Affinity** | Médio-Alto | Pod próximo a outro Pod | Apps que precisam estar juntas |
| **Pod Anti-Affinity** | Médio-Alto | Pod longe de outro Pod | Alta disponibilidade, distribuir |
| **Taints/Tolerations** | Médio | Bloquear nós | Nós especializados (GPU, SSD) |

---

## Exemplos Práticos para CKA

### Exemplo 1: GPU para Machine Learning

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  nodeSelector:
    hardware: gpu
  containers:
  - name: tensorflow
    image: tensorflow:latest
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1

```

**Setup:**

```bash
kubectl label nodes node-gpu hardware=gpu
kubectl create -f ml-training.yaml
kubectl get pods -o wide  # verifica em qual nó está

```

---

### Exemplo 2: Database com Persistência

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    app: postgres
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: postgres
    image: postgres:latest
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql
  volumes:
  - name: data
    hostPath:
      path: /fast-ssd

```

---

### Exemplo 3: Alta Disponibilidade com Anti-Affinity

```yaml
# 3 replicas de database spread entre nós
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-cluster
spec:
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - db
            topologyKey: kubernetes.io/hostname  # nó diferente
      containers:
      - name: postgres
        image: postgres:latest

```

---

### Exemplo 4: Nó Especializado com Taints

```bash
# Marcar nó como especializado
kubectl taint nodes node-special reserved=only:NoSchedule

# Pod que precisa desse nó
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: special-workload
spec:
  tolerations:
  - key: reserved
    operator: Equal
    value: only
    effect: NoSchedule
  containers:
  - name: app
    image: special-app:latest
EOF

```

---

## Comandos Essenciais para CKA

```bash
# ========== LABELS ==========
# Adicionar label a nó
kubectl label nodes node-1 gpu=true

# Ver labels de nó
kubectl get nodes --show-labels
kubectl label nodes node-1 --list

# Remover label
kubectl label nodes node-1 gpu-

# ========== TAINTS ==========
# Adicionar taint
kubectl taint nodes node-1 gpu=true:NoSchedule

# Ver taints
kubectl describe node node-1 | grep Taints

# Remover taint
kubectl taint nodes node-1 gpu=true:NoSchedule-

# ========== SCHEDULING ==========
# Criar Pod com nodeSelector (via YAML)
kubectl apply -f pod-with-selector.yaml

# Ver em qual nó Pod está
kubectl get pods -o wide

# Descrever Pod (ver se foi agendado)
kubectl describe pod nome-pod

# Ver events de Pod (agendamento)
kubectl describe pod nome-pod | grep Events

# ========== DEBUGGING ==========
# Pod stuck em Pending?
kubectl describe pod nome-pod
# Procure por "node affinity", "taint", "status"

# Teste scheduling manualmente
kubectl create --dry-run=client -o yaml -f pod.yaml | kubectl apply -f -

```

---

## Troubleshooting Comum

### Pod em Pending (não agenda)

```bash
# Causa: nodeSelector não match
kubectl describe pod meu-pod
# Procure por: "no nodes match"

# Solução: verificar labels
kubectl get nodes --show-labels
kubectl label nodes node-1 gpu=true  # adicionar label

# Verificar taints
kubectl describe node node-1 | grep Taints
# Se houver taint, Pod precisa de toleration

```

### Pod em CrashLoopBackOff

```bash
# Pode ser taint com NoExecute (mata Pod)
kubectl describe pod meu-pod
# Procure por: "tainted"

# Verificar taints do nó onde Pod estava
kubectl describe node onde-pod-estava

# Remover taint se necessário
kubectl taint nodes node-1 chave=valor:NoExecute-

```

### Anti-Affinity Impede Replicação

```bash
# Problema: Pod prefere spread, mas menos nós que replicas
kubectl describe pod meu-pod
# Se diz "no nodes", anti-affinity está prorrogando

# Solução: usar preferredDuringScheduling ao invés de required
# Ou adicionar mais nós

```

---

## Pontos Importantes para Prova CKA

✅ **nodeSelector** - forma mais simples, labels diretas no nó
✅ **nodeAffinity** - forma mais poderosa, múltiplas condições
✅ **Pod Affinity** - colocar Pods juntos (mesma zona/nó)
✅ **Pod Anti-Affinity** - separar Pods (diferentes nós/zonas)
✅ **Taints/Tolerations** - bloquear nós + permitir exceções
✅ **topologyKey** - define nível: hostname, zone, region
✅ **requiredDuringScheduling** - obrigatório (Pod fica Pending se não quer)
✅ **preferredDuringScheduling** - preferência (Pod pode rodar em outro lugar)
✅ **NoSchedule vs NoExecute** - schedule impede novo, execute mata existente
✅ **kubectl label nodes** - adicionar labels para seleção
✅ **kubectl taint nodes** - marcar nó como especial
✅ Ver labels: `kubectl get nodes --show-labels`
✅ Ver taints: `kubectl describe node | grep Taints`
✅ Debugging: `kubectl describe pod` vê eventos de agendamento
✅ Pod Pending = problema de scheduling, ver describe pod

---

## Quick Reference

```bash
# Setup nó com label
kubectl label nodes node-1 hardware=gpu

# Pod simples (nodeSelector)
kubectl run gpu-app --image=app --dry-run=client -o yaml | \
  sed 's/containers:/nodeSelector:\n    hardware: gpu\n  containers:/' | \
  kubectl apply -f -

# Ver resultado
kubectl get pods -o wide

# Taint nó
kubectl taint nodes node-1 reserved=true:NoSchedule

# Pod com toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  tolerations:
  - key: reserved
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: app
    image: nginx
EOF

```

---

## Resumo em Uma Frase

