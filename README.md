# Estudos para Certificação CKA (Certified Kubernetes Administrator)

## 📚 Sobre este Repositório

Este repositório contém material de estudo organizado para a certificação **Certified Kubernetes Administrator (CKA)**, versão Kubernetes v1.34.

**Informações da Prova:**
- ⏱️ Duração: 2 horas
- 📝 Questões: 17
- 🔗 Documentação: https://kubernetes.io/docs/home/

## ⚠️ IMPORTANTE: Escopo do Exame

**Nem tudo sobre Kubernetes está no CKA!** Veja **[ESCOPO-CKA.md](./ESCOPO-CKA.md)** para saber exatamente o que estudar.

**Tópicos COBERTOS (muita gente acha que não, mas SÃO!):**
- ✅ **HPA/VPA** (workload autoscaling)
- ✅ **Helm e Kustomize** (para instalar componentes do cluster)
- ✅ **CRDs e Operators** (instalar e configurar)
- ✅ **Gateway API** (gerenciar Ingress traffic)
- ✅ **HA control plane** (multi-master)

**Tópicos NÃO cobertos:**
- ❌ StatefulSets, DaemonSets, Jobs, CronJobs
- ❌ Cluster Autoscaler (apenas HPA/VPA)
- ❌ Service Mesh (Istio, Linkerd)
- ❌ Prometheus/Grafana (apenas `kubectl top`)
- ❌ GitOps (ArgoCD, Flux)
- ❌ Desenvolver Operators/CRDs (apenas instalar e configurar)

👉 **Leia [ESCOPO-CKA.md](./ESCOPO-CKA.md) antes de estudar!**

## 🎯 Domínios e Competências do Exame

A prova CKA é dividida nos seguintes domínios:

| Domínio | Peso | Pasta |
|---------|------|-------|
| Troubleshooting | 30% | [06-Troubleshooting](./06-Troubleshooting/) |
| Cluster Architecture, Installation & Configuration | 25% | [02-Cluster-Architecture](./02-Cluster-Architecture/) |
| Services & Networking | 20% | [04-Services-Networking](./04-Services-Networking/) |
| Workloads & Scheduling | 15% | [03-Workloads-Scheduling](./03-Workloads-Scheduling/) |
| Storage | 10% | [05-Storage](./05-Storage/) |

## 📁 Estrutura de Pastas

```
CKA-Kubernetes/
├── Conceitos-Fundamentais/            # Fundamentos e preparação
│   ├── dicas-e-links.md              # Dicas da prova e links úteis
│   ├── componentes-overview.md       # Overview dos componentes K8s
│   ├── docker-containerd.md          # Docker, ContainerD, OCI, CRI
│   └── docker-storage.md             # Docker Storage (volumes, layers, CoW)
│
├── Componentes-Control-Plane/        # Componentes do Control Plane
│   ├── etcd.md                       # Banco de dados do cluster
│   ├── backup-restore.md             # Backup e restore (ETCD, Velero)
│   ├── kube-apiserver.md             # API Server
│   ├── authentication.md             # Authentication (autenticação de usuários)
│   ├── admission-controllers.md      # Admission Controllers
│   ├── kube-controller-manager.md    # Controller Manager
│   ├── kube-scheduler.md             # Scheduler
│   ├── monitoring.md                 # Monitoring e observabilidade
│   └── cluster-upgrade.md            # Atualização de versão do cluster
│
├── Componentes-Worker-Nodes/         # Componentes dos Worker Nodes
│   ├── kubelet.md                    # Agente dos nós
│   └── kube-proxy.md                 # Proxy de rede
│
├── Workloads/                         # Cargas de trabalho
│   ├── pods.md                       # Pods e containers
│   ├── multi-container-pods.md       # Multi-container e design patterns
│   ├── resource-limits.md            # CPU/Memory requests e limits
│   ├── autoscaling.md                # HPA, VPA e Cluster Autoscaler
│   ├── configmaps-secrets.md         # ConfigMaps, Secrets e Encryption
│   ├── replicaset-deployments.md     # ReplicaSets e Deployments
│   ├── rolling-updates-rollbacks.md  # Rolling updates e rollbacks
│   ├── daemonsets.md                 # DaemonSets (1 pod por nó)
│   ├── static-pods.md                # Static Pods (gerenciados pelo kubelet)
│   ├── priority-class.md             # PriorityClass (priorização de pods)
│   ├── scheduler-profiles.md         # Scheduler Profiles (múltiplos perfis)
│   └── scheduling.md                 # Scheduling, affinity, taints
│
├── 05-Storage/                        # Storage (Armazenamento)
│   └── volumes-persistent-volumes.md # Volumes, PV/PVC, StorageClass, CSI
│
└── Networking/                        # Networking
    └── services.md                   # Services e endpoints
```

## 🚀 Dicas Essenciais para a Prova

### Aliases e Produtividade
```bash
# Configurar alias do kubectl
alias k=kubectl

# Configurar autocompletion
source <(kubectl completion bash)
complete -F __start_kubectl k
```

### Comandos Imperativos (Economiza Tempo!)
```bash
# Usar dry-run para gerar YAML
kubectl create deploy nginx --image=nginx --dry-run=client -o yaml > deploy.yaml

# Fazer backup antes de editar
kubectl get pod nginx -o yaml > pod-backup.yaml

# Explorar recursos
kubectl explain pods --recursive
```

### Durante a Prova
- ✅ Use a documentação do Kubernetes (permitida na prova)
- ✅ Estude VIM básico (editor padrão)
- ✅ Valide suas respostas antes de finalizar cada questão
- ✅ Leia as instruções pré-prova com atenção
- ✅ Use `-h` para ver exemplos: `kubectl create deploy -h`

## 📖 Como Estudar

1. **Conceitos Fundamentais**: Comece por [Conceitos-Fundamentais](./Conceitos-Fundamentais/)
   - Leia as dicas da prova
   - Entenda a arquitetura geral dos componentes
   - Compreenda Docker vs ContainerD

2. **Componentes do Control Plane**: Estude [Componentes-Control-Plane](./Componentes-Control-Plane/)
   - ETCD: banco de dados chave-valor
   - Backup e Restore: métodos completos (ETCD snapshot, Velero, declarativo)
   - Kube-API-Server: ponto central de comunicação
   - Kube-Controller-Manager: reconciliation loops
   - Kube-Scheduler: agendamento de pods
   - Cluster Upgrade: atualização de versão do Kubernetes

3. **Componentes dos Worker Nodes**: Entenda [Componentes-Worker-Nodes](./Componentes-Worker-Nodes/)
   - Kubelet: agente que roda em cada nó
   - Kube-Proxy: regras de rede e serviços

4. **Workloads**: Pratique em [Workloads](./Workloads/)
   - Pods: unidade básica do Kubernetes
   - ReplicaSets e Deployments: gerenciamento de réplicas
   - Scheduling: controle onde pods são executados

5. **Storage**: Entenda [05-Storage](./05-Storage/)
   - Volumes: emptyDir, hostPath, configMap, secret
   - Persistent Volumes (PV) e Persistent Volume Claims (PVC)
   - Storage Classes: provisionamento dinâmico
   - Container Storage Interface (CSI)
   - Access Modes, Volume Modes, Reclaim Policies

6. **Networking**: Domine [Networking](./Networking/)
   - Services: ClusterIP, NodePort, LoadBalancer
   - Endpoints e service discovery

## 🔗 Recursos Externos

### Documentação Oficial
- [Kubernetes Documentation](https://kubernetes.io/docs/home/)
- [CKA Exam Curriculum](https://github.com/cncf/curriculum)

### Cursos e Tutoriais
- [Pluralsight - Certified Kubernetes Administrator](https://www.pluralsight.com/paths/certified-kubernetes-administrator)
- Material em português disponível em [Conceitos-Fundamentais/dicas-e-links.md](./Conceitos-Fundamentais/dicas-e-links.md)

### Prática
- [Killer.sh](https://killer.sh/) - Simulados oficiais da Linux Foundation
- Laboratórios práticos com Minikube ou Kind

## 📝 Progresso dos Estudos

- [ ] Fundamentos de Kubernetes e Docker
- [ ] Componentes do Control Plane
- [ ] Backup e Restore do ETCD
- [ ] Authentication (certificados X.509, ServiceAccounts, OIDC)
- [ ] RBAC e Authorization
- [ ] Admission Controllers (validating, mutating, webhooks)
- [ ] Cluster Upgrade (atualização de versão)
- [ ] Componentes dos Worker Nodes
- [ ] Pods e Containers
- [ ] Deployments e ReplicaSets
- [ ] Services e Networking
- [ ] Storage (PV/PVC)
- [ ] ConfigMaps e Secrets
- [ ] Troubleshooting
- [ ] Simulados práticos

## 🎓 Certificação

Esta estrutura foi criada para facilitar o estudo para a certificação **CKA - Certified Kubernetes Administrator** da Cloud Native Computing Foundation (CNCF).

---

**Boa sorte nos estudos! 🚀**