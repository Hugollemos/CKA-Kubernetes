# Scheduler Profiles

> ❌ **FORA DO ESCOPO CKA** — Scheduler Profiles são detalhamento avançado e **não são cobrados na prova**. Este arquivo é apenas referência extra. Não gaste tempo de estudo aqui.

## 📋 O que são Scheduler Profiles?

**Scheduler Profiles** permitem configurar **múltiplos perfis de scheduling** em um único kube-scheduler. Cada perfil pode ter diferentes plugins e configurações, permitindo comportamentos distintos de scheduling para diferentes tipos de workloads.

### Características principais:
- ✅ Múltiplos perfis em um **único scheduler**
- ✅ Cada perfil pode ter **plugins diferentes** habilitados/desabilitados
- ✅ Pods escolhem o perfil via campo **`schedulerName`**
- ✅ Permite **customização** do comportamento de scheduling
- ✅ Mais eficiente que rodar **múltiplos schedulers** separados

## 🎯 Por que usar Scheduler Profiles?

### Cenários comuns:

1. **Diferentes estratégias de scheduling**
   - Perfil para workloads de GPU (com plugin de bin-packing)
   - Perfil para workloads distribuídos (com plugin de spreading)

2. **Compliance e regulamentação**
   - Perfil que garante pods em zonas específicas
   - Perfil que respeita restrições de data sovereignty

3. **Otimização de recursos**
   - Perfil que prioriza consolidação (economia)
   - Perfil que prioriza distribuição (alta disponibilidade)

4. **Multi-tenancy**
   - Perfil para equipe A (com suas regras)
   - Perfil para equipe B (com regras diferentes)

## 📊 Arquitetura do Scheduler

```
┌────────────────────────────────────────────────────────┐
│  Kube-Scheduler                                        │
│                                                        │
│  ┌──────────────────────────────────────────────┐    │
│  │  Scheduling Queue                            │    │
│  │  (Pods aguardando scheduling)                │    │
│  └────────────────┬─────────────────────────────┘    │
│                   │                                    │
│                   ↓                                    │
│  ┌──────────────────────────────────────────────┐    │
│  │  Profile Selector                            │    │
│  │  (Escolhe profile baseado em schedulerName)  │    │
│  └────────────────┬─────────────────────────────┘    │
│                   │                                    │
│       ┌───────────┴───────────┬───────────────┐      │
│       ↓                       ↓               ↓      │
│  ┌─────────┐           ┌─────────┐     ┌─────────┐  │
│  │Profile 1│           │Profile 2│     │Profile 3│  │
│  │default  │           │gpu-sched│     │custom   │  │
│  │         │           │         │     │         │  │
│  │Plugins: │           │Plugins: │     │Plugins: │  │
│  │- Filter │           │- Filter │     │- Custom │  │
│  │- Score  │           │- Score  │     │- Filter │  │
│  └─────────┘           └─────────┘     └─────────┘  │
└────────────────────────────────────────────────────────┘
```

## 🔌 Plugins do Scheduler

O scheduler do Kubernetes funciona através de **plugins** que são executados em diferentes fases:

### Fases do Scheduling (Extension Points)

```
Pod → Queue → PreFilter → Filter → PostFilter → PreScore → Score → NormalizeScore → Reserve → Permit → PreBind → Bind → PostBind
```

### Principais Plugins

| Plugin | Fase | Função |
|--------|------|--------|
| **NodeResourcesFit** | Filter, Score | Verifica se nó tem recursos suficientes |
| **NodeName** | Filter | Verifica se pod especifica um nodeName |
| **NodeUnschedulable** | Filter | Filtra nós marcados como unschedulable |
| **TaintToleration** | Filter, Score | Verifica tolerations vs taints |
| **NodeAffinity** | Filter, Score | Aplica node affinity rules |
| **PodTopologySpread** | PreFilter, Filter, Score | Distribui pods entre topologias |
| **InterPodAffinity** | PreFilter, Filter, Score | Aplica pod affinity/anti-affinity |
| **VolumeBinding** | Filter, Score | Verifica disponibilidade de volumes |
| **ImageLocality** | Score | Prefere nós que já têm a imagem |
| **DefaultBinder** | Bind | Faz o bind do pod ao nó |

## 📝 Configurando Scheduler Profiles

### Estrutura de Configuração

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler    # Profile padrão
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: ImageLocality
        weight: 1
      disabled:
      - name: "*"    # Desabilita todos os outros plugins de score
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated    # Prefere nós com menos recursos usados
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1

- schedulerName: gpu-scheduler    # Profile customizado para GPUs
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 2
      - name: NodeAffinity
        weight: 1
    filter:
      enabled:
      - name: NodeResourcesFit
      - name: NodeAffinity
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated    # Bin-packing: prefere nós mais cheios
        resources:
        - name: nvidia.com/gpu
          weight: 10
```

## 🚀 Usando Scheduler Profiles

### 1. Pod usando o scheduler padrão

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-default
spec:
  # schedulerName não especificado = usa "default-scheduler"
  containers:
  - name: nginx
    image: nginx:1.27
```

### 2. Pod usando um scheduler profile customizado

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  schedulerName: gpu-scheduler    # ← Usa o profile "gpu-scheduler"
  containers:
  - name: pytorch
    image: pytorch/pytorch:latest
    resources:
      limits:
        nvidia.com/gpu: 1
```

### 3. Deployment com scheduler customizado

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: batch-processor
spec:
  replicas: 5
  selector:
    matchLabels:
      app: batch
  template:
    metadata:
      labels:
        app: batch
    spec:
      schedulerName: batch-scheduler    # ← Todos os pods usam este scheduler
      containers:
      - name: processor
        image: batch-processor:1.0
```

## 🧪 Exemplo Prático: Múltiplos Profiles

### Cenário: Cluster com diferentes tipos de workload

```yaml
# scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:

# Profile 1: Default (balanced)
- schedulerName: default-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: PodTopologySpread
        weight: 2
      - name: ImageLocality
        weight: 1
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1

# Profile 2: GPU workloads (bin-packing)
- schedulerName: gpu-scheduler
  plugins:
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 5
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated    # Bin-packing para GPUs
        resources:
        - name: nvidia.com/gpu
          weight: 10
        - name: cpu
          weight: 1
        - name: memory
          weight: 1

# Profile 3: High availability (spreading)
- schedulerName: ha-scheduler
  plugins:
    score:
      enabled:
      - name: PodTopologySpread
        weight: 10
      - name: NodeResourcesFit
        weight: 1
  pluginConfig:
  - name: PodTopologySpread
    args:
      defaultConstraints:
      - maxSkew: 1
        topologyKey: kubernetes.io/hostname
        whenUnsatisfiable: ScheduleAnyway
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
```

### Aplicando a configuração

```bash
# 1. Criar ConfigMap com a configuração
kubectl create configmap scheduler-config \
  --from-file=scheduler-config.yaml \
  -n kube-system

# 2. Modificar o kube-scheduler para usar a config
# (Geralmente feito via manifest estático em /etc/kubernetes/manifests/)

# 3. Verificar se scheduler está usando os profiles
kubectl logs -n kube-system kube-scheduler-<node-name> | grep -i profile
```

## 🔍 Verificando Scheduler Profiles

### Ver eventos de scheduling

```bash
# Ver eventos de scheduling de um pod
kubectl describe pod <pod-name> | grep -A 5 "Events:"

# Ver qual scheduler agendou o pod
kubectl get pod <pod-name> -o jsonpath='{.spec.schedulerName}'

# Ver todos os pods e seus schedulers
kubectl get pods -A -o custom-columns=\
NAME:.metadata.name,\
NAMESPACE:.metadata.namespace,\
SCHEDULER:.spec.schedulerName,\
NODE:.spec.nodeName
```

### Ver logs do scheduler

```bash
# Ver logs do kube-scheduler
kubectl logs -n kube-system kube-scheduler-<controlplane-node> --tail=100

# Filtrar por profile específico
kubectl logs -n kube-system kube-scheduler-<controlplane-node> | grep "gpu-scheduler"

# Ver decisões de scheduling
kubectl logs -n kube-system kube-scheduler-<controlplane-node> | grep -i "schedule"
```

## 📊 Tipos de Scoring Strategies

### LeastAllocated (Padrão)
```yaml
scoringStrategy:
  type: LeastAllocated
```
- **Comportamento**: Prefere nós com **menos recursos utilizados**
- **Uso**: Workloads normais, distribuição equilibrada
- **Resultado**: Pods espalhados pelos nós (spreading)

### MostAllocated (Bin-packing)
```yaml
scoringStrategy:
  type: MostAllocated
```
- **Comportamento**: Prefere nós com **mais recursos utilizados**
- **Uso**: GPUs, licenças caras, consolidação
- **Resultado**: Pods concentrados em menos nós (packing)

### RequestedToCapacityRatio
```yaml
scoringStrategy:
  type: RequestedToCapacityRatio
  requestedToCapacityRatio:
    shape:
    - utilization: 0
      score: 10
    - utilization: 100
      score: 0
```
- **Comportamento**: Scoring customizado baseado em utilização
- **Uso**: Casos avançados com necessidades específicas
- **Resultado**: Controle fino sobre distribuição

## 🎯 Casos de Uso Práticos

### Caso 1: Cluster com GPUs (Bin-packing)

**Problema**: GPUs são caras, queremos maximizar uso de cada nó com GPU.

**Solução**:
```yaml
- schedulerName: gpu-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
        - name: nvidia.com/gpu
          weight: 10
```

**Pods**:
```yaml
spec:
  schedulerName: gpu-scheduler
  containers:
  - name: ml-training
    resources:
      limits:
        nvidia.com/gpu: 1
```

### Caso 2: Alta Disponibilidade (Spreading)

**Problema**: Queremos pods espalhados por zonas para tolerância a falhas.

**Solução**:
```yaml
- schedulerName: ha-scheduler
  pluginConfig:
  - name: PodTopologySpread
    args:
      defaultConstraints:
      - maxSkew: 1
        topologyKey: topology.kubernetes.io/zone
        whenUnsatisfiable: DoNotSchedule
```

**Pods**:
```yaml
spec:
  schedulerName: ha-scheduler
  replicas: 6    # Distribuídos por 3 zonas = 2 por zona
```

### Caso 3: Batch Jobs (Recursos sobrando)

**Problema**: Jobs batch devem usar nós com recursos disponíveis.

**Solução**:
```yaml
- schedulerName: batch-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: LeastAllocated
        resources:
        - name: cpu
          weight: 2
        - name: memory
          weight: 1
```

**Jobs**:
```yaml
spec:
  schedulerName: batch-scheduler
  containers:
  - name: batch-processor
    resources:
      requests:
        cpu: 2
        memory: 4Gi
```

## 🛠️ Comandos Úteis

### Ver scheduler configuration

```bash
# Ver configuração do kube-scheduler (static pod)
kubectl get pod -n kube-system kube-scheduler-<node> -o yaml

# Ver configmap de configuração (se usando)
kubectl get configmap -n kube-system scheduler-config -o yaml
```

### Testar scheduling

```bash
# Criar pod com scheduler específico
kubectl run test-pod --image=nginx --dry-run=client -o yaml > test-pod.yaml

# Adicionar schedulerName
cat <<EOF >> test-pod.yaml
spec:
  schedulerName: gpu-scheduler
EOF

# Aplicar e observar
kubectl apply -f test-pod.yaml
kubectl get pod test-pod -o wide --watch
```

### Debug de scheduling

```bash
# Ver por que pod não foi agendado
kubectl describe pod <pod-name> | grep -A 20 "Events:"

# Ver todos os pods pending
kubectl get pods -A --field-selector status.phase=Pending

# Ver eventos de scheduling failures
kubectl get events --sort-by='.lastTimestamp' | grep -i "FailedScheduling"
```

## ⚠️ Considerações Importantes

### 1. schedulerName deve existir

```yaml
# ❌ ERRO: scheduler não existe
spec:
  schedulerName: non-existent-scheduler

# Pod fica em Pending com evento:
# "0/3 nodes are available: 3 node(s) didn't match scheduler name."
```

**Solução**: Verificar schedulers disponíveis:
```bash
# Ver profiles configurados nos logs do scheduler
kubectl logs -n kube-system kube-scheduler-<node> | grep -i "profile"
```

### 2. Mudança de scheduler profile requer reinício

```bash
# Modificar scheduler config
kubectl edit configmap scheduler-config -n kube-system

# Reiniciar scheduler (se static pod)
kubectl delete pod -n kube-system kube-scheduler-<node>
# Kubelet recria automaticamente

# Ou editar o manifest
vi /etc/kubernetes/manifests/kube-scheduler.yaml
```

### 3. Compatibilidade de plugins

Nem todos os plugins funcionam bem juntos. Teste antes de usar em produção.

### 4. Performance

Múltiplos profiles em um scheduler é mais eficiente que múltiplos schedulers separados.

## 📚 Default Scheduler Behavior

Se você **não** especificar nenhum profile, o scheduler usa configuração padrão:

```yaml
# Configuração padrão implícita
profiles:
- schedulerName: default-scheduler
  plugins:
    queueSort:
      enabled:
      - name: PrioritySort
    preFilter:
      enabled:
      - name: NodeResourcesFit
      - name: NodePorts
      - name: PodTopologySpread
      - name: InterPodAffinity
      - name: VolumeBinding
    filter:
      enabled:
      - name: NodeUnschedulable
      - name: NodeName
      - name: TaintToleration
      - name: NodeAffinity
      - name: NodePorts
      - name: NodeResourcesFit
      - name: VolumeRestrictions
      - name: EBSLimits
      - name: GCEPDLimits
      - name: NodeVolumeLimits
      - name: AzureDiskLimits
      - name: VolumeBinding
      - name: VolumeZone
      - name: PodTopologySpread
      - name: InterPodAffinity
    postFilter:
      enabled:
      - name: DefaultPreemption
    preScore:
      enabled:
      - name: InterPodAffinity
      - name: PodTopologySpread
      - name: TaintToleration
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: InterPodAffinity
        weight: 1
      - name: NodeAffinity
        weight: 1
      - name: TaintToleration
        weight: 1
      - name: ImageLocality
        weight: 1
      - name: PodTopologySpread
        weight: 2
    bind:
      enabled:
      - name: DefaultBinder
```

## 🧪 Exemplo Completo de Teste

```bash
# 1. Criar arquivo de configuração do scheduler
cat <<EOF > scheduler-config.yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- schedulerName: default-scheduler
- schedulerName: custom-scheduler
  pluginConfig:
  - name: NodeResourcesFit
    args:
      scoringStrategy:
        type: MostAllocated
        resources:
        - name: cpu
          weight: 1
EOF

# 2. Criar pod com scheduler padrão
kubectl run nginx-default --image=nginx

# 3. Criar pod com scheduler customizado
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: nginx-custom
spec:
  schedulerName: custom-scheduler
  containers:
  - name: nginx
    image: nginx
EOF

# 4. Verificar em qual nó cada pod foi agendado
kubectl get pods -o wide

# 5. Comparar decisões de scheduling
kubectl describe pod nginx-default | grep "Node:"
kubectl describe pod nginx-custom | grep "Node:"

# 6. Ver eventos de scheduling
kubectl get events --sort-by='.lastTimestamp' | grep -i "scheduled"
```

## 📖 Recursos para Estudo

### Documentação Oficial
- [Scheduler Configuration](https://kubernetes.io/docs/reference/scheduling/config/)
- [Scheduling Framework](https://kubernetes.io/docs/concepts/scheduling-eviction/scheduling-framework/)
- [Scheduler Plugins](https://kubernetes.io/docs/reference/scheduling/config/#profiles)
- [Multiple Schedulers](https://kubernetes.io/docs/tasks/extend-kubernetes/configure-multiple-schedulers/)

### Comandos Rápidos de Revisão

```bash
# Ver scheduler configuration
kubectl get pod -n kube-system kube-scheduler-<node> -o yaml

# Ver logs do scheduler
kubectl logs -n kube-system kube-scheduler-<node>

# Ver qual scheduler um pod usa
kubectl get pod <pod-name> -o jsonpath='{.spec.schedulerName}'

# Ver todos os pods e seus schedulers
kubectl get pods -A -o custom-columns=NAME:.metadata.name,SCHEDULER:.spec.schedulerName

# Ver eventos de scheduling
kubectl get events --field-selector involvedObject.name=<pod-name>

# Criar pod com scheduler específico
kubectl run mypod --image=nginx --dry-run=client -o yaml | \
  sed '/spec:/a\  schedulerName: custom-scheduler' | \
  kubectl apply -f -
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **O que são Scheduler Profiles**
   - Múltiplos perfis em um único scheduler
   - Cada perfil pode ter plugins diferentes

2. **Como especificar scheduler em um pod**
   ```yaml
   spec:
     schedulerName: custom-scheduler
   ```

3. **Ver qual scheduler um pod usa**
   ```bash
   kubectl get pod <name> -o jsonpath='{.spec.schedulerName}'
   ```

4. **Entender scheduling padrão**
   - Se não especificar `schedulerName`, usa "default-scheduler"

5. **Debug de scheduling failures**
   ```bash
   kubectl describe pod <name>
   kubectl get events | grep FailedScheduling
   ```

6. **Diferença entre scheduler profiles e múltiplos schedulers**
   - Profiles: mais eficiente, um único processo
   - Múltiplos schedulers: processos separados (menos comum)

### 🧪 Cenário típico na prova:

> **"Crie um pod chamado 'batch-job' usando a imagem 'busybox' que execute o comando 'sleep 3600'. O pod deve usar o scheduler chamado 'batch-scheduler'."**

**Solução:**
```bash
# Criar pod com scheduler específico
kubectl run batch-job --image=busybox --command -- sleep 3600 \
  --dry-run=client -o yaml > batch-job.yaml

# Editar para adicionar schedulerName
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: batch-job
spec:
  schedulerName: batch-scheduler
  containers:
  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
EOF

# Verificar
kubectl get pod batch-job -o jsonpath='{.spec.schedulerName}'
# Output: batch-scheduler
```

## 💡 Dicas para a Prova

1. **Sempre verifique se o scheduler existe**
   ```bash
   # Pod ficará Pending se scheduler não existir
   kubectl describe pod <name> | grep -i "didn't match scheduler name"
   ```

2. **schedulerName é imutável**
   - Não pode mudar depois que o pod é criado
   - Precisa deletar e recriar

3. **Use dry-run para gerar YAML**
   ```bash
   kubectl run mypod --image=nginx --dry-run=client -o yaml > pod.yaml
   # Depois adicione schedulerName manualmente
   ```

4. **Ver eventos para debug**
   ```bash
   kubectl get events --sort-by='.lastTimestamp'
   ```

5. **schedulerName não precisa de namespace**
   - É cluster-scoped
   - Funciona em qualquer namespace

---

⬅️ **Anterior**: [priority-class.md](./priority-class.md) | ➡️ **Próximo**: [scheduling.md](./scheduling.md)
