# Componentes do Control Plane

Esta pasta contém guias completos sobre os componentes do Control Plane do Kubernetes, que são responsáveis por gerenciar o cluster.

## 📚 Conteúdo

### [etcd.md](./etcd.md)
**Banco de dados chave-valor distribuído**
- O que é ETCD e suas características
- O que o ETCD armazena no Kubernetes
- ETCDCTL: ferramenta CLI e comandos essenciais
- Backup e restore do ETCD
- Troubleshooting comum
- Boas práticas de segurança e performance

### [kube-apiserver.md](./kube-apiserver.md)
**Ponto central de comunicação do cluster**
- Responsabilidades do API Server
- Fluxo de requisições (autenticação, autorização, validação)
- Usando API REST diretamente
- RBAC (Role-Based Access Control)
- Métodos HTTP e recursos
- Troubleshooting de problemas de autenticação/autorização

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

## 🎯 Importância para o Exame CKA

Os componentes do Control Plane representam **25% da prova** no domínio "Cluster Architecture, Installation & Configuration".

É essencial entender:
- Como cada componente funciona
- Como eles interagem entre si
- Como fazer troubleshooting quando há problemas
- Como fazer backup/restore do ETCD

## 🔗 Ordem de Estudo Sugerida

1. **[etcd.md](./etcd.md)** - Base de dados do cluster, essencial entender primeiro
2. **[kube-apiserver.md](./kube-apiserver.md)** - Ponto central, todos se comunicam com ele
3. **[kube-controller-manager.md](./kube-controller-manager.md)** - Gerencia o estado do cluster
4. **[kube-scheduler.md](./kube-scheduler.md)** - Decide onde os pods rodam

## 💡 Dica de Prova

Na prova, você pode precisar:
- Fazer backup e restore do ETCD
- Configurar autenticação e RBAC via API Server
- Entender por que um Deployment não está criando pods (Controller Manager)
- Debugar por que um pod está em Pending (Scheduler)

---

⬅️ **Anterior**: [Conceitos-Fundamentais](../Conceitos-Fundamentais/) | ➡️ **Próximo**: [Componentes-Worker-Nodes](../Componentes-Worker-Nodes/)
