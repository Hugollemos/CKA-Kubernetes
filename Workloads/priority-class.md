# PriorityClass

## 📋 O que é PriorityClass?

**PriorityClass** é um objeto do Kubernetes que define a **prioridade relativa** de pods. Quando o cluster está com recursos limitados, o Kubernetes usa essas prioridades para:

- **Scheduling**: Decidir qual pod será agendado primeiro
- **Preemption**: Evict (remover) pods de menor prioridade para dar lugar a pods de maior prioridade
- **Eviction**: Determinar qual pod será removido primeiro em caso de pressão de recursos no nó

### Características principais:
- ✅ Objeto **cluster-scoped** (não está em um namespace)
- ✅ Define um **valor numérico de prioridade** (quanto maior, mais importante)
- ✅ Pode marcar pods como **não preemptíveis** (não podem ser removidos para dar lugar a outros)
- ✅ Usado pelo **Scheduler** para tomar decisões de agendamento
- ✅ Influencia ordem de **eviction** quando nó está com recursos baixos

## 🎯 Por que usar PriorityClass?

### Cenários comuns:

1. **Priorizar workloads críticos**
   - Banco de dados mais importante que cache
   - API de pagamento mais importante que serviço de notificações

2. **Garantir disponibilidade de serviços essenciais**
   - Evitar que pods críticos sejam evicted
   - Assegurar que pods importantes sejam agendados primeiro

3. **Otimizar uso de recursos**
   - Em clusters compartilhados (multi-tenant)
   - Quando há competição por recursos limitados

4. **Ambientes de desenvolvimento vs produção**
   - Produção tem prioridade sobre desenvolvimento
   - Jobs batch têm menor prioridade que serviços online

## 📊 Como Funciona?

```
┌────────────────────────────────────────────────────┐
│  Scheduler                                         │
│                                                    │
│  1. Pod com priority=1000 chega                   │
│  2. Pod com priority=100 chega                    │
│                                                    │
│  ┌──────────────────────────────────────────┐    │
│  │ Scheduling Queue                         │    │
│  │ ┌──────────────────────────────────┐    │    │
│  │ │ priority=1000 (high)  ← Primeiro │    │    │
│  │ └──────────────────────────────────┘    │    │
│  │ ┌──────────────────────────────────┐    │    │
│  │ │ priority=100 (low)    ← Segundo  │    │    │
│  │ └──────────────────────────────────┘    │    │
│  └──────────────────────────────────────────┘    │
│                                                    │
│  3. Scheduler agenda pods por ordem de prioridade │
└────────────────────────────────────────────────────┘

┌────────────────────────────────────────────────────┐
│  Preemption (quando não há recursos)              │
│                                                    │
│  Cluster está cheio (sem recursos disponíveis)    │
│                                                    │
│  1. Pod HIGH (priority=1000) chega                │
│  2. Não há recursos disponíveis                   │
│  3. Scheduler verifica pods rodando               │
│  4. Encontra pod LOW (priority=100)               │
│  5. Evict pod LOW                                 │
│  6. Agenda pod HIGH no lugar                      │
│                                                    │
│  Pod HIGH substitui Pod LOW                       │
└────────────────────────────────────────────────────┘
```

## 📝 Estrutura de um PriorityClass

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000              # Valor numérico da prioridade
globalDefault: false        # Se true, aplica a todos os pods sem priorityClassName
preemptionPolicy: PreemptLowerPriority  # Política de preempção
description: "Prioridade alta para serviços críticos"
```

### Campos importantes:

**value** (obrigatório)
- Valor inteiro da prioridade
- Quanto **maior**, mais prioritário
- Valores negativos são permitidos
- Valores acima de 1 bilhão são reservados para system pods

**globalDefault** (opcional)
- `true`: Aplica esta prioridade a todos os pods que não especificarem `priorityClassName`
- `false` (padrão): Pods sem `priorityClassName` recebem prioridade 0

**preemptionPolicy** (opcional)
- `PreemptLowerPriority` (padrão): Pode evict pods de menor prioridade
- `Never`: Nunca evict outros pods (espera recursos ficarem disponíveis)

**description** (opcional)
- Descrição legível do propósito desta priority class

## 🚀 Criando e Usando PriorityClass

### 1. Criar PriorityClasses

```yaml
# high-priority.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
preemptionPolicy: PreemptLowerPriority
description: "Usado para pods críticos que devem ser agendados primeiro"
---
# medium-priority.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: medium-priority
value: 500000
globalDefault: false
description: "Usado para pods de aplicações normais"
---
# low-priority.yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low-priority
value: 100000
globalDefault: false
description: "Usado para jobs batch e workloads não críticos"
```

```bash
# Aplicar as PriorityClasses
kubectl apply -f high-priority.yaml
kubectl apply -f medium-priority.yaml
kubectl apply -f low-priority.yaml

# Listar PriorityClasses
kubectl get priorityclasses
```

### 2. Usar PriorityClass em um Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority    # ← Referencia a PriorityClass
  containers:
  - name: app
    image: nginx:1.27
    resources:
      requests:
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 1000m
        memory: 512Mi
```

### 3. Usar PriorityClass em um Deployment

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
    spec:
      priorityClassName: medium-priority    # ← Aplica a todos os pods
      containers:
      - name: web
        image: nginx:1.27
        resources:
          requests:
            cpu: 200m
            memory: 128Mi
```

## 🔍 PriorityClasses Pré-definidas do Sistema

O Kubernetes já vem com algumas PriorityClasses pré-configuradas:

```bash
# Listar PriorityClasses do sistema
kubectl get priorityclasses
```

**Output típico:**
```
NAME                      VALUE        GLOBAL-DEFAULT   AGE
system-cluster-critical   2000000000   false            10d
system-node-critical      2000001000   false            10d
```

### system-cluster-critical
- **Valor**: 2.000.000.000
- **Uso**: Componentes críticos do cluster
- **Exemplos**: kube-dns, coredns, metrics-server

### system-node-critical
- **Valor**: 2.000.001.000 (maior que cluster-critical)
- **Uso**: Componentes críticos do nó
- **Exemplos**: kubelet, kube-proxy

⚠️ **Importante**: Não use essas PriorityClasses para suas aplicações! Elas são reservadas para componentes do sistema.

## 📊 Valores de Prioridade - Boas Práticas

```
2.000.001.000  ┐
               │  system-node-critical (RESERVADO)
2.000.000.000  ┘

1.000.000.000  ─  Limite recomendado para aplicações

  10.000.000   ─  critical: aplicações críticas de produção
   5.000.000   ─  high: aplicações importantes de produção
   1.000.000   ─  medium: aplicações normais de produção
     500.000   ─  low: aplicações de desenvolvimento
     100.000   ─  batch: jobs batch, processamento não urgente
           0   ─  default: pods sem priorityClassName
      -1.000   ─  very-low: tarefas de manutenção, limpeza
```

**Recomendações:**
- Mantenha valores **abaixo de 1 bilhão** para suas aplicações
- Use uma **escala consistente** (ex: múltiplos de 100.000)
- Deixe **espaço entre valores** para adicionar prioridades intermediárias depois
- Documente o significado de cada nível de prioridade

## 🎯 Preemption (Preempção)

### Como funciona a Preemption?

Quando um pod de alta prioridade não pode ser agendado devido à falta de recursos:

1. **Scheduler** identifica que não há recursos disponíveis
2. Procura por pods de **menor prioridade** rodando nos nós
3. Seleciona pods para **eviction** (remoção)
4. Remove os pods selecionados
5. Agenda o pod de alta prioridade

### Exemplo de Preemption

```yaml
# 1. Criar PriorityClass baixa
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 100
---
# 2. Criar PriorityClass alta
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 1000
---
# 3. Pod com prioridade BAIXA (criado primeiro)
apiVersion: v1
kind: Pod
metadata:
  name: low-priority-pod
spec:
  priorityClassName: low
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 1
        memory: 1Gi
---
# 4. Pod com prioridade ALTA (criado depois)
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: high
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 1
        memory: 1Gi
```

**Comportamento:**
- Se o cluster não tiver recursos para o `high-priority-pod`
- O Scheduler **evict** o `low-priority-pod`
- E agenda o `high-priority-pod` no lugar

### Desabilitar Preemption

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-no-preempt
value: 1000
preemptionPolicy: Never    # ← Nunca evict outros pods
description: "Alta prioridade mas não remove outros pods"
```

**Uso:**
- Pod será agendado primeiro quando houver recursos
- Mas **não** removerá outros pods se não houver recursos
- Ficará em `Pending` até recursos ficarem disponíveis

## 🧪 Exemplo Prático Completo

### Cenário: Cluster com aplicações de diferentes criticidades

```bash
# 1. Criar PriorityClasses
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-critical
value: 10000000
description: "Aplicações críticas de produção"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: production-normal
value: 5000000
description: "Aplicações normais de produção"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: development
value: 1000000
description: "Aplicações de desenvolvimento"
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: batch-jobs
value: 100000
description: "Jobs batch e processamento não urgente"
EOF

# 2. Verificar PriorityClasses criadas
kubectl get priorityclasses

# 3. Criar Deployments com diferentes prioridades

# 3a. Banco de dados (CRÍTICO)
kubectl create deployment postgres --image=postgres:15 --replicas=1 \
  --dry-run=client -o yaml | \
  sed '/spec:/a\      priorityClassName: production-critical' | \
  kubectl apply -f -

# 3b. API Backend (NORMAL)
kubectl create deployment api --image=nginx:1.27 --replicas=3 \
  --dry-run=client -o yaml | \
  sed '/spec:/a\      priorityClassName: production-normal' | \
  kubectl apply -f -

# 3c. App de desenvolvimento
kubectl create deployment dev-app --image=nginx:alpine --replicas=2 \
  --dry-run=client -o yaml | \
  sed '/spec:/a\      priorityClassName: development' | \
  kubectl apply -f -

# 4. Verificar prioridades dos pods
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
PRIORITY:.spec.priority,\
PRIORITY_CLASS:.spec.priorityClassName,\
STATUS:.status.phase

# Output:
# NAME                        PRIORITY    PRIORITY_CLASS          STATUS
# postgres-xxx                10000000    production-critical     Running
# api-xxx                     5000000     production-normal       Running
# api-yyy                     5000000     production-normal       Running
# api-zzz                     5000000     production-normal       Running
# dev-app-xxx                 1000000     development             Running
# dev-app-yyy                 1000000     development             Running
```

## 🔍 Comandos Úteis

### Ver PriorityClasses
```bash
# Listar todas as PriorityClasses
kubectl get priorityclasses
kubectl get pc    # alias

# Ver detalhes de uma PriorityClass
kubectl describe priorityclass high-priority

# Ver YAML completo
kubectl get priorityclass high-priority -o yaml
```

### Ver prioridades dos pods
```bash
# Ver prioridade de todos os pods
kubectl get pods -A -o custom-columns=\
NAME:.metadata.name,\
NAMESPACE:.metadata.namespace,\
PRIORITY:.spec.priority,\
PRIORITY_CLASS:.spec.priorityClassName

# Ver apenas pods com uma PriorityClass específica
kubectl get pods -A --field-selector spec.priorityClassName=high-priority

# Ordenar pods por prioridade (do maior para o menor)
kubectl get pods -A -o json | \
  jq -r '.items | sort_by(.spec.priority) | reverse |
  .[] | [.metadata.name, .spec.priority, .spec.priorityClassName] | @tsv'
```

### Criar PriorityClass via comando
```bash
# Criar PriorityClass imperativa (apenas o YAML)
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: my-priority
value: 1000
description: "Minha priority class"
EOF
```

### Editar/Deletar PriorityClass
```bash
# Editar PriorityClass
kubectl edit priorityclass high-priority

# Deletar PriorityClass
kubectl delete priorityclass high-priority

# ⚠️ Cuidado: Deletar uma PriorityClass NÃO afeta pods já criados
# Pods criados continuam com a prioridade original
```

## ⚠️ Limitações e Cuidados

### 1. PriorityClass não pode ser mudada em pods existentes
```bash
# ERRO: Não é possível mudar priorityClassName de um pod rodando
kubectl edit pod my-pod    # Tentativa de mudar priorityClassName
# Error: spec.priorityClassName: Forbidden: may not be changed

# SOLUÇÃO: Deletar e recriar o pod
kubectl delete pod my-pod
kubectl apply -f my-pod.yaml   # Com novo priorityClassName
```

### 2. Deletar PriorityClass não afeta pods existentes
```bash
# Deletar PriorityClass
kubectl delete priorityclass my-priority

# Pods criados com essa PriorityClass continuam com o valor original
kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority
# Os pods mantêm a prioridade numérica
```

### 3. Preemption pode causar disrupção
```yaml
# Se você não quer que pods sejam evicted, use:
preemptionPolicy: Never

# Ou garanta recursos suficientes no cluster
# Ou use PodDisruptionBudgets
```

### 4. Valores acima de 1 bilhão são reservados
```yaml
# ❌ EVITE: Valores muito altos
value: 2000000000    # Reservado para system pods

# ✅ USE: Valores abaixo de 1 bilhão
value: 10000000
```

## 🎯 PriorityClass vs QoS Class

**São conceitos diferentes!**

| Aspecto | PriorityClass | QoS Class |
|---------|---------------|-----------|
| **O que é** | Prioridade de scheduling | Qualidade de serviço |
| **Definido por** | `priorityClassName` | `requests` e `limits` |
| **Usado por** | Scheduler (scheduling, preemption) | Kubelet (eviction) |
| **Valores** | Número inteiro arbitrário | Guaranteed/Burstable/BestEffort |
| **Scope** | Cluster-wide | Por pod |

### Como trabalham juntos:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: important-app
spec:
  priorityClassName: high-priority    # ← Scheduling priority
  containers:
  - name: app
    image: nginx
    resources:
      requests:                        # ← QoS Class (Guaranteed)
        cpu: 500m
        memory: 256Mi
      limits:
        cpu: 500m
        memory: 256Mi
```

**Resultado:**
- **PriorityClass**: Pod é agendado primeiro, pode fazer preemption
- **QoS Guaranteed**: Pod tem menor chance de ser evicted quando nó está sob pressão

## 🧪 Teste de Preemption

```bash
# 1. Criar um cluster pequeno (exemplo com kind ou minikube)
# Ou usar um namespace com ResourceQuota limitado

# 2. Criar PriorityClasses
kubectl apply -f - <<EOF
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: low
value: 100
---
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high
value: 1000
EOF

# 3. Criar pod com prioridade BAIXA que consome muitos recursos
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: low-priority-pod
spec:
  priorityClassName: low
  containers:
  - name: stress
    image: polinux/stress
    command: ["stress"]
    args: ["--vm", "1", "--vm-bytes", "500M", "--vm-hang", "1"]
    resources:
      requests:
        cpu: 500m
        memory: 500Mi
EOF

# 4. Aguardar o pod estar rodando
kubectl get pods -w

# 5. Criar pod com prioridade ALTA
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: high-priority-pod
spec:
  priorityClassName: high
  containers:
  - name: nginx
    image: nginx
    resources:
      requests:
        cpu: 500m
        memory: 500Mi
EOF

# 6. Observar: Se não houver recursos, low-priority-pod será evicted
kubectl get pods -w

# 7. Ver eventos de preemption
kubectl get events --sort-by='.lastTimestamp' | grep -i preempt
```

## 📚 Recursos para Estudo

### Documentação Oficial
- [Pod Priority and Preemption](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/)
- [PriorityClass](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-priority-preemption/#priorityclass)
- [Pod Scheduling Readiness](https://kubernetes.io/docs/concepts/scheduling-eviction/pod-scheduling-readiness/)

### Comandos Rápidos de Revisão
```bash
# Criar PriorityClass
kubectl apply -f priorityclass.yaml

# Listar PriorityClasses
kubectl get priorityclasses

# Ver prioridades dos pods
kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority

# Descrever PriorityClass
kubectl describe priorityclass <name>

# Deletar PriorityClass
kubectl delete priorityclass <name>

# Ver eventos de preemption
kubectl get events --field-selector reason=Preempted
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Criar PriorityClass**
   ```yaml
   apiVersion: scheduling.k8s.io/v1
   kind: PriorityClass
   metadata:
     name: high-priority
   value: 1000000
   description: "Alta prioridade"
   ```

2. **Usar PriorityClass em um Pod**
   ```yaml
   spec:
     priorityClassName: high-priority
   ```

3. **Listar e visualizar PriorityClasses**
   ```bash
   kubectl get priorityclasses
   kubectl describe priorityclass <name>
   ```

4. **Entender Preemption**
   - Pods de alta prioridade podem evict pods de baixa prioridade
   - Use `preemptionPolicy: Never` para desabilitar

5. **Diferença entre PriorityClass e QoS**
   - PriorityClass: scheduling (quando agendar)
   - QoS: eviction (quando remover)

6. **Ver prioridade de pods**
   ```bash
   kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority
   ```

### 🧪 Cenário típico na prova:

> **"Crie uma PriorityClass chamada 'database-priority' com valor 1000000. Depois, crie um pod chamado 'postgres' usando a imagem 'postgres:15' que use essa PriorityClass."**

**Solução:**
```bash
# 1. Criar PriorityClass
cat <<EOF | kubectl apply -f -
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: database-priority
value: 1000000
description: "Prioridade para bancos de dados"
EOF

# 2. Criar Pod
kubectl run postgres --image=postgres:15 --dry-run=client -o yaml > postgres.yaml

# 3. Adicionar priorityClassName
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: postgres
spec:
  priorityClassName: database-priority
  containers:
  - name: postgres
    image: postgres:15
    env:
    - name: POSTGRES_PASSWORD
      value: secret
EOF

# 4. Verificar
kubectl get pod postgres -o yaml | grep -A 2 priority
```

## 💡 Dicas para a Prova

1. **Sempre teste a sintaxe YAML**
   ```bash
   kubectl apply --dry-run=client -f priorityclass.yaml
   ```

2. **Use kubectl get priorityclasses para ver as existentes**
   - Evite conflitos de nomes
   - Veja valores já usados

3. **Lembre que PriorityClass é cluster-scoped**
   - Não precisa especificar namespace
   - `kubectl get pc` funciona sem `-n`

4. **Para adicionar priorityClassName a um Deployment existente**
   ```bash
   kubectl edit deployment my-app
   # Adicione priorityClassName no spec.template.spec
   ```

5. **Ver se preemption aconteceu**
   ```bash
   kubectl get events | grep -i preempt
   ```

---

⬅️ **Anterior**: [static-pods.md](./static-pods.md) | ➡️ **Próximo**: [scheduling.md](./scheduling.md)
