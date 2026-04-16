# DaemonSets

## 📋 O que é um DaemonSet?

Um **DaemonSet** garante que uma cópia de um pod execute em **todos os nós** (ou em um subconjunto específico de nós) do cluster.

### Características principais:
- ✅ Automaticamente adiciona pods em **novos nós** quando entram no cluster
- ✅ Automaticamente remove pods quando **nós são removidos** do cluster
- ✅ Útil para serviços que precisam rodar em **todos os nós**
- ✅ Ignora o scheduler normal (cria pods diretamente nos nós)

## 🎯 Casos de Uso Comuns

### 1. **Agentes de Monitoramento**
- Prometheus Node Exporter
- Datadog Agent
- New Relic Infrastructure Agent

### 2. **Coletores de Logs**
- Fluentd
- Logstash
- Filebeat

### 3. **Componentes de Rede**
- kube-proxy (componente do Kubernetes)
- Calico/Weave/Flannel (CNI plugins)
- Ingress Controllers (em alguns casos)

### 4. **Storage Daemons**
- Ceph
- GlusterFS client
- Storage drivers

## 📝 Estrutura de um DaemonSet

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
  namespace: kube-system
  labels:
    k8s-app: fluentd-logging
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      tolerations:
      # Permite rodar em nós control-plane
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      terminationGracePeriodSeconds: 30
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

## 🔑 Componentes Importantes

### Selector
```yaml
selector:
  matchLabels:
    name: fluentd-elasticsearch
```
- Identifica quais pods pertencem a este DaemonSet
- Deve corresponder às labels no `template.metadata.labels`

### Template
```yaml
template:
  metadata:
    labels:
      name: fluentd-elasticsearch
  spec:
    containers:
    - name: app
      image: myapp:1.0
```
- Define o pod que será criado em cada nó
- Mesma estrutura de um Pod spec normal

### Tolerations
```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
```
- Permite que o DaemonSet rode em nós com taints
- Essencial se você quer rodar em nós control-plane/master

## 🚀 Comandos Úteis

### Criar DaemonSet
```bash
# Via arquivo YAML
kubectl apply -f daemonset.yaml

# Não existe comando imperativo direto para DaemonSet
# Você pode gerar um template de Deployment e converter:
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml > daemonset.yaml
# Depois edite: mude "kind: Deployment" para "kind: DaemonSet"
# E remova o campo "replicas"
```

### Listar DaemonSets
```bash
# Listar todos os DaemonSets
kubectl get daemonsets

# Com namespace específico
kubectl get daemonsets -n kube-system

# Todos os namespaces
kubectl get daemonsets --all-namespaces

# Com alias
kubectl get ds
```

### Ver detalhes de um DaemonSet
```bash
kubectl describe daemonset <daemonset-name>

# Ver em namespace específico
kubectl describe ds <daemonset-name> -n kube-system
```

### Ver pods de um DaemonSet
```bash
# Listar pods com label
kubectl get pods -l name=fluentd-elasticsearch

# Ver em quais nós os pods estão
kubectl get pods -l name=fluentd-elasticsearch -o wide
```

### Editar DaemonSet
```bash
kubectl edit daemonset <daemonset-name>
```

### Deletar DaemonSet
```bash
kubectl delete daemonset <daemonset-name>

# Deletar sem remover os pods (órfãos)
kubectl delete daemonset <daemonset-name> --cascade=orphan
```

### Ver status do rollout
```bash
kubectl rollout status daemonset/<daemonset-name>

# Ver histórico
kubectl rollout history daemonset/<daemonset-name>

# Fazer rollback
kubectl rollout undo daemonset/<daemonset-name>
```

## 📊 DaemonSet vs Deployment

| Característica | DaemonSet | Deployment |
|----------------|-----------|------------|
| **Réplicas** | 1 pod por nó | Número fixo de réplicas |
| **Scheduling** | Roda em todos os nós | Distribuído pelo scheduler |
| **Scaling** | Automático (com nós) | Manual ou HPA |
| **Uso típico** | Agentes, logs, monitoring | Aplicações stateless |
| **Campo replicas** | ❌ Não existe | ✅ Obrigatório |

## 🎯 Rodando DaemonSets em Nós Específicos

### 1. Usando nodeSelector
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: ssd-monitor
spec:
  selector:
    matchLabels:
      app: ssd-monitor
  template:
    metadata:
      labels:
        app: ssd-monitor
    spec:
      nodeSelector:
        disktype: ssd    # Roda apenas em nós com label disktype=ssd
      containers:
      - name: monitor
        image: ssd-monitor:1.0
```

**Criar label em nó:**
```bash
kubectl label nodes node1 disktype=ssd
```

### 2. Usando Node Affinity
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: gpu-monitor
spec:
  selector:
    matchLabels:
      app: gpu-monitor
  template:
    metadata:
      labels:
        app: gpu-monitor
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: gpu
                operator: In
                values:
                - nvidia
                - amd
      containers:
      - name: monitor
        image: gpu-monitor:1.0
```

### 3. Usando Tolerations para Control Plane
```yaml
tolerations:
# Permite rodar em nós control-plane (master)
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule

# Permite rodar em nós com taint específico
- key: dedicated
  operator: Equal
  value: monitoring
  effect: NoSchedule
```

## 🔄 Estratégias de Update

### RollingUpdate (Padrão)
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
spec:
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1    # Quantos pods podem estar indisponíveis ao mesmo tempo
  selector:
    matchLabels:
      app: fluentd
  template:
    metadata:
      labels:
        app: fluentd
    spec:
      containers:
      - name: fluentd
        image: fluentd:v1.2
```

**Comportamento:**
- Atualiza pods gradualmente
- Deleta 1 pod, espera o novo ficar pronto, depois passa para o próximo
- `maxUnavailable`: controla quantos pods podem estar down simultaneamente

### OnDelete
```yaml
spec:
  updateStrategy:
    type: OnDelete
```

**Comportamento:**
- Pods **não** são atualizados automaticamente
- Você precisa deletar manualmente cada pod
- O DaemonSet cria um novo pod com a nova versão

**Útil para:**
- Updates controlados manualmente
- Quando você quer testar em um nó antes de propagar

```bash
# Atualizar pod em nó específico
kubectl delete pod <pod-name> -n kube-system
# DaemonSet cria automaticamente um novo pod com nova versão
```

## 🧪 Exemplo Prático: DaemonSet de Monitoring

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: node-exporter
  namespace: monitoring
  labels:
    app: node-exporter
spec:
  selector:
    matchLabels:
      app: node-exporter
  updateStrategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: node-exporter
    spec:
      hostNetwork: true      # Usa rede do host
      hostPID: true          # Acessa PIDs do host
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: node-exporter
        image: prom/node-exporter:v1.3.1
        ports:
        - containerPort: 9100
          hostPort: 9100     # Expõe porta no host
        resources:
          limits:
            cpu: 200m
            memory: 100Mi
          requests:
            cpu: 100m
            memory: 50Mi
        volumeMounts:
        - name: proc
          mountPath: /host/proc
          readOnly: true
        - name: sys
          mountPath: /host/sys
          readOnly: true
      volumes:
      - name: proc
        hostPath:
          path: /proc
      - name: sys
        hostPath:
          path: /sys
```

**Aplicar e verificar:**
```bash
# Criar namespace
kubectl create namespace monitoring

# Aplicar DaemonSet
kubectl apply -f node-exporter.yaml

# Ver status
kubectl get daemonsets -n monitoring

# Ver pods em todos os nós
kubectl get pods -n monitoring -o wide

# Verificar se tem 1 pod por nó
kubectl get nodes
kubectl get pods -n monitoring -l app=node-exporter
```

## 🔍 Troubleshooting

### Pods não são criados em todos os nós

**1. Verificar status do DaemonSet:**
```bash
kubectl describe daemonset <name>
```

Procure por:
```
Pods Status:  3 Running / 0 Waiting / 0 Succeeded / 0 Failed
```

**2. Verificar se há taints nos nós:**
```bash
kubectl describe nodes | grep -i taint
```

Se houver taints, adicione tolerations no DaemonSet.

**3. Verificar nodeSelector ou affinity:**
```bash
kubectl get daemonset <name> -o yaml | grep -A 5 nodeSelector
```

Se houver nodeSelector, verifique se os nós têm as labels corretas:
```bash
kubectl get nodes --show-labels
```

### Pods em CrashLoopBackOff

```bash
# Ver logs
kubectl logs <pod-name>

# Ver eventos
kubectl describe pod <pod-name>

# Causas comuns:
# - Permissões insuficientes para acessar hostPath
# - Imagem incorreta
# - Recursos insuficientes
```

### DaemonSet não atualiza após mudança

```bash
# Verificar estratégia de update
kubectl get daemonset <name> -o jsonpath='{.spec.updateStrategy.type}'

# Se for "OnDelete", deletar pods manualmente
kubectl delete pod <pod-name>

# Ver status do rollout
kubectl rollout status daemonset/<name>
```

### Verificar quantos pods deveriam existir

```bash
# Ver número de nós schedulable
kubectl get nodes | grep -v "SchedulingDisabled"

# Ver status do DaemonSet
kubectl get daemonset <name>
# Compare DESIRED vs CURRENT vs READY
```

## ⚠️ Boas Práticas

### ✅ Sempre defina resources
```yaml
resources:
  limits:
    cpu: 200m
    memory: 200Mi
  requests:
    cpu: 100m
    memory: 100Mi
```
- DaemonSets rodam em todos os nós
- Sem limits, podem impactar outras workloads

### ✅ Use RollingUpdate com maxUnavailable
```yaml
updateStrategy:
  type: RollingUpdate
  rollingUpdate:
    maxUnavailable: 1
```
- Evita que todos os nós fiquem sem o daemon ao mesmo tempo

### ✅ Configure tolerations apropriadas
```yaml
tolerations:
- key: node-role.kubernetes.io/control-plane
  operator: Exists
  effect: NoSchedule
```
- Permite rodar em nós control-plane se necessário

### ✅ Use labels para organizar
```yaml
metadata:
  labels:
    app: fluentd
    component: logging
    tier: infrastructure
```

### ⚠️ Cuidado com hostPath
```yaml
volumes:
- name: varlog
  hostPath:
    path: /var/log
```
- Dá acesso ao sistema de arquivos do host
- Use apenas quando realmente necessário
- Considere implicações de segurança

### ⚠️ Cuidado com hostNetwork
```yaml
spec:
  hostNetwork: true
```
- Usa rede do host (mesma interface que o nó)
- Pode causar conflitos de porta
- Use apenas quando necessário (monitoring, networking)

## 📊 Monitoramento de DaemonSets

### Ver status geral
```bash
kubectl get daemonsets --all-namespaces

# Output:
# NAMESPACE     NAME           DESIRED   CURRENT   READY   UP-TO-DATE   AVAILABLE
# kube-system   kube-proxy     3         3         3       3            3
# kube-system   weave-net      3         3         3       3            3
```

**Colunas:**
- **DESIRED**: Número de nós onde o pod deveria rodar
- **CURRENT**: Número de nós onde o pod está rodando
- **READY**: Número de pods prontos
- **UP-TO-DATE**: Número de pods com versão atualizada
- **AVAILABLE**: Número de pods disponíveis

### Verificar distribuição por nó
```bash
# Ver quais nós têm pods do DaemonSet
kubectl get pods -o wide -l app=<label>

# Contar pods por nó
kubectl get pods -l app=<label> -o jsonpath='{range .items[*]}{.spec.nodeName}{"\n"}{end}' | sort | uniq -c
```

## 🧪 Exemplo Prático: Criando um DaemonSet de Logs

```bash
# 1. Criar arquivo YAML
cat <<EOF > fluentd-daemonset.yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd
  namespace: kube-system
spec:
  selector:
    matchLabels:
      name: fluentd
  template:
    metadata:
      labels:
        name: fluentd
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluent/fluentd:v1.14-1
        resources:
          limits:
            memory: 200Mi
          requests:
            cpu: 100m
            memory: 200Mi
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
EOF

# 2. Aplicar
kubectl apply -f fluentd-daemonset.yaml

# 3. Verificar
kubectl get daemonset -n kube-system fluentd

# 4. Ver pods
kubectl get pods -n kube-system -l name=fluentd -o wide

# 5. Ver logs de um pod específico
kubectl logs -n kube-system -l name=fluentd --tail=50

# 6. Atualizar imagem
kubectl set image daemonset/fluentd fluentd=fluent/fluentd:v1.15-1 -n kube-system

# 7. Verificar rollout
kubectl rollout status daemonset/fluentd -n kube-system
```

## 📚 Recursos para Estudo

### Documentação Oficial
- [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
- [Perform a Rolling Update on a DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/update-daemon-set/)
- [Perform a Rollback on a DaemonSet](https://kubernetes.io/docs/tasks/manage-daemon/rollback-daemon-set/)

### Comandos Rápidos de Revisão
```bash
# Criar (via conversão de Deployment)
kubectl create deployment nginx --image=nginx --dry-run=client -o yaml | \
  sed 's/Deployment/DaemonSet/g' | sed '/replicas/d' > daemonset.yaml

# Listar
kubectl get daemonsets --all-namespaces

# Descrever
kubectl describe daemonset <name>

# Ver pods
kubectl get pods -l <label-selector> -o wide

# Atualizar imagem
kubectl set image daemonset/<name> <container>=<image>

# Status do rollout
kubectl rollout status daemonset/<name>

# Rollback
kubectl rollout undo daemonset/<name>

# Deletar
kubectl delete daemonset <name>
```

## 🎯 Pontos Importantes para a Prova CKA

1. **Entender** quando usar DaemonSet vs Deployment
2. **Saber criar** um DaemonSet (converter de Deployment ou escrever YAML)
3. **Configurar tolerations** para rodar em nós com taints
4. **Usar nodeSelector** para rodar apenas em nós específicos
5. **Atualizar** DaemonSets (RollingUpdate vs OnDelete)
6. **Troubleshoot** quando pods não aparecem em todos os nós
7. **Verificar status** com `kubectl get daemonsets`

### Cenário típico na prova:
> "Crie um DaemonSet chamado 'log-collector' no namespace 'kube-system' que rode a imagem 'fluentd:latest' em todos os nós, incluindo os control-plane nodes. O container deve ter acesso a '/var/log' do host."

```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-collector
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-collector
  template:
    metadata:
      labels:
        app: log-collector
    spec:
      tolerations:
      - key: node-role.kubernetes.io/control-plane
        operator: Exists
        effect: NoSchedule
      containers:
      - name: fluentd
        image: fluentd:latest
        volumeMounts:
        - name: varlog
          mountPath: /var/log
      volumes:
      - name: varlog
        hostPath:
          path: /var/log
```

---

⬅️ **Anterior**: [replicaset-deployments.md](./replicaset-deployments.md) | ➡️ **Próximo**: [scheduling.md](./scheduling.md)
