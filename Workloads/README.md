# Workloads (Cargas de Trabalho)

Esta pasta contém guias sobre os principais objetos de carga de trabalho do Kubernetes e como gerenciá-los.

## 📚 Conteúdo

### [pods.md](./pods.md)
**Unidade básica do Kubernetes**
- O que é um Pod
- Características principais (compartilhamento de rede, volumes)
- Ciclo de vida de um Pod
- Multi-container Pods
- Init Containers
- Resources (CPU/Memory requests e limits)
- Health Checks (liveness, readiness, startup)
- Comandos essenciais kubectl

### [replicaset-deployments.md](./replicaset-deployments.md)
**Gerenciamento de réplicas e atualizações**
- ReplicaSet: garantir número de réplicas
- ReplicaSet Controller: reconciliation loop
- Deployments (RECOMENDADO para produção)
- Estratégias de deploy: RollingUpdate vs Recreate
- Rolling updates e rollbacks
- Escalonamento de réplicas
- Histórico de versões
- Troubleshooting de deployments

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

## 🎯 Importância para o Exame CKA

Workloads & Scheduling representa **15% da prova**.

Você precisa saber:
- Criar e gerenciar Pods
- Usar Deployments para atualizações
- Fazer rollback de versões
- Controlar onde pods são agendados
- Configurar health checks
- Gerenciar recursos (CPU/Memory)

## 🔗 Ordem de Estudo Sugerida

1. **[pods.md](./pods.md)** - BASE: entenda pods primeiro
2. **[replicaset-deployments.md](./replicaset-deployments.md)** - Gerenciamento de réplicas
3. **[scheduling.md](./scheduling.md)** - Controle avançado de scheduling

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

⬅️ **Anterior**: [Componentes-Worker-Nodes](../Componentes-Worker-Nodes/) | ➡️ **Próximo**: [Networking](../Networking/)
