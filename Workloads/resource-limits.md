# Resource Limits e Requests

## 📋 O que são Resources no Kubernetes?

Resources são a forma de especificar quanto de CPU e memória (RAM) um container precisa para executar. Eles permitem que o Kubernetes tome decisões inteligentes sobre:

- **Onde** agendar os pods (scheduling)
- **Quanto** de recursos cada pod pode consumar
- **Quando** evitar sobrecarga nos nós

## 🎯 Requests vs Limits

### **Requests** (Solicitações)
- Quantidade **mínima garantida** de recursos que o container precisa
- Usado pelo **Scheduler** para decidir em qual nó colocar o pod
- O container **sempre** terá pelo menos essa quantidade disponível

### **Limits** (Limites)
- Quantidade **máxima** de recursos que o container pode usar
- Impede que um container consuma todos os recursos do nó
- Se o container tentar ultrapassar:
  - **CPU**: será throttled (limitado)
  - **Memory**: pode ser killed (OOMKilled - Out Of Memory)

## 📊 Diagrama Conceitual

```
|-----------|------------|------------|
0         Request      Limit      ∞

- Container garante ter "Request"
- Container pode usar até "Limit"
- Se passar "Limit" de memória → OOMKilled
- Se passar "Limit" de CPU → Throttled
```

## 💾 Unidades de Medida

### CPU
- Medido em **cores** (núcleos de CPU)
- `1` = 1 CPU core
- `0.5` ou `500m` = meio core (500 millicores)
- `100m` = 0.1 core (10% de um core)

```yaml
cpu: "1"        # 1 core inteiro
cpu: "500m"     # 500 millicores = 0.5 core
cpu: "100m"     # 100 millicores = 0.1 core
```

### Memory
- Medido em **bytes**
- Pode usar: `Mi` (Mebibyte), `Gi` (Gibibyte), `M` (Megabyte), `G` (Gigabyte)

```yaml
memory: "128Mi"   # 128 Mebibytes (≈ 134 MB)
memory: "1Gi"     # 1 Gibibyte (≈ 1.07 GB)
memory: "256M"    # 256 Megabytes
```

> **Dica**: Prefira `Mi` e `Gi` (base 1024) ao invés de `M` e `G` (base 1000)

## 📝 Exemplo Completo de Pod com Resources

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-demo
spec:
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:           # Mínimo garantido
        memory: "64Mi"
        cpu: "250m"
      limits:             # Máximo permitido
        memory: "128Mi"
        cpu: "500m"
```

**Significado:**
- O pod é **garantido** ter 64Mi de RAM e 0.25 CPU (250m)
- O pod pode usar **até** 128Mi de RAM e 0.5 CPU (500m)
- O Scheduler só coloca esse pod em nós com **pelo menos** 250m CPU e 64Mi RAM disponíveis

## 🚀 Comandos Úteis

### Criar pod com resources (imperativo)
```bash
# Gerar YAML com dry-run
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Editar o YAML para adicionar resources
vim pod.yaml

# Aplicar
kubectl apply -f pod.yaml
```

### Ver recursos usados por pods
```bash
# Ver uso de recursos de todos os pods
kubectl top pods

# Ver uso de recursos de um pod específico
kubectl top pod <pod-name>

# Ver recursos dos nós
kubectl top nodes
```

### Ver requests e limits configurados
```bash
# Descrever pod
kubectl describe pod <pod-name>

# Ver recursos definidos
kubectl get pod <pod-name> -o jsonpath='{.spec.containers[*].resources}'
```

## 🎯 QoS Classes (Quality of Service)

O Kubernetes classifica pods automaticamente em 3 categorias com base nos resources:

### 1. **Guaranteed** (Garantido) - Maior prioridade
- Todos os containers têm **requests = limits**
- Menos provável de ser evicted (removido) em caso de pressão de recursos

```yaml
resources:
  requests:
    memory: "200Mi"
    cpu: "500m"
  limits:
    memory: "200Mi"    # Igual ao request
    cpu: "500m"        # Igual ao request
```

### 2. **Burstable** (Explosivo) - Prioridade média
- Pelo menos 1 container tem **requests < limits** (ou só requests)
- Pode usar mais recursos quando disponível

```yaml
resources:
  requests:
    memory: "100Mi"
    cpu: "250m"
  limits:
    memory: "200Mi"    # Maior que request
    cpu: "500m"        # Maior que request
```

### 3. **BestEffort** (Melhor Esforço) - Menor prioridade
- **Nenhum** request ou limit definido
- Primeiro a ser evicted em caso de pressão de recursos

```yaml
# Nenhum resources definido
spec:
  containers:
  - name: app
    image: nginx
```

### Ver a QoS Class de um pod
```bash
kubectl get pod <pod-name> -o jsonpath='{.status.qosClass}'
```

## ⚠️ O que acontece quando ultrapassar os limites?

### CPU Limit Excedido
- Container é **throttled** (limitado)
- Ele continua rodando, mas mais lento
- **Não é killed**

### Memory Limit Excedido
- Container é **killed** (OOMKilled - Out Of Memory)
- Pod entra em estado `CrashLoopBackOff`
- Kubernetes tenta reiniciar automaticamente

```bash
# Ver se pod foi OOMKilled
kubectl describe pod <pod-name>
# Procure por: "Last State: Terminated, Reason: OOMKilled"
```

## 📊 LimitRange - Definir padrões no Namespace

LimitRange permite definir limites padrão para todos os pods/containers em um namespace.

```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - max:                    # Máximo permitido
      memory: "1Gi"
      cpu: "1"
    min:                    # Mínimo requerido
      memory: "64Mi"
      cpu: "100m"
    default:                # Limit padrão (se não especificar)
      memory: "256Mi"
      cpu: "500m"
    defaultRequest:         # Request padrão (se não especificar)
      memory: "128Mi"
      cpu: "250m"
    type: Container
```

**Aplicar:**
```bash
kubectl apply -f limitrange.yaml
kubectl describe limitrange -n dev
```

**Resultado:**
- Qualquer pod criado no namespace `dev` sem resources definidos receberá os valores padrão
- Pods não podem exceder os valores `max` ou ser menor que `min`

## 📦 ResourceQuota - Limites por Namespace

ResourceQuota limita o **total** de recursos que um namespace inteiro pode usar.

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "4"          # Total de CPU requests no namespace
    requests.memory: "8Gi"     # Total de Memory requests
    limits.cpu: "8"            # Total de CPU limits
    limits.memory: "16Gi"      # Total de Memory limits
    pods: "10"                 # Máximo de pods
```

**Aplicar:**
```bash
kubectl apply -f resourcequota.yaml
kubectl describe resourcequota -n dev
```

**Ver uso atual:**
```bash
kubectl get resourcequota -n dev
```

**Resultado:**
- Impede criar mais pods se o namespace já estiver usando todos os recursos alocados
- Útil para ambientes multi-tenant (vários times compartilhando cluster)

## 🛠️ Boas Práticas

### ✅ Sempre defina requests
- Ajuda o Scheduler a tomar decisões corretas
- Garante recursos mínimos para seu app

### ✅ Defina limits para evitar "noisy neighbors"
- Impede que um pod consuma todos os recursos do nó
- Protege outros pods

### ✅ Use QoS Guaranteed para workloads críticos
- Defina `requests = limits`
- Menor chance de ser evicted

### ⚠️ Monitore recursos com `kubectl top`
- Ajuste requests/limits com base no uso real
- Evite over-provisioning (desperdiçar recursos)

### ⚠️ Cuidado com OOMKilled
- Se acontecer frequentemente, aumente o memory limit
- Ou otimize a aplicação para usar menos memória

## 🧪 Exemplo Prático: Deployment com Resources

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deploy
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
        image: nginx:1.27
        resources:
          requests:
            memory: "128Mi"
            cpu: "250m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        ports:
        - containerPort: 80
```

**Criar:**
```bash
kubectl apply -f nginx-deploy.yaml

# Ver recursos
kubectl top pods -l app=nginx

# Ver QoS
kubectl get pods -l app=nginx -o custom-columns=NAME:.metadata.name,QOS:.status.qosClass
```

## 🔍 Troubleshooting

### Pod em Pending - Recursos Insuficientes
```bash
kubectl describe pod <pod-name>
```
Procure por:
```
Events:
  Warning  FailedScheduling  ... Insufficient cpu
  Warning  FailedScheduling  ... Insufficient memory
```

**Soluções:**
- Reduzir os `requests` do pod
- Adicionar mais nós ao cluster
- Escalar down outros pods

### OOMKilled (Out Of Memory)
```bash
kubectl describe pod <pod-name>
```
Procure por:
```
Last State:     Terminated
  Reason:       OOMKilled
  Exit Code:    137
```

**Soluções:**
- Aumentar o `memory limit`
- Otimizar a aplicação
- Verificar memory leaks

### CPU Throttling
```bash
# Ver métricas detalhadas
kubectl top pod <pod-name>

# Se uso de CPU está sempre no limite, considere aumentar
```

## 📚 Recursos para Estudo

### Documentação Oficial
- [Managing Resources for Containers](https://kubernetes.io/docs/concepts/configuration/manage-resources-containers/)
- [Configure Quality of Service for Pods](https://kubernetes.io/docs/tasks/configure-pod-container/quality-service-pod/)
- [Limit Ranges](https://kubernetes.io/docs/concepts/policy/limit-range/)
- [Resource Quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/)

### Comandos Rápidos de Revisão
```bash
# Ver recursos requests/limits
kubectl describe nodes | grep -A 5 "Allocated resources"

# Ver todos os LimitRanges
kubectl get limitrange --all-namespaces

# Ver todos os ResourceQuotas
kubectl get resourcequota --all-namespaces

# Ver uso real vs configurado
kubectl top pods
kubectl describe pod <pod-name> | grep -A 10 "Limits"
```

## 🎯 Pontos Importantes para a Prova CKA

1. **Saber definir** requests e limits em pods
2. **Entender** a diferença entre requests e limits
3. **Conhecer** as 3 QoS classes (Guaranteed, Burstable, BestEffort)
4. **Saber criar** LimitRange e ResourceQuota
5. **Troubleshoot** pods em Pending por falta de recursos
6. **Troubleshoot** OOMKilled
7. **Usar** `kubectl top` para monitorar recursos

---

⬅️ **Anterior**: [pods.md](./pods.md) | ➡️ **Próximo**: [replicaset-deployments.md](./replicaset-deployments.md)
