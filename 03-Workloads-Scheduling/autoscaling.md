# Autoscaling no Kubernetes

## 📋 O que é Autoscaling?

**Autoscaling** é a capacidade de ajustar automaticamente o número de recursos (pods ou nós) com base na demanda, garantindo **eficiência** e **disponibilidade**.

### Tipos de Autoscaling:

| Tipo | O que escala | Baseado em | Reinicia pods? |
|------|--------------|------------|----------------|
| **HPA** (Horizontal Pod Autoscaler) | Número de pods | Métricas (CPU, memória, custom) | Não (cria novos) |
| **VPA** (Vertical Pod Autoscaler) | Recursos do pod (CPU/memory) | Uso histórico | Sim |
| **In-Place Resizing** | Recursos do pod (CPU/memory) | Manual/controlado | Não (CPU), Mínimo (Memory) |
| **Cluster Autoscaler** | Número de nós | Pods pending | N/A |

## 🔄 Horizontal Pod Autoscaler (HPA)

### O que é HPA?

**HPA** automaticamente escala o número de **pods** em um Deployment, ReplicaSet ou StatefulSet baseado em métricas observadas.

### Como funciona:

```
┌─────────────────────────────────────────────────┐
│ Metrics Server                                  │
│ (Coleta métricas de CPU/memória dos pods)      │
└────────────────┬────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────┐
│ HPA Controller                                  │
│ 1. Consulta métricas a cada 15s (padrão)       │
│ 2. Calcula: desired = current * (metric/target)│
│ 3. Ajusta número de réplicas                   │
└────────────────┬────────────────────────────────┘
                 │
                 ↓
┌─────────────────────────────────────────────────┐
│ Deployment/ReplicaSet                           │
│ - Se CPU > 80%: scale up (adiciona pods)       │
│ - Se CPU < 50%: scale down (remove pods)       │
└─────────────────────────────────────────────────┘
```

### Pré-requisitos:

1. **Metrics Server** instalado
2. **Resources requests** definidos nos pods
3. **Deployment** ou ReplicaSet existente

### Instalar Metrics Server

```bash
# Verificar se já está instalado
kubectl get deployment metrics-server -n kube-system

# Se não estiver, instalar
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Verificar se está rodando
kubectl get pods -n kube-system | grep metrics-server

# Testar
kubectl top nodes
kubectl top pods
```

## 📈 Criar HPA

### Método 1: Imperativo (mais rápido)

```bash
# Criar HPA baseado em CPU
kubectl autoscale deployment nginx \
  --cpu-percent=50 \
  --min=2 \
  --max=10

# Ver HPA criado
kubectl get hpa nginx
```

**Resultado:**
- Mantém entre 2 e 10 pods
- Scale up quando CPU média > 50%
- Scale down quando CPU média < 50%

### Método 2: YAML Declarativo

#### HPA v2 (Recomendado - versão atual)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # Métrica 1: CPU
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50    # 50% da CPU request

  # Métrica 2: Memória
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80    # 80% da memory request

  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300    # Aguarda 5min antes de scale down
      policies:
      - type: Percent
        value: 50                         # Remove no máximo 50% dos pods
        periodSeconds: 60                 # A cada 60s
    scaleUp:
      stabilizationWindowSeconds: 0      # Scale up imediato
      policies:
      - type: Percent
        value: 100                        # Dobra o número de pods
        periodSeconds: 15                 # A cada 15s
      - type: Pods
        value: 4                          # Ou adiciona 4 pods
        periodSeconds: 15
      selectPolicy: Max                   # Usa a política mais agressiva
```

#### HPA v1 (Legado - apenas CPU)

```yaml
apiVersion: autoscaling/v1
kind: HorizontalPodAutoscaler
metadata:
  name: nginx-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  minReplicas: 2
  maxReplicas: 10
  targetCPUUtilizationPercentage: 50
```

## 🧪 Exemplo Completo: HPA com Deployment

```yaml
# 1. Deployment com resources definidos (OBRIGATÓRIO!)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
      - name: php-apache
        image: registry.k8s.io/hpa-example
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 200m        # ← IMPORTANTE: HPA usa isso como base
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi
---
# 2. Service para expor
apiVersion: v1
kind: Service
metadata:
  name: php-apache
spec:
  selector:
    app: php-apache
  ports:
  - port: 80
---
# 3. HPA
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: php-apache
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: php-apache
  minReplicas: 1
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 50
```

### Testar o HPA

```bash
# 1. Aplicar os recursos
kubectl apply -f php-apache.yaml

# 2. Ver status do HPA
kubectl get hpa php-apache --watch

# Output inicial:
# NAME         REFERENCE               TARGETS   MINPODS   MAXPODS   REPLICAS   AGE
# php-apache   Deployment/php-apache   0%/50%    1         10        1          1m

# 3. Gerar carga (em outro terminal)
kubectl run -it --rm load-generator --image=busybox -- /bin/sh
# Dentro do container:
while true; do wget -q -O- http://php-apache; done

# 4. Observar o scale up
kubectl get hpa php-apache --watch
# TARGETS vai aumentar: 0% → 50% → 100% → 200%
# REPLICAS vai aumentar: 1 → 2 → 4 → 8

# 5. Ver pods sendo criados
kubectl get pods -l app=php-apache --watch

# 6. Parar a carga (Ctrl+C no load-generator)
# Após ~5min, scale down começa
# REPLICAS vai diminuir: 8 → 4 → 2 → 1
```

## 📊 Métricas Customizadas

### Custom Metrics (Prometheus, etc.)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # Métrica customizada: requisições por segundo
  - type: Pods
    pods:
      metric:
        name: http_requests_per_second
      target:
        type: AverageValue
        averageValue: "1000"    # 1000 req/s por pod
```

### External Metrics (AWS CloudWatch, GCP, etc.)

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: app-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: app
  minReplicas: 2
  maxReplicas: 10
  metrics:
  # Métrica externa: tamanho da fila SQS
  - type: External
    external:
      metric:
        name: sqs_queue_length
        selector:
          matchLabels:
            queue: my-queue
      target:
        type: AverageValue
        averageValue: "30"    # 30 mensagens por pod
```

## 📐 Fórmula de Cálculo do HPA

```
desired replicas = ceil[current replicas * (current metric / target metric)]
```

**Exemplo:**
- Current replicas: 4
- Current CPU: 200m (média por pod)
- Target CPU: 100m
- Desired replicas = ceil[4 * (200/100)] = ceil[8] = 8

**Limites:**
- Se desired < minReplicas → usa minReplicas
- Se desired > maxReplicas → usa maxReplicas

## 🔧 Comandos Úteis - HPA

```bash
# Criar HPA
kubectl autoscale deployment <name> --cpu-percent=50 --min=2 --max=10

# Listar HPAs
kubectl get hpa
kubectl get horizontalpodautoscalers

# Ver detalhes
kubectl describe hpa <name>

# Ver em tempo real
kubectl get hpa <name> --watch

# Editar
kubectl edit hpa <name>

# Deletar
kubectl delete hpa <name>

# Ver eventos
kubectl get events --field-selector involvedObject.name=<hpa-name>

# Ver métricas atuais
kubectl top pods -l app=<label>
```

## 📏 Vertical Pod Autoscaler (VPA)

### O que é VPA?

**VPA** automaticamente ajusta os **recursos (CPU/memória)** dos containers baseado no uso histórico.

**Diferença do HPA:**
- HPA: aumenta **número de pods**
- VPA: aumenta **recursos de cada pod**

### Componentes do VPA:

1. **Recommender**: Analisa uso e recomenda values
2. **Updater**: Aplica recomendações (reinicia pods)
3. **Admission Controller**: Injeta requests/limits ao criar pod

### Instalar VPA

```bash
# Clone o repositório
git clone https://github.com/kubernetes/autoscaler.git
cd autoscaler/vertical-pod-autoscaler/

# Instalar VPA
./hack/vpa-up.sh

# Verificar instalação
kubectl get pods -n kube-system | grep vpa
```

### Exemplo de VPA

```yaml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: nginx-vpa
spec:
  targetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
  updatePolicy:
    updateMode: "Auto"    # Auto, Initial, Off, Recreate
  resourcePolicy:
    containerPolicies:
    - containerName: nginx
      minAllowed:
        cpu: 100m
        memory: 50Mi
      maxAllowed:
        cpu: 1
        memory: 500Mi
```

### Update Modes:

| Mode | Comportamento |
|------|---------------|
| **Off** | Apenas recomenda, não aplica |
| **Initial** | Aplica apenas na criação do pod |
| **Recreate** | Aplica deletando e recriando pod |
| **Auto** | Aplica durante a vida do pod (eviction) |

```bash
# Ver recomendações do VPA
kubectl describe vpa nginx-vpa

# Output mostra:
# Recommendation:
#   Container Recommendations:
#     Container Name:  nginx
#     Lower Bound:
#       Cpu:     100m
#       Memory:  262144k
#     Target:
#       Cpu:     200m
#       Memory:  524288k
#     Upper Bound:
#       Cpu:     500m
#       Memory:  1Gi
```

## 🔄 In-Place Pod Resizing (Redimensionamento In-Place)

### O que é In-Place Pod Resizing?

**In-Place Pod Resizing** permite atualizar os **resources (requests e limits)** de containers em pods **sem reiniciá-los**.

**Disponibilidade:**
- Feature Gate: `InPlacePodVerticalScaling` (Beta desde Kubernetes 1.27)
- Habilitado por padrão a partir do Kubernetes 1.27

### Como funciona:

```
┌──────────────────────────────────────────────────┐
│ Pod Rodando                                      │
│ CPU: 100m → 200m (sem restart!)                 │
│ Memory: 128Mi → 256Mi (sem restart!)            │
└──────────────────────────────────────────────────┘
         │
         │ kubectl set resources / patch
         ↓
┌──────────────────────────────────────────────────┐
│ Kubelet ajusta cgroups do container             │
│ - CPU: ajuste instantâneo                       │
│ - Memory: pode requerer restart (depende)       │
└──────────────────────────────────────────────────┘
```

### Diferença do VPA:

| Feature | VPA | In-Place Resizing |
|---------|-----|-------------------|
| **Reinicia pod?** | Sim (eviction) | Não (na maioria dos casos) |
| **Modo** | Automático | Manual ou controlado |
| **Uso** | Recomendações baseadas em histórico | Ajuste direto |
| **Downtime** | Sim | Não (CPU), Mínimo (Memory) |

### Resize Policies (Políticas de Redimensionamento)

Você pode controlar como o redimensionamento acontece:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
    resizePolicy:
    - resourceName: cpu
      restartPolicy: NotRequired    # Não reinicia para CPU
    - resourceName: memory
      restartPolicy: RestartNotRequired    # Tenta não reiniciar para memória
```

### Restart Policies:

| Policy | Comportamento |
|--------|---------------|
| **NotRequired** | Não reinicia o container (padrão para CPU) |
| **RestartNotRequired** | Tenta não reiniciar, mas pode reiniciar se necessário (padrão para Memory) |

**Importante:**
- **CPU**: Sempre pode ser ajustada sem restart (cgroups)
- **Memory**: Pode precisar de restart dependendo do runtime (containerd vs CRI-O)

### Exemplo Completo: Resize Manual

```yaml
# 1. Deployment inicial
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
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
        image: nginx
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 200m
            memory: 256Mi
        resizePolicy:
        - resourceName: cpu
          restartPolicy: NotRequired
        - resourceName: memory
          restartPolicy: RestartNotRequired
```

### Redimensionar Pods

#### Método 1: kubectl set resources

```bash
# Aumentar recursos do deployment
kubectl set resources deployment nginx \
  --requests=cpu=200m,memory=256Mi \
  --limits=cpu=400m,memory=512Mi

# Ver status do resize
kubectl get pods -l app=nginx -w

# Verificar recursos atuais
kubectl describe pod <pod-name> | grep -A 10 "Containers:"
```

#### Método 2: kubectl patch

```bash
# Patch direto no pod
kubectl patch pod nginx-xxxx -p '{
  "spec": {
    "containers": [{
      "name": "nginx",
      "resources": {
        "requests": {
          "cpu": "200m",
          "memory": "256Mi"
        },
        "limits": {
          "cpu": "400m",
          "memory": "512Mi"
        }
      }
    }]
  }
}'
```

#### Método 3: kubectl edit

```bash
# Editar deployment/pod diretamente
kubectl edit deployment nginx

# Modifique os valores de resources e salve
```

### Status do Resize

Durante o resize, o pod passa por fases:

```bash
# Ver status detalhado
kubectl get pod nginx-xxxx -o yaml | grep -A 20 "status:"

# Ver resize status
kubectl get pod nginx-xxxx -o jsonpath='{.status.resize}'
```

**Status possíveis:**
- **Proposed**: Resize foi solicitado
- **InProgress**: Resize está acontecendo
- **Deferred**: Resize foi adiado (recurso não disponível)
- **Infeasible**: Resize não é possível (excede limites do nó)

### Limitações e Considerações

#### ✅ Funciona bem:
- **CPU**: Sempre ajustável sem restart
- **Increase memory** (aumentar): Geralmente sem restart
- **Pods stateless**: Redimensionamento transparente

#### ⚠️ Cuidados:
- **Decrease memory** (diminuir): Pode causar OOMKill se app usar mais que o novo limite
- **Memory limits**: Runtime pode requerer restart em alguns casos
- **StatefulSets**: Teste cuidadosamente antes de usar em produção

#### ❌ Não funciona:
- Pods com `restartPolicy: Never`
- Recursos além de CPU/Memory (GPU, etc.)
- Pods em `Failed` ou `Succeeded` state

### Exemplo Prático: Scale Up Gradual

```bash
# 1. Criar deployment inicial
kubectl create deployment nginx --image=nginx --replicas=3

# 2. Definir recursos iniciais
kubectl set resources deployment nginx \
  --requests=cpu=100m,memory=128Mi \
  --limits=cpu=200m,memory=256Mi

# 3. Monitorar uso
kubectl top pods -l app=nginx

# 4. Se CPU alto, aumentar gradualmente
kubectl set resources deployment nginx \
  --requests=cpu=150m,memory=128Mi \
  --limits=cpu=300m,memory=256Mi

# 5. Verificar que pods não reiniciaram
kubectl get pods -l app=nginx
# AGE não deve ter mudado!

# 6. Verificar novos recursos
kubectl describe deployment nginx | grep -A 10 "Containers:"
```

### Habilitar Feature Gate (se necessário)

No Kubernetes < 1.27 ou se desabilitado:

```bash
# Editar kube-apiserver
vi /etc/kubernetes/manifests/kube-apiserver.yaml

# Adicionar feature gate:
spec:
  containers:
  - command:
    - kube-apiserver
    - --feature-gates=InPlacePodVerticalScaling=true
    ...

# Editar kubelet em cada nó
vi /var/lib/kubelet/config.yaml

# Adicionar:
featureGates:
  InPlacePodVerticalScaling: true

# Reiniciar kubelet
systemctl restart kubelet
```

### Boas Práticas

1. **Teste em ambientes não-produção primeiro**
   - Valide comportamento com sua aplicação

2. **Monitore durante resize**
   ```bash
   kubectl get pods -w
   kubectl top pods -w
   ```

3. **Use resizePolicy explicitamente**
   - Deixe claro se restart é aceitável

4. **Não diminua memory agressivamente**
   - Pode causar OOMKill se app estiver usando mais memória

5. **Combine com HPA para escalabilidade completa**
   - HPA: escala número de pods
   - In-Place Resize: ajusta recursos dos pods

### Troubleshooting

#### Resize não acontece

```bash
# 1. Verificar feature gate
kubectl get --raw /api/v1 | jq '.serverAddressByClientCIDRs'

# 2. Ver eventos do pod
kubectl describe pod <pod-name> | grep -A 10 Events

# 3. Ver status do resize
kubectl get pod <pod-name> -o jsonpath='{.status.resize}'

# 4. Verificar logs do kubelet (no nó)
journalctl -u kubelet | grep -i resize
```

#### Pod reiniciou inesperadamente

```bash
# Ver motivo do restart
kubectl describe pod <pod-name> | grep -A 5 "Last State"

# Verificar se foi por OOMKill
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[0].lastState.terminated.reason}'

# Se foi OOM: aumentou memory limits, mas app usa mais que o novo valor
```

## 🚫 HPA vs VPA - Cuidado!

**IMPORTANTE**: Não use HPA e VPA juntos para CPU/memória!

```
❌ EVITE:
┌─────────────────────┐
│ Deployment          │
│ ├─ HPA (CPU)       │ ← Conflito!
│ └─ VPA (CPU)       │ ← Conflito!
└─────────────────────┘

✅ USE:
┌─────────────────────┐
│ Deployment          │
│ ├─ HPA (CPU)       │
│ └─ VPA (memory)    │ ← OK!
└─────────────────────┘

OU:

┌─────────────────────┐
│ Deployment          │
│ ├─ HPA (custom)    │ ← Métrica customizada
│ └─ VPA (CPU/mem)   │ ← OK!
└─────────────────────┘
```

## ☁️ Cluster Autoscaler

### O que é Cluster Autoscaler?

**Cluster Autoscaler** automaticamente ajusta o **número de nós** no cluster baseado em pods pending.

**Como funciona:**
1. Detecta pods em `Pending` (não conseguem ser agendados)
2. Adiciona nós ao cluster (se configurado)
3. Remove nós ociosos após um período

### Quando adiciona nós:
- Pods estão em Pending por falta de recursos
- Não há nós disponíveis para agendá-los

### Quando remove nós:
- Nó está subutilizado por ~10min
- Todos os pods podem ser movidos para outros nós
- Nó não tem pods com PodDisruptionBudget que bloqueia eviction

### Configuração (depende do cloud provider)

**AWS:**
```bash
# Configurar AutoScaling Group com tags
aws autoscaling create-or-update-tags \
  --tags ResourceId=my-asg,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/enabled,Value=true \
  --tags ResourceId=my-asg,ResourceType=auto-scaling-group,Key=k8s.io/cluster-autoscaler/my-cluster,Value=owned
```

**GCP:**
```bash
# Criar node pool com autoscaling
gcloud container node-pools create my-pool \
  --cluster=my-cluster \
  --enable-autoscaling \
  --min-nodes=1 \
  --max-nodes=10
```

## 📋 Boas Práticas

### Para HPA:

1. **Sempre defina resources requests**
   ```yaml
   resources:
     requests:
       cpu: 100m
       memory: 128Mi
   ```

2. **Use stabilizationWindow**
   - Evita flapping (scale up/down rápido)
   - Padrão: 5min para scale down

3. **Teste limites (min/max)**
   - minReplicas: mínimo para disponibilidade
   - maxReplicas: limite de custo/capacidade

4. **Monitore métricas**
   ```bash
   kubectl get hpa --watch
   kubectl top pods
   ```

### Para VPA:

1. **Comece com updateMode: "Off"**
   - Observe recomendações
   - Aplique manualmente depois

2. **Defina limites (min/max)**
   - Evita recomendações absurdas

3. **Cuidado com stateful workloads**
   - VPA reinicia pods
   - Pode causar downtime

## 🧪 Troubleshooting

### HPA não escala

```bash
# 1. Verificar se Metrics Server está rodando
kubectl get pods -n kube-system | grep metrics-server

# 2. Verificar se métricas estão disponíveis
kubectl top pods

# 3. Ver status do HPA
kubectl describe hpa <name>

# Procure por:
# - Conditions: ScalingActive=False
# - Events: "failed to get cpu utilization"

# 4. Verificar se pods têm resources requests
kubectl get deployment <name> -o yaml | grep -A 5 resources

# 5. Ver eventos
kubectl get events --sort-by='.lastTimestamp' | grep hpa
```

**Problemas comuns:**
- Metrics Server não instalado
- Pods sem `resources.requests`
- Deployment não existe
- Métricas não disponíveis (aguardar ~1min)

### HPA escala constantemente (flapping)

```bash
# Adicionar stabilizationWindow
kubectl edit hpa <name>
```

```yaml
behavior:
  scaleDown:
    stabilizationWindowSeconds: 300    # 5 minutos
  scaleUp:
    stabilizationWindowSeconds: 60     # 1 minuto
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Criar HPA**
   ```bash
   kubectl autoscale deployment nginx --cpu-percent=50 --min=2 --max=10
   ```

2. **Ver status do HPA**
   ```bash
   kubectl get hpa
   kubectl describe hpa <name>
   ```

3. **Metrics Server é necessário**
   ```bash
   kubectl top nodes
   kubectl top pods
   ```

4. **Resources requests são obrigatórios**
   ```yaml
   resources:
     requests:
       cpu: 100m
   ```

5. **Diferença HPA vs VPA vs In-Place Resizing**
   - HPA: horizontal (mais pods) - sem restart
   - VPA: vertical (mais recursos por pod) - com restart
   - In-Place Resizing: vertical (mais recursos) - sem restart (manual)

6. **In-Place Resizing**
   ```bash
   # Redimensionar deployment sem reiniciar pods
   kubectl set resources deployment nginx \
     --requests=cpu=200m,memory=256Mi \
     --limits=cpu=400m,memory=512Mi

   # Verificar que pods não reiniciaram (AGE não muda)
   kubectl get pods -l app=nginx
   ```

### 🧪 Cenário típico na prova:

> **"Crie um HPA para o deployment 'webapp' que mantém entre 3 e 10 pods, escalando quando CPU média ultrapassar 70%."**

```bash
# Criar HPA
kubectl autoscale deployment webapp \
  --cpu-percent=70 \
  --min=3 \
  --max=10

# Verificar
kubectl get hpa webapp
kubectl describe hpa webapp
```

> **"O HPA não está funcionando. Investigue e corrija o problema."**

```bash
# 1. Ver status
kubectl get hpa
kubectl describe hpa <name>

# 2. Verificar Metrics Server
kubectl get pods -n kube-system | grep metrics-server
kubectl top nodes

# 3. Verificar resources requests no deployment
kubectl get deployment <name> -o yaml | grep -A 5 resources

# 4. Se não tiver requests, adicionar
kubectl set resources deployment <name> \
  --requests=cpu=100m,memory=128Mi
```

## 💡 Dicas para a Prova

1. **HPA precisa de Metrics Server**
   - Verifique se está instalado: `kubectl top nodes`

2. **Resources requests são obrigatórios**
   - HPA não funciona sem requests definidos

3. **Aguarde ~1min para métricas**
   - Após criar deployment, aguarde métricas aparecerem

4. **Use --watch para monitorar**
   ```bash
   kubectl get hpa --watch
   ```

5. **Comando imperativo é mais rápido**
   ```bash
   kubectl autoscale deployment <name> --cpu-percent=50 --min=2 --max=10
   ```

---

⬅️ **Anterior**: [resource-limits.md](./resource-limits.md) | ➡️ **Próximo**: [configmaps-secrets.md](./configmaps-secrets.md)
