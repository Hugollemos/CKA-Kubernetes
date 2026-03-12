# Cluster Upgrade (Atualização de Cluster Kubernetes)

## 📋 O que é Cluster Upgrade?

**Cluster Upgrade** é o processo de atualizar os componentes do Kubernetes (control plane e worker nodes) para uma nova versão, garantindo **compatibilidade**, **segurança** e acesso a **novos recursos**.

## 🔄 Versioning do Kubernetes

### Formato de Versão: `vX.Y.Z`

```
v1.28.3
│ │  │
│ │  └─ Patch version (correção de bugs)
│ └──── Minor version (novas features)
└───── Major version (breaking changes)
```

**Exemplo:**
- **v1.28.0** → **v1.28.3**: Patch (correção de bugs) ✅
- **v1.28.0** → **v1.29.0**: Minor (nova versão com features) ✅
- **v1.28.0** → **v2.0.0**: Major (breaking changes) ⚠️

### Release Cycle

- **Nova minor version**: ~3 vezes por ano (a cada 4 meses)
- **Suporte**: Últimas 3 minor versions
- **Patch releases**: Regularmente para correções de segurança

```
┌──────────────────────────────────────────────────┐
│ Suporte Oficial (3 versões)                     │
├──────────────────────────────────────────────────┤
│ v1.30.x ← Atual                                  │
│ v1.29.x ← Suportada                              │
│ v1.28.x ← Suportada                              │
│ v1.27.x ← Sem suporte (deprecated)               │
└──────────────────────────────────────────────────┘
```

## 🎯 Estratégia de Upgrade

### Regras de Compatibilidade

**1. Control Plane components:**
- **kube-apiserver**: Versão de referência (não pode ser mais antiga)
- **kube-controller-manager**: Max 1 versão atrás do apiserver
- **kube-scheduler**: Max 1 versão atrás do apiserver
- **etcd**, **cloud-controller-manager**: Versões compatíveis documentadas

**2. Worker Nodes:**
- **kubelet**: Max 2 versões atrás do apiserver
- **kube-proxy**: Mesma versão do kubelet

**Exemplo de compatibilidade:**
```
kube-apiserver:           v1.30
kube-controller-manager:  v1.30 ou v1.29
kube-scheduler:           v1.30 ou v1.29
kubelet:                  v1.30, v1.29 ou v1.28
kube-proxy:               v1.30, v1.29 ou v1.28
```

### Ordem do Upgrade

```
1. ETCD (backup primeiro!) ←────────┐
   ↓                                │ Control
2. kube-apiserver                   │ Plane
   ↓                                │
3. kube-controller-manager          │
   ↓                                │
4. kube-scheduler                   │
   ↓                                │
5. cloud-controller-manager ────────┘
   ↓
6. Worker Nodes (um de cada vez) ←── Workers
   - kubelet
   - kube-proxy
```

**IMPORTANTE:**
- ✅ Atualizar **um minor version por vez** (v1.28 → v1.29 → v1.30)
- ❌ Não pular versões (v1.28 → v1.30 diretamente)
- ✅ Control plane **antes** dos worker nodes
- ✅ Fazer **backup do etcd** antes de qualquer upgrade

## 🛠️ Upgrade com kubeadm

### Pré-requisitos

```bash
# 1. Verificar versão atual
kubectl version --short
kubectl get nodes

# 2. Ver versões disponíveis
apt update
apt-cache madison kubeadm

# Output:
#   kubeadm | 1.30.0-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
#   kubeadm | 1.29.4-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
#   kubeadm | 1.29.3-00 | https://apt.kubernetes.io kubernetes-xenial/main amd64 Packages
```

### Processo de Upgrade

#### FASE 1: Upgrade do Control Plane (Master Node)

**Step 1: Atualizar kubeadm no master**

```bash
# 1. Fazer SSH no nó control plane
ssh user@control-plane-node

# 2. Atualizar kubeadm para versão desejada (exemplo: v1.29.0)
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# 3. Verificar versão instalada
kubeadm version
```

**Step 2: Verificar plano de upgrade**

```bash
# Ver o que será atualizado
kubeadm upgrade plan

# Output mostra:
# Components that must be upgraded manually after you have upgraded the control plane with 'kubeadm upgrade apply':
# COMPONENT   CURRENT       TARGET
# kubelet     3 x v1.28.0   v1.29.0
#
# Upgrade to the latest stable version:
#
# COMPONENT                 CURRENT   TARGET
# kube-apiserver            v1.28.0   v1.29.0
# kube-controller-manager   v1.28.0   v1.29.0
# kube-scheduler            v1.28.0   v1.29.0
# kube-proxy                v1.28.0   v1.29.0
# CoreDNS                   v1.10.1   v1.11.1
# etcd                      3.5.9     3.5.10
```

**Step 3: Fazer backup do etcd (CRUCIAL!)**

```bash
# Backup do etcd antes do upgrade
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot-pre-upgrade.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Verificar backup
ETCDCTL_API=3 etcdctl snapshot status /backup/etcd-snapshot-pre-upgrade.db
```

**Step 4: Drenar o nó control plane**

```bash
# Marcar nó como unschedulable e evict pods
kubectl drain control-plane-node --ignore-daemonsets

# Output:
# node/control-plane-node cordoned
# evicting pod default/nginx-abc123
# pod/nginx-abc123 evicted
# node/control-plane-node drained
```

**Step 5: Aplicar o upgrade**

```bash
# Executar upgrade do control plane
kubeadm upgrade apply v1.29.0

# Confirmar quando solicitado (y)
# O processo pode levar alguns minutos

# Output final:
# [upgrade/successful] SUCCESS! Your cluster was upgraded to "v1.29.0". Enjoy!
```

**Step 6: Atualizar kubelet e kubectl no control plane**

```bash
# 1. Atualizar kubelet e kubectl
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl

# 2. Reiniciar kubelet
systemctl daemon-reload
systemctl restart kubelet

# 3. Verificar status
systemctl status kubelet
```

**Step 7: Uncordon o nó control plane**

```bash
# Marcar nó como schedulable novamente
kubectl uncordon control-plane-node

# Verificar nós
kubectl get nodes

# Output:
# NAME                 STATUS   ROLES           AGE   VERSION
# control-plane-node   Ready    control-plane   10d   v1.29.0  ← Atualizado!
# worker-node-1        Ready    <none>          10d   v1.28.0
# worker-node-2        Ready    <none>          10d   v1.28.0
```

#### FASE 2: Upgrade dos Worker Nodes

**Upgrade um worker node por vez para evitar downtime!**

**Step 1: Atualizar kubeadm no worker**

```bash
# 1. SSH no worker node
ssh user@worker-node-1

# 2. Atualizar kubeadm
apt-mark unhold kubeadm
apt-get update
apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm
```

**Step 2: Drenar o worker node (do control plane)**

```bash
# No control plane, drenar o worker
kubectl drain worker-node-1 --ignore-daemonsets --delete-emptydir-data

# Flags importantes:
# --ignore-daemonsets: Ignora DaemonSet pods (não podem ser evicted)
# --delete-emptydir-data: Deleta pods com emptyDir volumes
```

**Step 3: Atualizar configuração do kubelet (no worker)**

```bash
# No worker node, atualizar config
kubeadm upgrade node

# Output:
# [upgrade] Reading configuration from the cluster...
# [upgrade] Upgrading your Static Pod-hosted control plane instance to version "v1.29.0"
# [upgrade] Successfully upgraded your Static Pod-hosted control plane instance to version "v1.29.0"
```

**Step 4: Atualizar kubelet e kubectl (no worker)**

```bash
# 1. Atualizar pacotes
apt-mark unhold kubelet kubectl
apt-get update
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl

# 2. Reiniciar kubelet
systemctl daemon-reload
systemctl restart kubelet

# 3. Verificar status
systemctl status kubelet
```

**Step 5: Uncordon o worker node (do control plane)**

```bash
# No control plane, marcar worker como schedulable
kubectl uncordon worker-node-1

# Verificar
kubectl get nodes

# Output:
# NAME                 STATUS   ROLES           AGE   VERSION
# control-plane-node   Ready    control-plane   10d   v1.29.0
# worker-node-1        Ready    <none>          10d   v1.29.0  ← Atualizado!
# worker-node-2        Ready    <none>          10d   v1.28.0
```

**Step 6: Repetir para os demais workers**

```bash
# Repetir Steps 1-5 para worker-node-2, worker-node-3, etc.
# UM DE CADA VEZ para evitar downtime!
```

## 🔍 Verificação Pós-Upgrade

```bash
# 1. Verificar versões de todos os nós
kubectl get nodes -o wide

# 2. Verificar componentes do control plane
kubectl get pods -n kube-system

# 3. Verificar versões dos componentes
kubectl version --short

# 4. Verificar health do cluster
kubectl get componentstatuses  # Deprecated mas útil
kubectl get --raw='/readyz?verbose'

# 5. Testar criando um pod
kubectl run test-nginx --image=nginx
kubectl get pods
kubectl delete pod test-nginx

# 6. Verificar logs dos componentes (se houver problemas)
kubectl logs -n kube-system kube-apiserver-control-plane-node
kubectl logs -n kube-system kube-controller-manager-control-plane-node
kubectl logs -n kube-system kube-scheduler-control-plane-node
```

## 📊 Fluxo Completo de Upgrade

```
┌─────────────────────────────────────────────────┐
│ 1. PRÉ-UPGRADE                                  │
├─────────────────────────────────────────────────┤
│ ✅ Ler release notes da nova versão            │
│ ✅ Verificar compatibilidade (APIs deprecated)  │
│ ✅ Backup do ETCD                               │
│ ✅ Backup dos manifestos (/etc/kubernetes)      │
│ ✅ Testar em ambiente não-produção              │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ 2. UPGRADE CONTROL PLANE                        │
├─────────────────────────────────────────────────┤
│ 1. Atualizar kubeadm                            │
│ 2. kubeadm upgrade plan                         │
│ 3. kubectl drain control-plane                  │
│ 4. kubeadm upgrade apply v1.X.Y                 │
│ 5. Atualizar kubelet e kubectl                  │
│ 6. Reiniciar kubelet                            │
│ 7. kubectl uncordon control-plane               │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ 3. UPGRADE WORKERS (um por vez)                 │
├─────────────────────────────────────────────────┤
│ 1. Atualizar kubeadm                            │
│ 2. kubectl drain worker-node-X                  │
│ 3. kubeadm upgrade node                         │
│ 4. Atualizar kubelet e kubectl                  │
│ 5. Reiniciar kubelet                            │
│ 6. kubectl uncordon worker-node-X               │
│ 7. Repetir para próximo worker                  │
└─────────────────────────────────────────────────┘
                    ↓
┌─────────────────────────────────────────────────┐
│ 4. PÓS-UPGRADE                                  │
├─────────────────────────────────────────────────┤
│ ✅ Verificar kubectl get nodes                  │
│ ✅ Verificar pods do kube-system                │
│ ✅ Testar criação de recursos                   │
│ ✅ Monitorar logs por 24-48h                    │
└─────────────────────────────────────────────────┘
```

## 🚨 Troubleshooting

### Upgrade falhou no kubeadm upgrade apply

```bash
# 1. Ver logs detalhados
kubeadm upgrade apply v1.29.0 --v=5

# 2. Verificar se todos os componentes estão healthy
kubectl get pods -n kube-system

# 3. Verificar certificados
kubeadm certs check-expiration

# 4. Se necessário, reverter usando backup do etcd
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot-pre-upgrade.db
```

### Kubelet não inicia após upgrade

```bash
# 1. Ver logs do kubelet
journalctl -u kubelet -f

# 2. Ver status
systemctl status kubelet

# 3. Problemas comuns:
# - Config antiga: rm /var/lib/kubelet/config.yaml && kubeadm upgrade node
# - Certificado expirado: kubeadm certs renew all
# - Porta em uso: lsof -i :10250
```

### Nó não fica Ready após uncordon

```bash
# 1. Descrever o nó
kubectl describe node worker-node-1

# 2. Ver conditions
kubectl get node worker-node-1 -o jsonpath='{.status.conditions}'

# 3. Verificar CNI plugin
kubectl get pods -n kube-system | grep -E 'calico|flannel|weave'

# 4. Reiniciar CNI se necessário
kubectl delete pod -n kube-system -l k8s-app=calico-node
```

### Pods não agendam após upgrade

```bash
# 1. Ver eventos
kubectl get events --sort-by='.lastTimestamp'

# 2. Ver scheduler logs
kubectl logs -n kube-system kube-scheduler-control-plane-node

# 3. Verificar taints
kubectl describe node worker-node-1 | grep Taints

# 4. Remover taint se necessário
kubectl taint nodes worker-node-1 node.kubernetes.io/unschedulable:NoSchedule-
```

## 📝 Exemplo Completo: v1.28.0 → v1.29.0

### Control Plane Node

```bash
# === PRÉ-UPGRADE ===
# Backup do etcd
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# === UPGRADE CONTROL PLANE ===
# 1. Atualizar kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# 2. Ver plano
kubeadm upgrade plan

# 3. Drenar nó
kubectl drain control-plane-node --ignore-daemonsets

# 4. Aplicar upgrade
kubeadm upgrade apply v1.29.0

# 5. Atualizar kubelet e kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# 6. Uncordon
kubectl uncordon control-plane-node

# 7. Verificar
kubectl get nodes
```

### Worker Nodes (repetir para cada worker)

```bash
# === WORKER NODE 1 ===
# No control plane: drenar worker
kubectl drain worker-node-1 --ignore-daemonsets --delete-emptydir-data

# No worker node:
# 1. Atualizar kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# 2. Upgrade node config
kubeadm upgrade node

# 3. Atualizar kubelet e kubectl
apt-mark unhold kubelet kubectl
apt-get update && apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# No control plane: uncordon worker
kubectl uncordon worker-node-1

# Verificar
kubectl get nodes
```

## ✅ Checklist de Upgrade

**Pré-Upgrade:**
- [ ] Ler release notes da versão target
- [ ] Verificar APIs deprecated/removed
- [ ] Backup do etcd
- [ ] Backup de /etc/kubernetes
- [ ] Testar upgrade em ambiente de teste
- [ ] Planejar janela de manutenção

**Durante Upgrade:**
- [ ] Atualizar control plane primeiro
- [ ] Verificar que control plane está healthy
- [ ] Atualizar workers um por vez
- [ ] Aguardar cada worker ficar Ready antes do próximo
- [ ] Não pular minor versions

**Pós-Upgrade:**
- [ ] Verificar versões: `kubectl get nodes`
- [ ] Verificar pods do kube-system
- [ ] Testar criação de recursos
- [ ] Verificar aplicações funcionando
- [ ] Monitorar logs por 24-48h
- [ ] Atualizar documentação

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Ordem de upgrade**
   - Control plane antes de workers
   - Um minor version por vez

2. **Comandos essenciais**
   ```bash
   # Ver plano
   kubeadm upgrade plan

   # Aplicar upgrade (control plane)
   kubeadm upgrade apply v1.X.Y

   # Upgrade node config (workers)
   kubeadm upgrade node

   # Drenar/Uncordon nós
   kubectl drain <node> --ignore-daemonsets
   kubectl uncordon <node>
   ```

3. **Backup do etcd ANTES de upgrade**
   ```bash
   ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key
   ```

4. **Versão do kubeadm deve ser atualizada PRIMEIRO**

5. **Flags importantes do drain**
   - `--ignore-daemonsets`: Ignora DaemonSets
   - `--delete-emptydir-data`: Deleta pods com emptyDir

### 🧪 Cenário típico na prova:

> **"Atualize o cluster da versão v1.28.0 para v1.29.0. O cluster foi instalado com kubeadm."**

```bash
# CONTROL PLANE
ssh control-plane-node

# 1. Backup etcd
ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 2. Upgrade kubeadm
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm

# 3. Drenar
kubectl drain control-plane-node --ignore-daemonsets

# 4. Aplicar upgrade
kubeadm upgrade apply v1.29.0

# 5. Upgrade kubelet/kubectl
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet

# 6. Uncordon
kubectl uncordon control-plane-node

# WORKERS (repetir para cada)
kubectl drain worker-node-1 --ignore-daemonsets

ssh worker-node-1
apt-mark unhold kubeadm
apt-get update && apt-get install -y kubeadm=1.29.0-00
apt-mark hold kubeadm
kubeadm upgrade node
apt-mark unhold kubelet kubectl
apt-get install -y kubelet=1.29.0-00 kubectl=1.29.0-00
apt-mark hold kubelet kubectl
systemctl daemon-reload
systemctl restart kubelet
exit

kubectl uncordon worker-node-1

# Verificar
kubectl get nodes
```

## 💡 Dicas para a Prova

1. **Sempre faça backup do etcd primeiro**
   - Pode ser solicitado explicitamente

2. **Não pule versões**
   - v1.28 → v1.29 → v1.30 (correto)
   - v1.28 → v1.30 (errado)

3. **Use apt-mark hold/unhold**
   - Evita upgrades acidentais

4. **Drain antes, uncordon depois**
   - SEMPRE nessa ordem

5. **Um worker por vez**
   - Evita downtime completo

6. **Verifique com kubectl get nodes**
   - Confirme versões após upgrade

---

⬅️ **Anterior**: [monitoring.md](./monitoring.md) | ➡️ **Próximo**: [Componentes-Worker-Nodes](../Componentes-Worker-Nodes/)
