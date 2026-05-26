# ✅ Escopo do Exame CKA

Este documento lista **o que está DENTRO e FORA** do escopo do exame **CKA (Certified Kubernetes Administrator)**.

## 📋 Informações do Exame

- **Versão Kubernetes**: v1.31 ou superior
- **Duração**: 2 horas
- **Questões**: ~17 questões práticas (hands-on)
- **Formato**: Linha de comando (terminal) + acesso à documentação oficial
- **Passing Score**: 66%

## ✅ O QUE ESTÁ NO ESCOPO

### 1. Troubleshooting (30%)

**Competências oficiais:**
- ✅ Evaluate cluster and node logging
- ✅ Understand how to monitor applications
- ✅ Manage container stdout & stderr logs
- ✅ Troubleshoot application failure
- ✅ Troubleshoot cluster component failure
- ✅ Troubleshoot networking
- ✅ Troubleshoot cluster and node resource usage

**Na prática:**
- ✅ Troubleshooting de aplicações (pods em CrashLoopBackOff, ImagePullBackOff, etc.)
- ✅ Troubleshooting de cluster (nodes NotReady, componentes do control plane down)
- ✅ Troubleshooting de rede (services não acessíveis, DNS resolution)
- ✅ Logs e eventos (`kubectl logs`, `kubectl describe`, `kubectl get events`)
- ✅ Monitoramento de aplicações e recursos (`kubectl top nodes/pods`)

**NÃO inclui:**
- ❌ Prometheus/Grafana setup (apenas monitoramento básico com `kubectl top`)
- ❌ EFK/ELK stack setup
- ❌ Distributed tracing (Jaeger, Zipkin)

---

### 2. Cluster Architecture, Installation & Configuration (25%)

**Competências oficiais:**
- ✅ Manage role-based access control (RBAC)
- ✅ Prepare underlying infrastructure for installing a Kubernetes cluster
- ✅ Create and manage Kubernetes clusters using kubeadm
- ✅ Manage the lifecycle of Kubernetes clusters
- ✅ **Implement and configure a highly-available control plane**
- ✅ **Use Helm and Kustomize to install cluster components**
- ✅ Understand extension interfaces (CNI, CSI, CRI, etc.)
- ✅ **Understand CRDs, install and configure operators**

**Na prática:**
- ✅ ETCD backup e restore
- ✅ Upgrade de cluster com kubeadm
- ✅ Authentication (certificados X.509, ServiceAccounts)
- ✅ RBAC (Roles, RoleBindings, ClusterRoles, ClusterRoleBindings)
- ✅ Admission Controllers (conceitos básicos: LimitRanger, ResourceQuota, PodSecurity)
- ✅ Kubeadm para criar e gerenciar clusters
- ✅ Componentes do control plane e worker nodes
- ✅ **HA control plane (multi-master setup)**
- ✅ **Helm** (instalação e uso para deploy de componentes do cluster)
- ✅ **Kustomize** (instalação e uso para customizar manifests)
- ✅ **CRDs** (Custom Resource Definitions - entender e criar)
- ✅ **Operators** (instalar e configurar, não desenvolver)
- ✅ CNI, CSI, CRI (Container Network Interface, Storage, Runtime)

**NÃO inclui:**
- ❌ Instalação "from scratch" (sem kubeadm) - apenas kubeadm
- ❌ Cluster Federation
- ❌ Custom Admission Webhooks (implementação detalhada de webhooks customizados)
- ❌ OIDC configuration (apenas conceitos básicos)
- ❌ **Desenvolvimento** de Operators (apenas instalação e configuração)

---

### 3. Services & Networking (20%)

**Competências oficiais:**
- ✅ Understand connectivity between Pods
- ✅ Define and enforce Network Policies
- ✅ Use ClusterIP, NodePort, LoadBalancer service types and endpoints
- ✅ **Use the Gateway API to manage Ingress traffic**
- ✅ Know how to use Ingress controllers and Ingress resources
- ✅ Understand and use CoreDNS
- ✅ Choose an appropriate container network interface plugin

**Na prática:**
- ✅ Services: ClusterIP, NodePort, LoadBalancer
- ✅ Endpoints e EndpointSlices
- ✅ **Ingress** (conceitos, criação de recursos Ingress)
- ✅ **Gateway API** (gerenciar tráfego Ingress com Gateway API)
- ✅ Network Policies (criar e aplicar políticas de rede)
- ✅ CoreDNS (entender e troubleshooting básico)
- ✅ CNI plugins (escolher e entender diferenças)
- ✅ Pod networking (como Pods se comunicam)

**NÃO inclui:**
- ❌ Service Mesh (Istio, Linkerd, Consul) - Gateway API é cobrado, Service Mesh não
- ❌ Instalação e configuração detalhada de CNI plugins (apenas escolher apropriado)
- ❌ Calico/Cilium/Weave Net advanced features
- ❌ Network troubleshooting avançado (tcpdump, wireshark)
- ❌ Configuração avançada de Ingress Controllers (apenas uso básico)

---

### 4. Workloads & Scheduling (15%)

**Competências oficiais:**
- ✅ Understand application deployments and how to perform rolling update and rollbacks
- ✅ Use ConfigMaps and Secrets to configure applications
- ✅ **Configure workload autoscaling**
- ✅ Understand the primitives used to create robust, self-healing, application deployments
- ✅ Understand how resource limits can affect Pod scheduling
- ✅ Understand Pod admission and how to configure Pod scheduling

**Na prática:**
- ✅ Pods
- ✅ Deployments
- ✅ ReplicaSets
- ✅ Rolling updates e rollbacks
- ✅ ConfigMaps e Secrets
- ✅ Resource requests e limits
- ✅ **Horizontal Pod Autoscaler (HPA)**
- ✅ **Vertical Pod Autoscaler (VPA)**
- ✅ **DaemonSets** (primitives — kube-proxy, CNI plugins rodam como DaemonSets)
- ✅ **Jobs** (primitives — pode aparecer na prova)
- ⚠️ CronJobs (zona cinza — pode aparecer como primitive)
- ✅ Node selectors
- ✅ Node affinity e anti-affinity
- ✅ Taints e tolerations
- ✅ Pod affinity e anti-affinity
- ✅ Static Pods
- ✅ Manual scheduling (nodeName)
- ✅ Liveness, Readiness, Startup Probes
- ✅ Init Containers
- ✅ Multi-container Pods (sidecar, adapter, ambassador patterns)

**NÃO inclui:**
- ❌ StatefulSets (foco do CKAD, não cobrado no CKA)
- ❌ Cluster Autoscaler (apenas HPA e VPA)
- ❌ Custom Schedulers (desenvolvimento de schedulers customizados)
- ❌ Scheduler Profiles (detalhamento avançado)
- ⚠️ PriorityClasses (zona cinza — conceito básico pode aparecer em "Pod admission and scheduling", mas preempção avançada não é foco)

---

### 5. Storage (10%)

**Competências oficiais:**
- ✅ Implement storage classes and dynamic volume provisioning
- ✅ Configure volume types, access modes and reclaim policies
- ✅ Manage persistent volumes and persistent volume claims

**Na prática:**
- ✅ Volumes: emptyDir, hostPath, configMap, secret
- ✅ PersistentVolumes (PV)
- ✅ PersistentVolumeClaims (PVC)
- ✅ StorageClasses (provisionamento dinâmico)
- ✅ Access Modes (RWO, ROX, RWX, RWOP)
- ✅ Reclaim Policies (Retain, Delete)
- ✅ Volume expansion
- ✅ CSI drivers (conceitos básicos, não implementação)
- ✅ Volume Modes (Filesystem vs Block)
- ✅ Static vs Dynamic Provisioning

**NÃO inclui:**
- ❌ StatefulSets com VolumeClaimTemplates (StatefulSets não é cobrado no CKA)
- ❌ Volume Snapshots (feature avançada)
- ❌ Volume Cloning (feature avançada)
- ❌ Custom CSI driver development (apenas uso de drivers existentes)
- ❌ Raw block volumes (detalhamento avançado)
- ❌ Volume topology awareness (detalhamento avançado)

---

## ❌ O QUE DEFINITIVAMENTE NÃO ESTÁ NO ESCOPO

### Ferramentas e Tecnologias NÃO Cobertas

- ❌ **GitOps** (ArgoCD, Flux) - não é testado
- ❌ **CI/CD pipelines** (Jenkins, GitLab CI, GitHub Actions) - não é testado
- ❌ **Service Mesh** (Istio, Linkerd, Consul) - não é testado
- ❌ **Prometheus/Grafana** (instalação e configuração) - apenas monitoramento básico
- ❌ **EFK/ELK stack** - não é testado
- ❌ **Kubernetes Dashboard** - não é testado
- ❌ **Lens** ou outras UIs - não é testado

### Workloads NÃO Cobertos

- ❌ **StatefulSets** (foco do CKAD, não cobrado no CKA)

### Features Avançadas NÃO Cobertas

- ❌ **Cluster Autoscaler** (apenas HPA/VPA são cobrados)
- ❌ **Advanced Scheduling**:
  - Custom Schedulers (desenvolvimento)
  - Scheduler Profiles (detalhamento)
  - Pod Overhead
  - Pod Topology Spread Constraints (apenas conceitos básicos)
- ❌ **Advanced Security**:
  - Pod Security Policies (PSP) - deprecated
  - OPA Gatekeeper (implementação detalhada)
  - Falco runtime security
  - Image scanning tools (Trivy, Clair)
- ❌ **Advanced Networking**:
  - NetworkPolicy Providers advanced features (Calico policies, Cilium policies)
  - Multi-cluster networking
  - Service mesh traffic management
- ❌ **Advanced Storage**:
  - Volume Snapshots
  - Volume Cloning
  - CSI driver development (apenas uso)
- ❌ **Multi-tenancy**:
  - Hierarchical Namespaces
  - Virtual Clusters (vcluster)

### Cloud Provider Específico

- ❌ **AWS EKS** específico (managed node groups, Fargate, etc.)
- ❌ **Azure AKS** específico (AKS networking, Azure CNI)
- ❌ **Google GKE** específico (GKE autopilot, Workload Identity)
- ❌ **Cloud-specific load balancers** (detalhamento)

### Desenvolvimento de Aplicações

- ❌ **Dockerfile** creation
- ❌ **Container image building**
- ❌ **Multi-stage builds**
- ❌ **Application development** (isso é CKAD, não CKA)

---

## 🎯 Foco Principal do CKA

O exame CKA foca em **administração de clusters Kubernetes**:

### Você SERÁ testado em:

1. **Instalar e configurar** clusters com kubeadm
2. **Implementar HA control plane** (multi-master)
3. **Fazer backup e restore** do ETCD
4. **Atualizar** clusters para novas versões (cluster lifecycle)
5. **Troubleshooting** de problemas comuns (30% da prova!)
6. **Gerenciar** autenticação e autorização (RBAC)
7. **Configurar** networking básico, network policies, e Gateway API
8. **Gerenciar** storage (PV, PVC, StorageClasses, dynamic provisioning)
9. **Escalar e atualizar** aplicações (Deployments, rolling updates, rollbacks)
10. **Configurar autoscaling** (HPA, VPA)
11. **Configurar** scheduling (affinity, taints/tolerations, resource limits)
12. **Usar Helm e Kustomize** para instalar componentes do cluster
13. **Entender CRDs e instalar/configurar Operators**
14. **Monitorar** recursos básicos (kubectl top)
15. **Usar Gateway API** para gerenciar tráfego Ingress

### Você NÃO será testado em:

1. **Desenvolver** aplicações Kubernetes-native (isso é CKAD)
2. **Configurar** CI/CD pipelines (GitOps, ArgoCD, Flux)
3. **Implementar** service mesh (Istio, Linkerd)
4. **Desenvolver** Operators ou CRDs (apenas instalar e configurar)
5. **Configurar** Cluster Autoscaler (apenas HPA/VPA)
6. **Desenvolver** CSI drivers customizados (apenas usar)
7. **StatefulSets** (foco do CKAD, não cobrado no CKA)

---

## 📖 Como Estudar de Forma Eficiente

### ✅ Priorize (por peso na prova):

#### Troubleshooting (30%)
1. **kubectl logs, describe, get events** (ESSENCIAL!)
2. **kubectl top** nodes/pods
3. Troubleshooting de pods (CrashLoopBackOff, ImagePullBackOff)
4. Troubleshooting de networking (services, DNS)
5. Troubleshooting de cluster (nodes NotReady, control plane components)

#### Cluster Architecture (25%)
1. **ETCD backup/restore** (MUITO COMUM na prova!)
2. **Cluster upgrade** com kubeadm (MUITO COMUM!)
3. **RBAC** (criar Roles, RoleBindings, ClusterRoles, ClusterRoleBindings)
4. **HA control plane** (multi-master setup)
5. **Helm** (instalar charts, usar Helm para componentes)
6. **Kustomize** (usar para customizar manifests)
7. **CRDs e Operators** (instalar e configurar, não desenvolver)

#### Services & Networking (20%)
1. **Services** (ClusterIP, NodePort, LoadBalancer)
2. **Network Policies** (criar e aplicar)
3. **Gateway API** (gerenciar Ingress traffic)
4. **Ingress** (criar recursos Ingress)
5. **CoreDNS** (troubleshooting básico)

#### Workloads & Scheduling (15%)
1. **Deployments** (rollout, rollback, scaling)
2. **ConfigMaps e Secrets**
3. **HPA** (Horizontal Pod Autoscaler) - AGORA É COBRADO!
4. **VPA** (Vertical Pod Autoscaler) - AGORA É COBRADO!
5. **Resource requests e limits**
6. **Scheduling** (affinity, taints/tolerations)

#### Storage (10%)
1. **PV/PVC** (criar e usar)
2. **StorageClasses** (dynamic provisioning)
3. **Access Modes** (RWO, ROX, RWX)
4. **Reclaim Policies** (Retain, Delete)

### ⚠️ Não gaste tempo em:

- ❌ StatefulSets (foco do CKAD, NÃO está no escopo do CKA)
- ❌ Cluster Autoscaler (apenas HPA/VPA são cobrados)
- ❌ Service Mesh (NÃO está no escopo)
- ❌ Prometheus/Grafana setup (apenas `kubectl top`)
- ❌ GitOps tools (ArgoCD, Flux)
- ❌ Desenvolver Operators (apenas instalar/configurar)

---

## 🔗 Recursos Oficiais

- [CKA Curriculum (oficial)](https://github.com/cncf/curriculum)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Killer.sh](https://killer.sh/) - Simulados oficiais

---

## 📝 Nota Importante

Este documento é baseado no **CKA Curriculum atual (v1.31+)**. O currículo pode mudar entre versões. Sempre consulte o [curriculum oficial](https://github.com/cncf/curriculum) para a versão mais atualizada.

### 🆕 Mudanças Recentes no Currículo

Comparado com versões anteriores, o CKA **AGORA INCLUI**:
- ✅ **Workload autoscaling** (HPA, VPA)
- ✅ **Helm e Kustomize**
- ✅ **CRDs e Operators** (instalar e configurar)
- ✅ **Gateway API**
- ✅ **HA control plane**

E **REMOVEU** do escopo explícito (mas "primitives" ainda os cobre implicitamente):
- ❌ StatefulSets (foco do CKAD)

---

**Boa sorte nos estudos! Foque no que importa para o exame! 🚀**
