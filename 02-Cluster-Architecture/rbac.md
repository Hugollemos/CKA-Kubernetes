# RBAC — Controle de Acesso Baseado em Funções

## Visão Geral de Segurança no Kubernetes

Dois pilares principais de segurança:

1. **Autenticação** — Quem pode acessar? (definido por mecanismos de auth)
2. **Autorização** — O que podem fazer? (definido por mecanismos de authz)

---

## Mecanismos de Autorização

| Mecanismo | Descrição |
|-----------|-----------|
| **Node Authorization** | Permite que kubelets acessem a API do apiserver |
| **ABAC** (Attribute-Based) | Políticas baseadas em atributos — difícil de gerenciar |
| **RBAC** (Role-Based) | Políticas baseadas em funções — padrão recomendado |
| **Webhook** | Autorização delegada a serviço externo |

Os modos são configurados no `kube-apiserver`:

```bash
--authorization-mode=Node,RBAC,Webhook
```

Quando múltiplos modos são especificados, o Kubernetes testa em ordem — se um aprovar, o acesso é concedido.

---

## API Groups

A API do Kubernetes é dividida em grupos:

```
/api           → Core group (v1): pods, services, namespaces, secrets, etc.
/apis          → Named groups: apps, networking.k8s.io, storage.k8s.io, etc.
/healthz
/metrics
/logs
/version
```

### Core Group (`/api/v1`)

Recursos: pods, services, namespaces, configmaps, secrets, events, endpoints, nodes, persistentvolumes, persistentvolumeclaims, etc.

### Named Groups (`/apis`)

```
/apis/apps/v1                    → deployments, replicasets, statefulsets, daemonsets
/apis/batch/v1                   → jobs, cronjobs
/apis/networking.k8s.io/v1       → networkpolicies, ingresses
/apis/storage.k8s.io/v1          → storageclasses
/apis/certificates.k8s.io/v1     → certificatesigningrequests
/apis/rbac.authorization.k8s.io  → roles, rolebindings, clusterroles
```

```bash
# Listar todos os API groups
kubectl api-resources

# Ver recursos de um grupo específico
kubectl api-resources --api-group=apps

# Acessar a API via kubectl proxy (sem precisar de certificados)
kubectl proxy 8001 &
curl http://localhost:8001/apis
```

---

## RBAC — Roles e RoleBindings

### Criar um Role

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
  namespace: default
rules:
- apiGroups: [""]          # "" = core group
  resources: ["pods"]
  verbs: ["get", "list", "create", "update", "delete"]
- apiGroups: [""]
  resources: ["configmaps"]
  verbs: ["create"]
- apiGroups: ["apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "create"]
```

```bash
kubectl create -f developer-role.yaml
```

### Criar um RoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: devuser-developer-binding
  namespace: default
subjects:
- kind: User
  name: dev-user           # nome do usuário (case-sensitive)
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: developer
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f devuser-developer-binding.yaml
```

> Roles e RoleBindings são **com escopo de namespace**.

### Criar via linha de comando

```bash
# Criar role
kubectl create role developer \
  --verb=get,list,create,update,delete \
  --resource=pods

# Criar rolebinding
kubectl create rolebinding devuser-developer-binding \
  --role=developer \
  --user=dev-user
```

### Ver e descrever

```bash
kubectl get roles
kubectl get rolebindings

kubectl describe role developer
kubectl describe rolebinding devuser-developer-binding
```

---

## Verificar Permissões

```bash
# Verificar se EU tenho permissão
kubectl auth can-i create deployments
kubectl auth can-i delete nodes

# Verificar permissão de OUTRO usuário (como admin)
kubectl auth can-i create deployments --as dev-user
kubectl auth can-i create pods --as dev-user --namespace test
```

---

## Restringir a Recursos Específicos (resourceNames)

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: developer
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "update", "create"]
  resourceNames: ["blue", "orange"]  # apenas esses pods específicos
```

---

## ClusterRoles e ClusterRoleBindings

Roles têm escopo de namespace. Alguns recursos são **cluster-scoped** (sem namespace): nodes, persistentvolumes, clusterroles, namespaces, etc.

```bash
# Ver recursos com namespace
kubectl api-resources --namespaced=true

# Ver recursos sem namespace (cluster-scoped)
kubectl api-resources --namespaced=false
```

### ClusterRole

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  name: cluster-administrator
rules:
- apiGroups: [""]
  resources: ["nodes"]
  verbs: ["get", "list", "delete", "create"]
- apiGroups: [""]
  resources: ["persistentvolumes"]
  verbs: ["get", "list", "create", "delete"]
```

### ClusterRoleBinding

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: cluster-admin-role-binding
subjects:
- kind: User
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: ClusterRole
  name: cluster-administrator
  apiGroup: rbac.authorization.k8s.io
```

```bash
kubectl create -f cluster-admin-role.yaml
kubectl create -f cluster-admin-role-binding.yaml
```

> Um ClusterRole pode também ser vinculado a recursos com namespace — nesse caso, o usuário terá acesso ao recurso em **todos os namespaces**.

### Criar via linha de comando

```bash
# Criar ClusterRole
kubectl create clusterrole cluster-administrator \
  --verb=get,list,create,delete \
  --resource=nodes

# Criar ClusterRoleBinding
kubectl create clusterrolebinding cluster-admin-binding \
  --clusterrole=cluster-administrator \
  --user=cluster-admin
```

---

## Resumo: Role vs ClusterRole

| | Role | ClusterRole |
|--|------|-------------|
| Escopo | Namespace | Cluster todo |
| Recursos | Com namespace (pods, services, etc.) | Sem namespace (nodes, PVs, etc.) |
| Binding | RoleBinding | ClusterRoleBinding |
| Pode usar ClusterRole | Sim (via RoleBinding) | Sim (via ClusterRoleBinding) |

---

## Referências

- https://kubernetes.io/docs/reference/access-authn-authz/rbac/
- https://kubernetes.io/docs/reference/access-authn-authz/authorization/
- https://kubernetes.io/docs/concepts/overview/kubernetes-api/
