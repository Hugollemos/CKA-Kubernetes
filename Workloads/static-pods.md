# Static Pods

## 📋 O que são Static Pods?

**Static Pods** são pods gerenciados **diretamente pelo kubelet** em um nó específico, sem a necessidade do API Server do Kubernetes.

### Características principais:
- ✅ Criados e gerenciados pelo **kubelet** (não pelo API Server)
- ✅ Definidos por arquivos YAML em um **diretório local** do nó
- ✅ Kubelet monitora o diretório e cria/atualiza/deleta pods automaticamente
- ✅ Aparecem no API Server como **mirror pods** (somente leitura)
- ✅ **Não podem** ser gerenciados por kubectl (delete, edit, etc.)
- ✅ Usados principalmente para **componentes do Control Plane**

## 🎯 Diferenças: Static Pods vs Pods Normais

| Característica | Static Pods | Pods Normais |
|----------------|-------------|--------------|
| **Gerenciado por** | Kubelet | API Server |
| **Localização da definição** | Arquivo local no nó | etcd (via API Server) |
| **Pode ser deletado via kubectl** | ❌ Não | ✅ Sim |
| **Automaticamente reiniciado** | ✅ Sim (pelo kubelet) | ✅ Sim (pelo kubelet) |
| **Visível no kubectl get pods** | ✅ Sim (mirror pod) | ✅ Sim |
| **Pode ser agendado em outros nós** | ❌ Não (fixo no nó) | ✅ Sim |
| **Uso típico** | Control Plane components | Aplicações |

## 🏗️ Como Funcionam os Static Pods?

```
┌─────────────────────────────────────────┐
│  Nó (Node)                              │
│                                         │
│  ┌──────────────────────────────────┐  │
│  │ /etc/kubernetes/manifests/       │  │
│  │                                  │  │
│  │  ├── kube-apiserver.yaml        │  │
│  │  ├── etcd.yaml                  │  │
│  │  ├── kube-controller-manager.yaml│ │
│  │  └── kube-scheduler.yaml        │  │
│  └────────────┬─────────────────────┘  │
│               │                         │
│               ↓                         │
│  ┌──────────────────────────────────┐  │
│  │      Kubelet                     │  │
│  │  - Monitora o diretório          │  │
│  │  - Cria/atualiza pods            │  │
│  │  - Reinicia se falhar            │  │
│  └────────────┬─────────────────────┘  │
│               │                         │
│               ↓                         │
│  ┌──────────────────────────────────┐  │
│  │  Pods rodando no nó              │  │
│  │  - kube-apiserver                │  │
│  │  - etcd                          │  │
│  │  - kube-controller-manager       │  │
│  │  - kube-scheduler                │  │
│  └──────────────────────────────────┘  │
└─────────────────────────────────────────┘

         │ (cria mirror pod)
         ↓

┌─────────────────────────────────────────┐
│  API Server / etcd                      │
│  - Mirror pods (somente leitura)        │
│  - Visível via kubectl get pods         │
└─────────────────────────────────────────┘
```

## 📂 Onde os Static Pods são Definidos?

### 1. Descobrir o diretório de Static Pods

O caminho padrão é configurado no arquivo de configuração do kubelet.

```bash
# Verificar a configuração do kubelet
ps aux | grep kubelet | grep config

# OU ver diretamente o arquivo de config
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Caminho padrão comum:
# staticPodPath: /etc/kubernetes/manifests
```

### 2. Caminhos comuns por distribuição

| Distribuição | Caminho padrão |
|--------------|----------------|
| **kubeadm** | `/etc/kubernetes/manifests` |
| **minikube** | `/etc/kubernetes/manifests` |
| **kubespray** | `/etc/kubernetes/manifests` |
| **manual** | Definido no `--pod-manifest-path` ou config file |

## 🚀 Criando um Static Pod

### Método 1: Criar arquivo YAML diretamente no nó

```bash
# 1. SSH no nó onde quer criar o static pod
ssh user@node01

# 2. Descobrir o diretório de static pods
sudo cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Resultado: staticPodPath: /etc/kubernetes/manifests

# 3. Criar arquivo YAML no diretório
sudo vi /etc/kubernetes/manifests/my-static-pod.yaml
```

**Exemplo de Static Pod:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-static-pod
  labels:
    app: static-app
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
```

```bash
# 4. Salvar o arquivo
# O kubelet detecta automaticamente e cria o pod!

# 5. Verificar se o pod foi criado
kubectl get pods -o wide
# Você verá: my-static-pod-node01 (nome + sufixo do nó)
```

### Método 2: Gerar YAML com kubectl dry-run

```bash
# 1. Gerar YAML
kubectl run static-nginx --image=nginx --dry-run=client -o yaml > static-nginx.yaml

# 2. Copiar para o nó
scp static-nginx.yaml user@node01:/tmp/

# 3. SSH no nó
ssh user@node01

# 4. Mover para o diretório de manifests
sudo mv /tmp/static-nginx.yaml /etc/kubernetes/manifests/

# 5. Verificar
kubectl get pods -o wide
```

## 🔍 Identificando Static Pods

### 1. Via kubectl
```bash
# Listar todos os pods
kubectl get pods -A -o wide

# Static pods têm o nome do nó como sufixo
# Exemplo: kube-apiserver-controlplane
#          etcd-controlplane
#          my-static-pod-node01
```

### 2. Verificar o ownerReferences
```bash
kubectl get pod <pod-name> -o yaml | grep ownerReferences -A 5

# Static pods têm ownerReferences do tipo "Node"
# Pods normais têm ownerReferences do tipo "ReplicaSet", "Deployment", etc.
```

**Exemplo de Static Pod:**
```yaml
ownerReferences:
- apiVersion: v1
  kind: Node              # ← Tipo "Node" indica Static Pod
  name: node01
```

**Exemplo de Pod Normal:**
```yaml
ownerReferences:
- apiVersion: apps/v1
  kind: ReplicaSet        # ← Tipo "ReplicaSet" indica pod normal
  name: nginx-7854ff8877
```

### 3. Listar arquivos no diretório de manifests
```bash
# SSH no nó
ssh user@node01

# Listar static pods definidos
sudo ls -la /etc/kubernetes/manifests/
```

## 🛠️ Gerenciando Static Pods

### Criar Static Pod
```bash
# 1. Criar arquivo YAML no diretório de manifests
sudo vi /etc/kubernetes/manifests/my-pod.yaml

# 2. Kubelet detecta e cria automaticamente
kubectl get pods
```

### Editar Static Pod
```bash
# 1. SSH no nó
ssh user@node01

# 2. Editar o arquivo YAML
sudo vi /etc/kubernetes/manifests/my-pod.yaml

# 3. Salvar - kubelet detecta e atualiza automaticamente
# O pod será recriado com as novas configurações
```

### Deletar Static Pod
```bash
# MÉTODO 1: Remover o arquivo YAML
ssh user@node01
sudo rm /etc/kubernetes/manifests/my-pod.yaml
# Kubelet detecta e deleta o pod automaticamente

# MÉTODO 2: Mover o arquivo para outro diretório
sudo mv /etc/kubernetes/manifests/my-pod.yaml /tmp/
# Pod é deletado (e pode ser restaurado movendo de volta)
```

### ⚠️ O que NÃO funciona
```bash
# Tentar deletar via kubectl NÃO funciona
kubectl delete pod my-static-pod-node01
# O kubelet recria o pod imediatamente!

# Tentar editar via kubectl NÃO persiste
kubectl edit pod my-static-pod-node01
# Mudanças são descartadas na próxima reconciliação
```

## 🎯 Casos de Uso

### 1. **Componentes do Control Plane** (uso principal)
```bash
# Ver os static pods do control plane
kubectl get pods -n kube-system -o wide

# Você verá pods como:
# - etcd-controlplane
# - kube-apiserver-controlplane
# - kube-controller-manager-controlplane
# - kube-scheduler-controlplane
```

**Por que usar static pods para control plane?**
- Kubelet pode iniciar os pods **sem** depender do API Server
- Se o API Server cair, kubelet ainda consegue reiniciar os componentes
- Bootstrapping: os componentes podem iniciar antes do cluster estar funcional

### 2. **Serviços críticos de infraestrutura**
- Agentes de monitoramento críticos
- Proxies de rede locais
- Serviços que precisam iniciar antes do cluster estar pronto

### 3. **Troubleshooting e debugging**
- Testar configurações de pods sem afetar o cluster
- Rodar ferramentas de debug em nós específicos

## 📊 Static Pods vs DaemonSets

| Característica | Static Pods | DaemonSets |
|----------------|-------------|------------|
| **Gerenciado por** | Kubelet | DaemonSet Controller (API Server) |
| **Precisa do API Server** | ❌ Não | ✅ Sim |
| **Roda em todos os nós** | ❌ Não (1 nó específico) | ✅ Sim |
| **Pode usar nodeSelector** | ❌ Não | ✅ Sim |
| **Gerenciável via kubectl** | ❌ Não (só leitura) | ✅ Sim |
| **Uso típico** | Control Plane | Agentes, logging, monitoring |
| **Quando usar** | Bootstrapping, services críticos | Serviços que rodam em múltiplos nós |

## 🧪 Exemplo Prático Completo

### Cenário: Criar um Static Pod no nó worker01

```bash
# 1. SSH no nó
ssh user@worker01

# 2. Descobrir o diretório de static pods
sudo cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Output: staticPodPath: /etc/kubernetes/manifests

# 3. Criar o arquivo YAML
sudo cat <<EOF > /etc/kubernetes/manifests/nginx-static.yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-static
  labels:
    app: nginx
    tier: frontend
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    resources:
      requests:
        cpu: 100m
        memory: 128Mi
      limits:
        cpu: 200m
        memory: 256Mi
    volumeMounts:
    - name: html
      mountPath: /usr/share/nginx/html
  volumes:
  - name: html
    hostPath:
      path: /var/www/html
      type: DirectoryOrCreate
EOF

# 4. Verificar se o arquivo foi criado
sudo ls -la /etc/kubernetes/manifests/

# 5. Aguardar alguns segundos e verificar o pod
kubectl get pods -o wide | grep nginx-static
# Output: nginx-static-worker01   1/1   Running   0   10s   worker01

# 6. Ver detalhes do pod
kubectl describe pod nginx-static-worker01

# 7. Verificar que é um static pod (ownerReferences)
kubectl get pod nginx-static-worker01 -o yaml | grep -A 5 ownerReferences
# Deve mostrar: kind: Node

# 8. Tentar deletar via kubectl (não funciona!)
kubectl delete pod nginx-static-worker01
# O pod é recriado imediatamente

# 9. Deletar corretamente (remover arquivo)
sudo rm /etc/kubernetes/manifests/nginx-static.yaml

# 10. Verificar que o pod foi deletado
kubectl get pods | grep nginx-static
# Nenhum resultado
```

## 🔧 Configuração do Kubelet para Static Pods

### Método 1: Via arquivo de configuração (recomendado)

```yaml
# /var/lib/kubelet/config.yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
staticPodPath: /etc/kubernetes/manifests  # ← Define o diretório
```

### Método 2: Via flag na linha de comando

```bash
# Adicionar flag ao kubelet
kubelet --pod-manifest-path=/etc/kubernetes/manifests ...
```

### Verificar configuração atual

```bash
# Ver processo do kubelet
ps aux | grep kubelet

# Ver arquivo de configuração
sudo cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Ver flags
sudo systemctl status kubelet
```

### Alterar o diretório de Static Pods

```bash
# 1. Editar configuração do kubelet
sudo vi /var/lib/kubelet/config.yaml

# 2. Alterar ou adicionar:
staticPodPath: /meu/novo/caminho

# 3. Reiniciar kubelet
sudo systemctl restart kubelet

# 4. Verificar se está rodando
sudo systemctl status kubelet

# 5. Mover os arquivos YAML para o novo diretório
sudo mv /etc/kubernetes/manifests/*.yaml /meu/novo/caminho/
```

## 🔍 Troubleshooting

### Problema: Static Pod não aparece

**1. Verificar se o arquivo está no diretório correto**
```bash
# Descobrir o diretório configurado
sudo cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Listar arquivos
sudo ls -la /etc/kubernetes/manifests/
```

**2. Verificar sintaxe do YAML**
```bash
# Validar YAML
kubectl apply --dry-run=client -f /etc/kubernetes/manifests/my-pod.yaml
```

**3. Verificar logs do kubelet**
```bash
# Ver logs do kubelet
sudo journalctl -u kubelet -f

# Procurar por erros relacionados ao static pod
sudo journalctl -u kubelet | grep -i "static pod"
```

**4. Verificar permissões do arquivo**
```bash
# Arquivo deve ser legível pelo kubelet
sudo chmod 644 /etc/kubernetes/manifests/my-pod.yaml
```

### Problema: Pod é recriado após deletar via kubectl

**Isso é comportamento normal!** Static pods são gerenciados pelo kubelet.

**Solução:**
```bash
# Deletar o arquivo YAML no nó
ssh user@node01
sudo rm /etc/kubernetes/manifests/my-pod.yaml
```

### Problema: Mudanças via kubectl edit não persistem

**Isso é comportamento normal!** Static pods só podem ser editados via arquivo YAML.

**Solução:**
```bash
# Editar o arquivo YAML no nó
ssh user@node01
sudo vi /etc/kubernetes/manifests/my-pod.yaml
# Salvar - kubelet detecta e atualiza automaticamente
```

### Problema: Kubelet não está monitorando o diretório

**1. Verificar se o kubelet está rodando**
```bash
sudo systemctl status kubelet
```

**2. Verificar se staticPodPath está configurado**
```bash
sudo cat /var/lib/kubelet/config.yaml | grep staticPodPath
```

**3. Reiniciar kubelet**
```bash
sudo systemctl restart kubelet
sudo systemctl status kubelet
```

## 📚 Comandos Úteis - Resumo

### Descobrir Static Pods
```bash
# Listar todos os pods (static pods têm sufixo do nó)
kubectl get pods -A -o wide

# Verificar ownerReferences (procure por kind: Node)
kubectl get pod <pod-name> -o yaml | grep -A 5 ownerReferences

# Ver qual nó está rodando o static pod
kubectl get pod <pod-name> -o jsonpath='{.spec.nodeName}'
```

### Gerenciar Static Pods
```bash
# Descobrir o diretório de static pods
sudo cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Listar static pods definidos no nó
sudo ls -la /etc/kubernetes/manifests/

# Criar static pod
sudo vi /etc/kubernetes/manifests/my-pod.yaml

# Editar static pod
sudo vi /etc/kubernetes/manifests/my-pod.yaml

# Deletar static pod
sudo rm /etc/kubernetes/manifests/my-pod.yaml
```

### Troubleshooting
```bash
# Ver logs do kubelet
sudo journalctl -u kubelet -f

# Ver status do kubelet
sudo systemctl status kubelet

# Reiniciar kubelet
sudo systemctl restart kubelet

# Validar YAML
kubectl apply --dry-run=client -f /etc/kubernetes/manifests/my-pod.yaml
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **O que são Static Pods**
   - Gerenciados pelo kubelet (não pelo API Server)
   - Definidos por arquivos YAML em diretório local

2. **Como identificar Static Pods**
   - Nome do pod tem sufixo do nó (ex: `nginx-node01`)
   - ownerReferences tem `kind: Node`

3. **Onde os Static Pods são definidos**
   - Descobrir com: `cat /var/lib/kubelet/config.yaml | grep staticPodPath`
   - Caminho padrão: `/etc/kubernetes/manifests`

4. **Como criar/editar/deletar Static Pods**
   - Criar: colocar YAML no diretório de manifests
   - Editar: editar o arquivo YAML no nó
   - Deletar: remover o arquivo YAML do diretório

5. **Diferença entre Static Pods e DaemonSets**
   - Static Pods: 1 nó específico, sem API Server
   - DaemonSets: todos os nós, gerenciado pelo API Server

6. **Static Pods não podem ser gerenciados via kubectl**
   - `kubectl delete` não funciona (pod é recriado)
   - `kubectl edit` não persiste mudanças

### 🧪 Cenário típico na prova:

> **"Crie um static pod chamado 'redis-pod' no nó 'worker02' usando a imagem 'redis:alpine'. O pod deve ter um volume hostPath montado em /data no container e apontando para /opt/redis/data no host."**

**Solução:**
```bash
# 1. SSH no nó
ssh worker02

# 2. Descobrir diretório de static pods
sudo cat /var/lib/kubelet/config.yaml | grep staticPodPath
# Output: /etc/kubernetes/manifests

# 3. Criar o arquivo YAML
sudo vi /etc/kubernetes/manifests/redis-pod.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: redis-pod
spec:
  containers:
  - name: redis
    image: redis:alpine
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    hostPath:
      path: /opt/redis/data
      type: DirectoryOrCreate
```

```bash
# 4. Salvar e sair (:wq)
# Kubelet cria o pod automaticamente

# 5. Verificar
kubectl get pods -o wide | grep redis-pod
# Output: redis-pod-worker02   1/1   Running   0   10s   worker02
```

## 📖 Recursos para Estudo

### Documentação Oficial
- [Static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)
- [Kubelet Configuration](https://kubernetes.io/docs/reference/config-api/kubelet-config.v1beta1/)
- [Create static Pods](https://kubernetes.io/docs/tasks/configure-pod-container/static-pod/)

### Comparação Rápida
```
┌─────────────────────────────────────────────────────┐
│                    Workload Types                   │
├─────────────────────────────────────────────────────┤
│ Pod          → Unidade básica                       │
│ ReplicaSet   → Mantém N réplicas                    │
│ Deployment   → ReplicaSet + rollout                 │
│ DaemonSet    → 1 pod por nó (via API Server)       │
│ Static Pod   → 1 pod por nó (via kubelet)          │
│ StatefulSet  → Pods com identidade (não coberto)   │
│ Job          → Execução única (não coberto)        │
└─────────────────────────────────────────────────────┘
```

---

⬅️ **Anterior**: [daemonsets.md](./daemonsets.md) | ➡️ **Próximo**: [scheduling.md](./scheduling.md)
