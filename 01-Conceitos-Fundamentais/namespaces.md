# Namespaces no Kubernetes

## O que são Namespaces?

**Namespaces** são mecanismos de isolamento lógico dentro de um cluster Kubernetes. Eles permitem que múltiplos projetos, equipes ou ambientes compartilhem o mesmo cluster sem interferir uns nos outros.

Por padrão, o Kubernetes cria o namespace **`default`** automaticamente quando o cluster é inicializado.

---

## Namespaces padrão do Kubernetes

| Namespace | Descrição |
|-----------|-----------|
| `default` | Namespace padrão para objetos sem namespace especificado |
| `kube-system` | Componentes internos do Kubernetes (kube-dns, coredns, etc.) |
| `kube-public` | Dados públicos acessíveis por todos |
| `kube-node-lease` | Informações de heartbeat dos nós |

---

## Comandos Essenciais

### Listar Pods

```bash
# Listar pods no namespace default
kubectl get pods

# Listar pods em outro namespace
kubectl get pods --namespace=kube-system
kubectl get pods -n kube-system

# Listar pods em TODOS os namespaces
kubectl get pods --all-namespaces
kubectl get pods -A
```

### Criar objetos em outros namespaces

```bash
# Criar pod em namespace específico via flag
kubectl create -f pod-definition.yaml --namespace=dev

# Ou definir o namespace direto no YAML
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  namespace: dev         # namespace definido aqui
  labels:
    app: myapp
spec:
  containers:
  - name: nginx-container
    image: nginx
```

---

## Criar um Namespace

### Via YAML

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

```bash
kubectl create -f namespace-dev.yaml
```

### Via linha de comando

```bash
kubectl create namespace dev
kubectl create ns dev
```

---

## Mudar o Namespace padrão da sessão

```bash
# Definir o namespace padrão para a sessão
kubectl config set-context $(kubectl config current-context) --namespace=dev

# Verificar contexto atual
kubectl config current-context
```

---

## DNS entre Namespaces

Dentro de um mesmo namespace, os serviços se comunicam pelo nome simples:

```
mysql.connect("db-service")
```

Para acessar um serviço em **outro namespace**, use o FQDN:

```
mysql.connect("db-service.dev.svc.cluster.local")
```

Formato do FQDN:
```
<nome-servico>.<namespace>.svc.cluster.local
```

---

## ResourceQuota — Limitar recursos por Namespace

Para limitar recursos em um namespace, crie um objeto **`ResourceQuota`**:

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    pods: "10"
    requests.cpu: "4"
    requests.memory: 5Gi
    limits.cpu: "10"
    limits.memory: 10Gi
```

```bash
kubectl create -f compute-quota.yaml

# Verificar quotas
kubectl get resourcequota -n dev
kubectl describe resourcequota compute-quota -n dev
```

---

## Resumo dos Comandos

```bash
# Criar namespace
kubectl create namespace <nome>

# Ver todos os namespaces
kubectl get namespaces
kubectl get ns

# Mudar namespace padrão
kubectl config set-context --current --namespace=<nome>

# Listar pods em namespace específico
kubectl get pods -n <nome>

# Listar recursos em todos os namespaces
kubectl get pods -A
kubectl get services -A
```

---

## Referências

- https://kubernetes.io/docs/concepts/overview/working-with-objects/namespaces/
- https://kubernetes.io/docs/tasks/administer-cluster/namespaces-walkthrough/
- https://kubernetes.io/docs/tasks/administer-cluster/manage-resources/quota-memory-cpu-namespace/
