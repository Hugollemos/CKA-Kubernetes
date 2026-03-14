# Componentes do Control Plane

Esta pasta contém guias completos sobre os componentes do Control Plane do Kubernetes, que são responsáveis por gerenciar o cluster.

## 📚 Conteúdo

### [etcd.md](./etcd.md)
**Banco de dados chave-valor distribuído**
- O que é ETCD e suas características
- O que o ETCD armazena no Kubernetes
- ETCDCTL: ferramenta CLI e comandos essenciais
- Backup e restore básico do ETCD
- Troubleshooting comum
- Boas práticas de segurança e performance

### [backup-restore.md](./backup-restore.md)
**Métodos completos de backup e restore do cluster**
- Backup do ETCD (snapshot save/restore)
- Backup declarativo (resource definitions YAML)
- Backup de certificados e configurações
- Velero (ferramenta de backup enterprise)
- Scripts de automação de backup
- Disaster recovery scenarios
- Comparação de métodos e quando usar cada um
- Troubleshooting de backup e restore

### [kube-apiserver.md](./kube-apiserver.md)
**Ponto central de comunicação do cluster**
- Responsabilidades do API Server
- Fluxo de requisições (autenticação, autorização, validação)
- Usando API REST diretamente
- RBAC (Role-Based Access Control)
- Métodos HTTP e recursos
- Troubleshooting de problemas de autenticação/autorização

### [authentication.md](./authentication.md)
**Autenticação de usuários e aplicações no cluster**
- O que é Authentication vs Authorization vs Admission Control
- Tipos de usuários (Service Accounts vs Normal Users)
- Métodos de autenticação: X.509 Certificates, Bearer Tokens, OIDC, Webhook
- Criar usuários com certificados X.509 e CertificateSigningRequest
- Service Accounts: criação, tokens e uso em pods
- Static tokens e passwords (não recomendado)
- Configuração de OIDC para empresas
- Troubleshooting de problemas de autenticação (401 Unauthorized)
- Comandos essenciais: kubectl auth whoami, can-i, impersonate

### [kube-controller-manager.md](./kube-controller-manager.md)
**Orquestrador dos controladores**
- O que é o Controller Manager
- Principais controllers: Deployment, ReplicaSet, Node, Job, CronJob, etc.
- Reconciliation Loop (loop de reconciliação)
- Estado desejado vs estado atual
- Exemplos práticos de atuação dos controllers
- Monitoramento e troubleshooting

### [kube-scheduler.md](./kube-scheduler.md)
**Agendador de pods nos nós**
- Processo de scheduling (Filter e Scoring)
- Critérios de filtragem de nós
- Restrições e preferências (nodeSelector, affinity, taints/tolerations)
- Pod Topology Spread
- Priority Classes
- Troubleshooting de pods em estado Pending

### [admission-controllers.md](./admission-controllers.md)
**Plugins que interceptam requisições ao API Server**
- O que são Admission Controllers (mutating e validating)
- Fluxo de requisições no API Server
- Principais admission controllers (LimitRanger, ResourceQuota, ServiceAccount, etc.)
- Pod Security Admission (níveis: privileged, baseline, restricted)
- Admission Webhooks customizados
- Configurar e habilitar/desabilitar admission controllers
- Troubleshooting de requisições rejeitadas

### [monitoring.md](./monitoring.md)
**Monitoramento e observabilidade do cluster**
- Metrics Server (kubectl top nodes/pods)
- Logs de componentes (kubectl logs, journalctl)
- Events do Kubernetes (kubectl get events)
- Health checks (API Server, etcd, kubelet)
- Node conditions e recursos alocáveis
- Troubleshooting com logs e métricas
- Ferramentas avançadas (Prometheus, Grafana, EFK)

### [cluster-upgrade.md](./cluster-upgrade.md)
**Atualização de versão do cluster Kubernetes**
- Versionamento do Kubernetes (Major.Minor.Patch)
- Estratégia e ordem de upgrade (control plane → workers)
- Regras de compatibilidade entre componentes
- Processo completo com kubeadm (upgrade apply/node)
- Backup do etcd antes do upgrade
- Drain e uncordon de nós durante upgrade
- Troubleshooting de problemas no upgrade
- Checklist e boas práticas

## 🎯 Importância para o Exame CKA

Os componentes do Control Plane representam **25% da prova** no domínio "Cluster Architecture, Installation & Configuration".

É essencial entender:
- Como cada componente funciona
- Como eles interagem entre si
- Como fazer troubleshooting quando há problemas
- Como fazer backup/restore do ETCD

## 🔗 Ordem de Estudo Sugerida

1. **[etcd.md](./etcd.md)** - Base de dados do cluster, essencial entender primeiro
2. **[backup-restore.md](./backup-restore.md)** - Backup e restore do cluster (MUITO IMPORTANTE!)
3. **[kube-apiserver.md](./kube-apiserver.md)** - Ponto central, todos se comunicam com ele
4. **[authentication.md](./authentication.md)** - Autenticação de usuários e aplicações (primeira camada de segurança)
5. **[admission-controllers.md](./admission-controllers.md)** - Interceptam requisições no API Server (terceira camada)
6. **[kube-controller-manager.md](./kube-controller-manager.md)** - Gerencia o estado do cluster
7. **[kube-scheduler.md](./kube-scheduler.md)** - Decide onde os pods rodam
8. **[monitoring.md](./monitoring.md)** - Monitoramento e troubleshooting do cluster
9. **[cluster-upgrade.md](./cluster-upgrade.md)** - Atualização de versão do cluster (IMPORTANTE!)

## 💡 Dica de Prova

Na prova, você pode precisar:
- Fazer backup e restore do ETCD (MUITO COMUM!)
- Atualizar o cluster para uma nova versão (MUITO COMUM!)
- Criar usuários com certificados X.509 (COMUM!)
- Criar ServiceAccounts e configurar em pods (COMUM!)
- Configurar autenticação e RBAC via API Server
- Testar permissões com kubectl auth can-i
- Configurar Pod Security Admission em namespaces (Admission Controllers)
- Criar LimitRange e ResourceQuota (Admission Controllers)
- Entender por que um Deployment não está criando pods (Controller Manager)
- Debugar por que um pod está em Pending (Scheduler)
- Investigar pods em CrashLoopBackOff usando logs e events (Monitoring)
- Verificar uso de recursos com kubectl top (Monitoring)

---

⬅️ **Anterior**: [Conceitos-Fundamentais](../Conceitos-Fundamentais/) | ➡️ **Próximo**: [Componentes-Worker-Nodes](../Componentes-Worker-Nodes/)
