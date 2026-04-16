# Workloads (Cargas de Trabalho)

Esta pasta contém guias sobre os principais objetos de carga de trabalho do Kubernetes e como gerenciá-los.

## 📚 Conteúdo

### [pods.md](./pods.md)
**Unidade básica do Kubernetes**
- O que é um Pod
- Características principais (compartilhamento de rede, volumes)
- Ciclo de vida de um Pod
- Resources (CPU/Memory requests e limits)
- Health Checks (liveness, readiness, startup)
- Comandos essenciais kubectl

### [multi-container-pods.md](./multi-container-pods.md)
**Pods com múltiplos containers e design patterns**
- O que são Multi-Container Pods
- Design Patterns: Sidecar, Ambassador, Adapter
- Init Containers (inicialização e setup)
- Volumes compartilhados entre containers
- Comunicação: localhost, filesystem, IPC
- Exemplos práticos (Nginx + Fluentd, Git Sync, Cache)
- Comandos para logs e exec em containers específicos

### [replicaset-deployments.md](./replicaset-deployments.md)
**Gerenciamento de réplicas e atualizações**
- ReplicaSet: garantir número de réplicas
- ReplicaSet Controller: reconciliation loop
- Deployments (RECOMENDADO para produção)
- Estratégias de deploy: RollingUpdate vs Recreate
- Escalonamento de réplicas
- Histórico de versões
- Troubleshooting de deployments

### [rolling-updates-rollbacks.md](./rolling-updates-rollbacks.md)
**Atualizações sem downtime e reversão de mudanças**
- O que são Rolling Updates (zero downtime)
- Estratégias: RollingUpdate vs Recreate
- maxUnavailable e maxSurge (controle de pods)
- Realizar updates (kubectl set image, edit, apply)
- Monitorar rollouts (kubectl rollout status)
- Pausar e retomar rollouts
- Rollbacks (kubectl rollout undo)
- Histórico de revisões (kubectl rollout history)
- Troubleshooting de rollouts travados

### [configmaps-secrets.md](./configmaps-secrets.md)
**Configuração e dados sensíveis**
- ConfigMaps: dados de configuração não-sensíveis
- Secrets: dados sensíveis (senhas, tokens, chaves)
- Criar ConfigMaps e Secrets (literal, arquivo, YAML)
- Usar como environment variables
- Montar como volumes (arquivos)
- Tipos de Secrets (Opaque, TLS, Docker Registry)
- Encryption at Rest (criptografar secrets no etcd)
- Base64 encoding/decoding

### [daemonsets.md](./daemonsets.md)
**Pods que rodam em todos os nós**
- O que é um DaemonSet (1 pod por nó)
- Casos de uso: monitoring, logging, networking
- DaemonSet vs Deployment
- Rodando em nós específicos (nodeSelector, affinity)
- Tolerations para nós control-plane
- Estratégias de update (RollingUpdate vs OnDelete)
- Troubleshooting de DaemonSets

### [static-pods.md](./static-pods.md)
**Pods gerenciados pelo kubelet (não pelo API Server)**
- O que são Static Pods
- Como funcionam e onde são definidos
- Static Pods vs DaemonSets vs Pods normais
- Criar, editar e deletar Static Pods
- Identificar Static Pods (nome com sufixo do nó, ownerReferences)
- Uso no Control Plane (kube-apiserver, etcd, etc.)
- Troubleshooting de Static Pods

### [priority-class.md](./priority-class.md)
**Priorização de pods no scheduling e eviction**
- O que é PriorityClass (prioridade numérica de pods)
- Scheduling: pods de alta prioridade são agendados primeiro
- Preemption: pods podem evict outros de menor prioridade
- PriorityClass vs QoS Class (conceitos diferentes)
- Criar e usar PriorityClass em pods/deployments
- PriorityClasses do sistema (system-cluster-critical, system-node-critical)
- Troubleshooting e boas práticas

### [scheduler-profiles.md](./scheduler-profiles.md)
**Múltiplos perfis de scheduling em um único scheduler**
- O que são Scheduler Profiles
- Plugins do scheduler (Filter, Score, Bind, etc.)
- Configurar profiles com diferentes estratégias
- Scoring strategies (LeastAllocated, MostAllocated, RequestedToCapacityRatio)
- Usar schedulerName para escolher profile
- Casos de uso: GPU bin-packing, HA spreading, batch jobs
- Debug de scheduling failures

### [scheduling.md](./scheduling.md)
**Controle sobre onde os pods são executados**
- Scheduling manual
- nodeSelector (abordagem simples)
- Node Affinity (abordagem avançada)
- Pod Affinity e Anti-Affinity
- Taints e Tolerations
- Topology Spread Constraints
- Comandos úteis para debugging
- Cenários práticos

### [resource-limits.md](./resource-limits.md)
**Gerenciamento de recursos (CPU e Memory)**
- Diferença entre Requests e Limits
- Unidades de medida (CPU millicores, Memory Mi/Gi)
- QoS Classes (Guaranteed, Burstable, BestEffort)
- LimitRange: limites padrão por namespace
- ResourceQuota: limites totais por namespace
- Troubleshooting de OOMKilled e recursos insuficientes
- Boas práticas de resource management

### [image-security.md](./image-security.md)
**Segurança de imagens de container**
- Convenção de nomes de imagens (registry/usuário/nome:tag)
- Registries privados: criar Secret `docker-registry`
- Usar `imagePullSecrets` em pods
- ImagePullPolicy (Always, IfNotPresent, Never)
- Boas práticas: tags fixas, escaneamento de vulnerabilidades

### [security-context.md](./security-context.md)
**Configurações de segurança para Pods e containers**
- SecurityContext no nível de Pod e de container
- `runAsUser`, `runAsGroup`, `runAsNonRoot`
- `allowPrivilegeEscalation`, `readOnlyRootFilesystem`
- Linux Capabilities (add/drop)
- Exemplo seguro para produção

### [autoscaling.md](./autoscaling.md)
**Escalabilidade automática de pods e clusters**
- HPA (Horizontal Pod Autoscaler): escala número de pods
- VPA (Vertical Pod Autoscaler): escala recursos dos pods
- Cluster Autoscaler: escala número de nós
- Metrics Server (pré-requisito para HPA)
- Métricas: CPU, memória, custom, external
- Fórmula de cálculo e comportamento
- HPA vs VPA: quando usar cada um
- Troubleshooting de autoscaling

## 🎯 Importância para o Exame CKA

Workloads & Scheduling representa **15% da prova**.

Você precisa saber:
- Criar e gerenciar Pods
- Usar Deployments para atualizações
- Fazer rollback de versões
- Controlar onde pods são agendados
- Configurar health checks
- Gerenciar recursos (CPU/Memory requests e limits)
- Troubleshoot problemas de recursos (OOMKilled, Pending)

## 🔗 Ordem de Estudo Sugerida

1. **[pods.md](./pods.md)** - BASE: entenda pods primeiro
2. **[multi-container-pods.md](./multi-container-pods.md)** - Múltiplos containers e patterns
3. **[resource-limits.md](./resource-limits.md)** - Gerenciamento de recursos (IMPORTANTE!)
4. **[autoscaling.md](./autoscaling.md)** - Escalabilidade automática (HPA, VPA)
5. **[configmaps-secrets.md](./configmaps-secrets.md)** - Configuração e segurança (IMPORTANTE!)
6. **[replicaset-deployments.md](./replicaset-deployments.md)** - Gerenciamento de réplicas
7. **[rolling-updates-rollbacks.md](./rolling-updates-rollbacks.md)** - Updates e rollbacks (IMPORTANTE!)
8. **[daemonsets.md](./daemonsets.md)** - Pods em todos os nós
9. **[static-pods.md](./static-pods.md)** - Pods gerenciados pelo kubelet
10. **[priority-class.md](./priority-class.md)** - Priorização de pods
11. **[scheduler-profiles.md](./scheduler-profiles.md)** - Configuração do scheduler
12. **[scheduling.md](./scheduling.md)** - Controle avançado de scheduling

## 💡 Conceitos-Chave

### Pods
```bash
# Criar pod imperativo (rápido)
kubectl run nginx --image=nginx

# Gerar YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml > pod.yaml

# Health checks
livenessProbe   # Reinicia container se falhar
readinessProbe  # Remove de Service se não estiver pronto
startupProbe    # Protege durante inicialização lenta
```

### Multi-Container Pods
```bash
# Ver logs de container específico
kubectl logs <pod> -c <container-name>

# Exec em container específico
kubectl exec <pod> -c <container-name> -- command

# Ver status de todos os containers
kubectl describe pod <pod>

# Pod com init container
spec:
  initContainers:
  - name: init
    image: busybox
    command: ["sh", "-c", "sleep 10"]
  containers:
  - name: main
    image: nginx

# Volume compartilhado
volumes:
- name: shared
  emptyDir: {}
```

### Autoscaling (HPA)
```bash
# Criar HPA
kubectl autoscale deployment nginx --cpu-percent=50 --min=2 --max=10

# Ver status
kubectl get hpa
kubectl get hpa nginx --watch

# Descrever HPA
kubectl describe hpa nginx

# Ver métricas (requer Metrics Server)
kubectl top nodes
kubectl top pods

# Deletar HPA
kubectl delete hpa nginx
```

### Deployments
```bash
# Criar deployment
kubectl create deployment nginx --image=nginx --replicas=3

# Atualizar imagem (trigger rollout)
kubectl set image deployment/nginx nginx=nginx:1.16.1

# Rollback se deu problema
kubectl rollout undo deployment/nginx

# Ver histórico
kubectl rollout history deployment/nginx
```

### Rolling Updates e Rollbacks
```bash
# Fazer rolling update
kubectl set image deployment/nginx nginx=nginx:1.21 --record

# Ver status do rollout
kubectl rollout status deployment/nginx

# Pausar rollout
kubectl rollout pause deployment/nginx

# Retomar rollout
kubectl rollout resume deployment/nginx

# Ver histórico de revisões
kubectl rollout history deployment/nginx

# Rollback para versão anterior
kubectl rollout undo deployment/nginx

# Rollback para revisão específica
kubectl rollout undo deployment/nginx --to-revision=2
```

### ConfigMaps e Secrets
```bash
# Criar ConfigMap
kubectl create configmap app-config --from-literal=APP_ENV=prod

# Criar Secret
kubectl create secret generic db-secret --from-literal=password=secret123

# Ver Secret (dados ocultos)
kubectl get secret db-secret

# Decodificar Secret
kubectl get secret db-secret -o jsonpath='{.data.password}' | base64 --decode

# Usar em pod como env
env:
- name: DB_PASS
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password

# Montar como volume
volumeMounts:
- name: config
  mountPath: /etc/config
volumes:
- name: config
  configMap:
    name: app-config
```

### DaemonSets
```bash
# Listar DaemonSets
kubectl get daemonsets --all-namespaces

# Ver pods do DaemonSet em todos os nós
kubectl get pods -l <label> -o wide

# Atualizar DaemonSet
kubectl set image daemonset/<name> <container>=<image>

# Ver status
kubectl rollout status daemonset/<name>
```

### Static Pods
```bash
# Descobrir diretório de static pods
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Criar static pod
vi /etc/kubernetes/manifests/my-pod.yaml

# Deletar static pod (remover arquivo)
rm /etc/kubernetes/manifests/my-pod.yaml

# Identificar static pods (nome tem sufixo do nó)
kubectl get pods -A -o wide | grep <node-name>
```

### PriorityClass
```bash
# Criar PriorityClass
kubectl apply -f priorityclass.yaml

# Listar PriorityClasses
kubectl get priorityclasses

# Ver prioridades dos pods
kubectl get pods -o custom-columns=NAME:.metadata.name,PRIORITY:.spec.priority

# Usar em pod
spec:
  priorityClassName: high-priority
```

### Scheduler Profiles
```bash
# Ver qual scheduler um pod usa
kubectl get pod <name> -o jsonpath='{.spec.schedulerName}'

# Ver todos os pods e seus schedulers
kubectl get pods -A -o custom-columns=NAME:.metadata.name,SCHEDULER:.spec.schedulerName

# Usar scheduler específico em pod
spec:
  schedulerName: custom-scheduler

# Ver logs do scheduler
kubectl logs -n kube-system kube-scheduler-<node>
```

### Scheduling
```bash
# nodeSelector (simples)
nodeSelector:
  disktype: ssd

# Taint (marcar nó)
kubectl taint nodes node1 key=value:NoSchedule

# Toleration (permitir pod em nó tainted)
tolerations:
- key: "key"
  operator: "Equal"
  value: "value"
  effect: "NoSchedule"
```

## 🚨 Troubleshooting Comum

### Pod não inicia
```bash
# Ver eventos
kubectl describe pod <pod-name>

# Ver logs
kubectl logs <pod-name>

# Se CrashLoopBackOff
kubectl logs <pod-name> --previous
```

### Deployment não atualiza
```bash
# Ver status do rollout
kubectl rollout status deployment/<name>

# Ver histórico
kubectl rollout history deployment/<name>

# Pausar/retomar
kubectl rollout pause deployment/<name>
kubectl rollout resume deployment/<name>
```

### Pod em Pending
```bash
# Verificar por que não foi agendado
kubectl describe pod <pod-name>
# Procure por: "FailedScheduling"

# Causas comuns:
# - Recursos insuficientes
# - nodeSelector não match
# - Taint sem toleration
```

---

⬅️ **Anterior**: [02-Cluster-Architecture](../02-Cluster-Architecture/) | ➡️ **Próximo**: [04-Services-Networking](../04-Services-Networking/)
