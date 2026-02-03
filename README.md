# Estudos para Certificação CKA (Certified Kubernetes Administrator)

## 📚 Sobre este Repositório

Este repositório contém material de estudo organizado para a certificação **Certified Kubernetes Administrator (CKA)**, versão Kubernetes v1.34.

**Informações da Prova:**
- ⏱️ Duração: 2 horas
- 📝 Questões: 17
- 🔗 Documentação: https://kubernetes.io/docs/home/

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
│   └── docker-containerd.md          # Docker, ContainerD, OCI, CRI
│
├── Componentes-Control-Plane/        # Componentes do Control Plane
│   ├── etcd.md                       # Banco de dados do cluster
│   ├── kube-apiserver.md             # API Server
│   ├── kube-controller-manager.md    # Controller Manager
│   └── kube-scheduler.md             # Scheduler
│
├── Componentes-Worker-Nodes/         # Componentes dos Worker Nodes
│   ├── kubelet.md                    # Agente dos nós
│   └── kube-proxy.md                 # Proxy de rede
│
├── Workloads/                         # Cargas de trabalho
│   ├── pods.md                       # Pods e containers
│   ├── replicaset-deployments.md     # ReplicaSets e Deployments
│   └── scheduling.md                 # Scheduling, affinity, taints
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
   - Kube-API-Server: ponto central de comunicação
   - Kube-Controller-Manager: reconciliation loops
   - Kube-Scheduler: agendamento de pods

3. **Componentes dos Worker Nodes**: Entenda [Componentes-Worker-Nodes](./Componentes-Worker-Nodes/)
   - Kubelet: agente que roda em cada nó
   - Kube-Proxy: regras de rede e serviços

4. **Workloads**: Pratique em [Workloads](./Workloads/)
   - Pods: unidade básica do Kubernetes
   - ReplicaSets e Deployments: gerenciamento de réplicas
   - Scheduling: controle onde pods são executados

5. **Networking**: Domine [Networking](./Networking/)
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
- [ ] Componentes dos Worker Nodes
- [ ] Pods e Containers
- [ ] Deployments e ReplicaSets
- [ ] Services e Networking
- [ ] Storage (PV/PVC)
- [ ] ConfigMaps e Secrets
- [ ] RBAC e Segurança
- [ ] Troubleshooting
- [ ] Simulados práticos

## 🎓 Certificação

Esta estrutura foi criada para facilitar o estudo para a certificação **CKA - Certified Kubernetes Administrator** da Cloud Native Computing Foundation (CNCF).

---

**Boa sorte nos estudos! 🚀**