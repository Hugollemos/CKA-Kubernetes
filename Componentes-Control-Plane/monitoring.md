# Monitoring e Observabilidade do Cluster

## 📋 O que é Monitoramento no Kubernetes?

**Monitoramento** envolve coletar, visualizar e analisar métricas e logs dos componentes do cluster para garantir saúde, performance e troubleshooting eficiente.

### Objetivos do Monitoramento:
- ✅ Verificar **saúde** dos componentes (API Server, etcd, scheduler, etc.)
- ✅ Monitorar **recursos** (CPU, memória, disco, rede)
- ✅ Coletar **logs** para troubleshooting
- ✅ Detectar **problemas** antes que afetem usuários
- ✅ Analisar **performance** e capacidade

## 🎯 Componentes a Monitorar

### Control Plane Components
1. **kube-apiserver**: Latência de requisições, taxa de erro, throughput
2. **etcd**: Latência de operações, uso de disco, sincronização
3. **kube-controller-manager**: Status dos controllers, reconciliation loops
4. **kube-scheduler**: Latência de scheduling, pods pending
5. **cloud-controller-manager**: Integração com cloud provider

### Worker Node Components
1. **kubelet**: Status dos pods, uso de recursos do nó
2. **kube-proxy**: Regras de iptables/ipvs, performance de rede
3. **container runtime**: Docker/containerd, pull de imagens

### Add-ons e Workloads
1. **CoreDNS**: Resolução de nomes, cache DNS
2. **Pods de aplicação**: CPU, memória, restarts, health checks
3. **Persistent Volumes**: Uso de storage

## 📊 Metrics Server

### O que é Metrics Server?

**Metrics Server** é um agregador de métricas de recursos do cluster. Coleta métricas de CPU e memória dos nós e pods através do kubelet.

### Características:
- ✅ Coleta métricas de **kubelet** em cada nó
- ✅ Expõe via **Metrics API** (`metrics.k8s.io`)
- ✅ Usado por **kubectl top** e **HPA** (Horizontal Pod Autoscaler)
- ✅ **Não** armazena histórico (apenas dados atuais)
- ✅ Leve e eficiente

### Instalar Metrics Server

```bash
# Método 1: Via componentes oficiais
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Método 2: Via Helm
helm repo add metrics-server https://kubernetes-sigs.github.io/metrics-server/
helm upgrade --install metrics-server metrics-server/metrics-server -n kube-system

# Verificar se está rodando
kubectl get deployment metrics-server -n kube-system

# Ver logs
kubectl logs -n kube-system deployment/metrics-server
```

### Verificar se Metrics Server está funcionando

```bash
# Testar a API
kubectl get apiservices | grep metrics

# Deve mostrar:
# v1beta1.metrics.k8s.io   kube-system/metrics-server   True

# Ver métricas dos nós
kubectl top nodes

# Ver métricas dos pods
kubectl top pods -A

# Ver métricas de um pod específico
kubectl top pod <pod-name> -n <namespace>
```

## 🔍 kubectl top - Visualizar Métricas

### kubectl top nodes

```bash
# Ver uso de recursos de todos os nós
kubectl top nodes

# Output:
# NAME           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# controlplane   250m         12%    1024Mi          54%
# node01         180m         9%     768Mi           40%
# node02         220m         11%    896Mi           47%

# Ordenar por CPU
kubectl top nodes --sort-by=cpu

# Ordenar por memória
kubectl top nodes --sort-by=memory
```

### kubectl top pods

```bash
# Ver uso de recursos de pods no namespace atual
kubectl top pods

# Ver pods de todos os namespaces
kubectl top pods -A

# Ver pods de um namespace específico
kubectl top pods -n kube-system

# Ordenar por CPU
kubectl top pods --sort-by=cpu

# Ordenar por memória
kubectl top pods --sort-by=memory

# Ver pods com containers
kubectl top pods --containers

# Output:
# POD            CONTAINER   CPU(cores)   MEMORY(bytes)
# nginx-pod      nginx       10m          32Mi
# app-pod        app         50m          128Mi
# app-pod        sidecar     5m           16Mi
```

## 📝 Logs dos Componentes

### Logs via kubectl logs

```bash
# Ver logs de um pod
kubectl logs <pod-name>

# Ver logs de um container específico (multi-container pod)
kubectl logs <pod-name> -c <container-name>

# Ver logs anteriores (se pod crashou e reiniciou)
kubectl logs <pod-name> --previous

# Follow logs (tail -f)
kubectl logs <pod-name> -f

# Ver últimas 100 linhas
kubectl logs <pod-name> --tail=100

# Ver logs desde uma timestamp
kubectl logs <pod-name> --since=1h
kubectl logs <pod-name> --since=2024-02-26T10:00:00Z

# Ver logs de todos os pods com label
kubectl logs -l app=nginx

# Ver logs de deployment
kubectl logs deployment/<deployment-name>
```

### Logs dos Componentes do Control Plane

Os componentes do control plane geralmente rodam como **static pods**:

```bash
# Ver logs do API Server
kubectl logs -n kube-system kube-apiserver-<node-name>

# Ver logs do etcd
kubectl logs -n kube-system etcd-<node-name>

# Ver logs do Controller Manager
kubectl logs -n kube-system kube-controller-manager-<node-name>

# Ver logs do Scheduler
kubectl logs -n kube-system kube-scheduler-<node-name>

# Ver logs do CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns

# Ver logs do kube-proxy
kubectl logs -n kube-system -l k8s-app=kube-proxy
```

### Logs via journalctl (se não são static pods)

```bash
# Logs do kubelet (sempre via journalctl)
sudo journalctl -u kubelet -f

# Ver últimas 100 linhas do kubelet
sudo journalctl -u kubelet -n 100

# Ver logs desde 1 hora atrás
sudo journalctl -u kubelet --since "1 hour ago"

# Ver logs do container runtime (containerd)
sudo journalctl -u containerd -f

# Ver logs do docker (se usando docker)
sudo journalctl -u docker -f
```

## 🔔 Events - Sistema de Eventos do Kubernetes

### Ver Events

```bash
# Ver todos os eventos do cluster
kubectl get events -A

# Ver eventos do namespace atual
kubectl get events

# Ver eventos de um namespace específico
kubectl get events -n kube-system

# Ordenar por timestamp
kubectl get events --sort-by='.lastTimestamp'

# Ver apenas eventos de Warning
kubectl get events -A --field-selector type=Warning

# Ver apenas eventos de Normal
kubectl get events -A --field-selector type=Normal

# Ver eventos de um objeto específico
kubectl get events --field-selector involvedObject.name=<pod-name>

# Ver eventos nos últimos 10 minutos
kubectl get events --sort-by='.lastTimestamp' | head -20
```

### Tipos de Events Comuns

**Warning Events (problemas):**
```bash
# FailedScheduling: Pod não pode ser agendado
# BackOff: Container está em CrashLoopBackOff
# Failed: Falha ao criar pod
# Unhealthy: Health check falhou
# FailedMount: Falha ao montar volume
# FailedAttachVolume: Falha ao atachar volume
```

**Normal Events (informativo):**
```bash
# Scheduled: Pod foi agendado em um nó
# Pulled: Imagem foi baixada
# Created: Container foi criado
# Started: Container iniciou
# Killing: Container está sendo terminado
```

### Exemplo de Troubleshooting com Events

```bash
# 1. Ver pods com problema
kubectl get pods | grep -E "Error|CrashLoop|Pending"

# 2. Ver eventos do pod
kubectl describe pod <pod-name> | grep -A 20 "Events:"

# 3. Ver todos os eventos de Warning
kubectl get events --field-selector type=Warning --sort-by='.lastTimestamp'

# 4. Ver eventos relacionados a scheduling
kubectl get events -A | grep -i "FailedScheduling"

# 5. Ver eventos relacionados a volumes
kubectl get events -A | grep -i "volume\|mount"
```

## 🏥 Health Checks dos Componentes

### API Server Health

```bash
# Health check via API
kubectl get --raw='/healthz'
# Output: ok

# Health check detalhado
kubectl get --raw='/healthz?verbose'

# Livez (liveness)
kubectl get --raw='/livez?verbose'

# Readyz (readiness)
kubectl get --raw='/readyz?verbose'

# Via curl (se tiver acesso direto)
curl -k https://localhost:6443/healthz
```

### etcd Health

```bash
# Via etcdctl (dentro do pod do etcd)
kubectl exec -n kube-system etcd-<node-name> -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  endpoint health

# Output:
# https://127.0.0.1:2379 is healthy: successfully committed proposal

# Ver status dos membros do cluster etcd
kubectl exec -n kube-system etcd-<node-name> -- \
  etcdctl --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  member list
```

### Controller Manager e Scheduler Health

```bash
# Via healthz endpoint
kubectl get --raw='/healthz/controller-manager'
kubectl get --raw='/healthz/scheduler'

# Verificar se estão rodando
kubectl get pods -n kube-system | grep -E "controller-manager|scheduler"
```

### Kubelet Health

```bash
# No nó (via systemd)
sudo systemctl status kubelet

# Ver se kubelet está registrado no cluster
kubectl get nodes

# Verificar condições do nó
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# Output esperado:
# Conditions:
#   Type             Status
#   ----             ------
#   MemoryPressure   False
#   DiskPressure     False
#   PIDPressure      False
#   Ready            True
```

## 📊 Monitoramento de Recursos do Nó

### Node Conditions

```bash
# Ver condições de todos os nós
kubectl get nodes -o custom-columns=\
NAME:.metadata.name,\
STATUS:.status.conditions[?(@.type==\"Ready\")].status,\
MEMORY:.status.conditions[?(@.type==\"MemoryPressure\")].status,\
DISK:.status.conditions[?(@.type==\"DiskPressure\")].status

# Descrever nó para ver detalhes
kubectl describe node <node-name>
```

### Capacity vs Allocatable

```bash
# Ver capacidade e recursos alocáveis dos nós
kubectl describe nodes | grep -A 5 "Capacity\|Allocatable"

# Output:
# Capacity:
#   cpu:                4
#   memory:             8Gi
#   pods:               110
# Allocatable:
#   cpu:                4
#   memory:             7.5Gi
#   pods:               110

# Ver em formato JSON
kubectl get nodes -o json | jq '.items[] | {
  name: .metadata.name,
  capacity: .status.capacity,
  allocatable: .status.allocatable
}'
```

### Recursos Solicitados vs Usados

```bash
# Ver recursos solicitados (requests) nos nós
kubectl describe nodes | grep -A 15 "Allocated resources:"

# Output:
# Allocated resources:
#   (Total limits may be over 100 percent)
#   Resource           Requests      Limits
#   --------           --------      ------
#   cpu                1500m (37%)   3000m (75%)
#   memory             2Gi (26%)     4Gi (53%)

# Ver uso real (com Metrics Server)
kubectl top nodes
```

## 🔧 Troubleshooting com Monitoramento

### Cenário 1: Pod em CrashLoopBackOff

```bash
# 1. Ver status do pod
kubectl get pod <pod-name>

# 2. Ver eventos
kubectl describe pod <pod-name> | grep -A 20 "Events:"

# 3. Ver logs do container atual
kubectl logs <pod-name>

# 4. Ver logs do container anterior (que crashou)
kubectl logs <pod-name> --previous

# 5. Ver uso de recursos
kubectl top pod <pod-name>
```

### Cenário 2: Nó com MemoryPressure

```bash
# 1. Ver condição do nó
kubectl describe node <node-name> | grep -A 10 "Conditions:"

# 2. Ver uso de memória
kubectl top node <node-name>

# 3. Ver pods no nó ordenados por memória
kubectl top pods -A --sort-by=memory | grep <node-name>

# 4. Ver pods que podem ser evicted
kubectl get pods -A -o wide --field-selector spec.nodeName=<node-name>

# 5. Ver eventos do nó
kubectl get events --field-selector involvedObject.name=<node-name>
```

### Cenário 3: API Server lento

```bash
# 1. Ver logs do API Server
kubectl logs -n kube-system kube-apiserver-<node>

# 2. Testar latência
time kubectl get nodes

# 3. Verificar health do etcd
kubectl exec -n kube-system etcd-<node> -- etcdctl endpoint health

# 4. Ver uso de recursos do API Server
kubectl top pod -n kube-system kube-apiserver-<node>

# 5. Ver se há muitas requisições
kubectl logs -n kube-system kube-apiserver-<node> | grep -i "request"
```

### Cenário 4: Scheduler não agendando pods

```bash
# 1. Ver pods em Pending
kubectl get pods -A --field-selector status.phase=Pending

# 2. Ver por que não foram agendados
kubectl describe pod <pod-name> | grep -A 10 "Events:"

# 3. Ver logs do scheduler
kubectl logs -n kube-system kube-scheduler-<node>

# 4. Ver recursos disponíveis nos nós
kubectl describe nodes | grep -A 5 "Allocatable"

# 5. Verificar taints nos nós
kubectl describe nodes | grep -i taint
```

## 📈 Ferramentas de Monitoramento Avançadas

### Prometheus + Grafana (stack completo)

```bash
# Instalar via Helm (kube-prometheus-stack)
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
helm install kube-prometheus prometheus-community/kube-prometheus-stack -n monitoring --create-namespace

# Acessar Grafana
kubectl port-forward -n monitoring svc/kube-prometheus-grafana 3000:80

# Acessar Prometheus
kubectl port-forward -n monitoring svc/kube-prometheus-kube-prome-prometheus 9090:9090
```

**Métricas disponíveis:**
- CPU, memória, disco, rede de nós e pods
- Latência de API Server
- Performance do etcd
- Taxa de erro de controllers
- Dashboards prontos no Grafana

### Elasticsearch + Fluentd + Kibana (EFK Stack)

**Para agregação de logs:**
- Fluentd coleta logs de todos os containers
- Elasticsearch armazena e indexa logs
- Kibana visualiza e busca logs

### Outros Tools

**Weave Scope**: Visualização gráfica do cluster
**K9s**: Terminal UI para Kubernetes
**Lens**: IDE para Kubernetes
**Octant**: Web UI para Kubernetes

## 🧪 Comandos Práticos de Monitoramento

### Script de Health Check Completo

```bash
#!/bin/bash
echo "=== KUBERNETES CLUSTER HEALTH CHECK ==="
echo ""

echo "1. Cluster Info"
kubectl cluster-info
echo ""

echo "2. Nodes Status"
kubectl get nodes
echo ""

echo "3. System Pods Status"
kubectl get pods -n kube-system
echo ""

echo "4. Top Nodes (Resources)"
kubectl top nodes 2>/dev/null || echo "Metrics Server not available"
echo ""

echo "5. Top Pods (kube-system)"
kubectl top pods -n kube-system 2>/dev/null || echo "Metrics Server not available"
echo ""

echo "6. Recent Warning Events"
kubectl get events -A --field-selector type=Warning --sort-by='.lastTimestamp' | tail -10
echo ""

echo "7. Pods NOT Running"
kubectl get pods -A | grep -v "Running\|Completed"
echo ""

echo "8. Component Status (deprecated but useful)"
kubectl get cs 2>/dev/null || echo "ComponentStatus API removed in recent versions"
echo ""

echo "9. API Server Health"
kubectl get --raw='/healthz'
echo ""

echo "=== END OF HEALTH CHECK ==="
```

### Monitoramento Contínuo

```bash
# Watch pods em tempo real
watch kubectl get pods -A

# Watch nodes em tempo real
watch kubectl top nodes

# Watch events em tempo real
kubectl get events -A --watch

# Follow logs de múltiplos pods
kubectl logs -f -l app=nginx --all-containers=true

# Monitor de recursos em loop
while true; do
  clear
  echo "=== $(date) ==="
  kubectl top nodes
  echo ""
  kubectl top pods -A --sort-by=memory | head -20
  sleep 5
done
```

## 📚 Recursos para Estudo

### Documentação Oficial
- [Metrics Server](https://github.com/kubernetes-sigs/metrics-server)
- [Monitoring Architecture](https://kubernetes.io/docs/tasks/debug/debug-cluster/resource-metrics-pipeline/)
- [Logging Architecture](https://kubernetes.io/docs/concepts/cluster-administration/logging/)
- [System Logs](https://kubernetes.io/docs/concepts/cluster-administration/system-logs/)
- [Troubleshooting Clusters](https://kubernetes.io/docs/tasks/debug/debug-cluster/)

### Comandos Rápidos de Revisão

```bash
# Métricas
kubectl top nodes
kubectl top pods -A

# Logs
kubectl logs <pod-name>
kubectl logs <pod-name> --previous
sudo journalctl -u kubelet

# Events
kubectl get events -A --sort-by='.lastTimestamp'
kubectl get events --field-selector type=Warning

# Health
kubectl get --raw='/healthz'
kubectl get nodes
kubectl get pods -n kube-system

# Describe (troubleshooting detalhado)
kubectl describe node <node-name>
kubectl describe pod <pod-name>
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Usar kubectl top**
   ```bash
   kubectl top nodes
   kubectl top pods -A
   ```

2. **Ver logs de componentes**
   ```bash
   kubectl logs -n kube-system <pod-name>
   sudo journalctl -u kubelet
   ```

3. **Analisar events**
   ```bash
   kubectl get events --sort-by='.lastTimestamp'
   kubectl describe pod <name>
   ```

4. **Verificar health do cluster**
   ```bash
   kubectl get nodes
   kubectl get pods -n kube-system
   kubectl get --raw='/healthz'
   ```

5. **Troubleshoot pods com problemas**
   ```bash
   kubectl describe pod <name>
   kubectl logs <name>
   kubectl logs <name> --previous
   ```

6. **Ver recursos dos nós**
   ```bash
   kubectl describe nodes
   kubectl top nodes
   ```

### 🧪 Cenários típicos na prova:

> **"Há um pod em CrashLoopBackOff no namespace 'production'. Investigue o problema."**

```bash
# 1. Ver o pod
kubectl get pods -n production

# 2. Ver detalhes e eventos
kubectl describe pod <pod-name> -n production

# 3. Ver logs atuais
kubectl logs <pod-name> -n production

# 4. Ver logs anteriores (do crash)
kubectl logs <pod-name> -n production --previous

# 5. Ver uso de recursos
kubectl top pod <pod-name> -n production
```

> **"Verifique a saúde do cluster e identifique componentes com problema."**

```bash
# 1. Ver status dos nós
kubectl get nodes

# 2. Ver pods do sistema
kubectl get pods -n kube-system

# 3. Ver eventos de warning
kubectl get events -A --field-selector type=Warning

# 4. Ver health do API Server
kubectl get --raw='/healthz'

# 5. Ver logs dos componentes
kubectl logs -n kube-system kube-apiserver-<node>
```

## 💡 Dicas para a Prova

1. **Instale o Metrics Server se não estiver disponível**
   - É fundamental para `kubectl top`

2. **Use describe antes de logs**
   - `kubectl describe` mostra eventos úteis

3. **Logs anteriores são importantes**
   - Use `--previous` para ver logs de containers que crasharam

4. **journalctl para kubelet**
   - Kubelet não tem pod, logs via systemd

5. **Events têm TTL**
   - Eventos são deletados após 1 hora
   - Use `--sort-by` para ver os mais recentes

---

⬅️ **Anterior**: [admission-controllers.md](./admission-controllers.md) | ➡️ **Próximo**: [../Componentes-Worker-Nodes/](../Componentes-Worker-Nodes/)
