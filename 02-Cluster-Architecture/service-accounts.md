# Service Accounts

## O que são Service Accounts?

O Kubernetes opera com dois tipos de usuários:

1. **Usuários humanos** — interagem via `kubectl`, certificados, etc.
2. **Aplicações/Bots** — Prometheus, Jenkins, scripts automatizados

**Service Accounts** são usadas para autenticar **aplicações e bots** na API do Kubernetes.

---

## Criar e Usar um Service Account

```bash
# Criar um service account
kubectl create serviceaccount meu-sa
kubectl create sa meu-sa

# Listar service accounts
kubectl get serviceaccounts
kubectl get sa

# Descrever
kubectl describe serviceaccount meu-sa
```

---

## Como os Pods usam Service Accounts

Por padrão, cada namespace tem um service account **`default`**. Todo Pod naquele namespace é automaticamente montado com o token do `default` service account.

Para usar um SA específico:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  serviceAccountName: meu-sa    # especificar o SA aqui
  containers:
  - name: myapp
    image: myapp:latest
```

Para desativar o automount do token:

```yaml
spec:
  automountServiceAccountToken: false
```

O token é montado dentro do pod em:
```
/var/run/secrets/kubernetes.io/serviceaccount/token
```

---

## Evolução dos tokens (por versão)

### Antes do Kubernetes v1.22

- Quando um SA era criado, um Secret era gerado automaticamente com um token JWT **sem expiração**
- O token era montado nos Pods como volume

### Kubernetes v1.22+ — TokenRequest API

A partir do v1.22, os tokens passaram a ser obtidos via **TokenRequest API**:

- **Bounded por audiência** (audience-bound)
- **Bounded por tempo** (expiração padrão: 1 hora)
- **Bounded por objeto** (vinculado ao Pod específico)

O token agora é injetado como **Projected Volume** em vez de Secret.

### Kubernetes v1.24+

A criação automática de Secrets de token foi eliminada. Para criar um token manualmente:

```bash
# Gerar token temporário para um SA
kubectl create token meu-sa

# Token com expiração customizada (em segundos)
kubectl create token meu-sa --duration=86400s
```

---

## Criar um Token Permanente (quando necessário)

Apenas quando não é possível usar a TokenRequest API, crie um Secret de token persistente:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: meu-sa-token
  annotations:
    kubernetes.io/service-account.name: meu-sa   # vincular ao SA
type: kubernetes.io/service-account-token
```

```bash
kubectl create -f sa-secret.yaml

# Ver o token gerado
kubectl describe secret meu-sa-token
kubectl get secret meu-sa-token -o jsonpath='{.data.token}' | base64 --decode
```

---

## Exemplo: Aplicação acessando a API do Kubernetes

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: dashboard-sa
---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: dashboard-role
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: dashboard-rolebinding
subjects:
- kind: ServiceAccount
  name: dashboard-sa
  namespace: default
roleRef:
  kind: Role
  name: dashboard-role
  apiGroup: rbac.authorization.k8s.io
---
apiVersion: v1
kind: Pod
metadata:
  name: dashboard-pod
spec:
  serviceAccountName: dashboard-sa
  containers:
  - name: dashboard
    image: dashboard:latest
```

---

## Resumo dos Comandos

```bash
# Criar SA
kubectl create serviceaccount <nome>

# Gerar token temporário
kubectl create token <nome-sa>

# Ver SA de um pod
kubectl get pod <nome> -o jsonpath='{.spec.serviceAccountName}'

# Ver o token montado no pod
kubectl exec <pod> -- cat /var/run/secrets/kubernetes.io/serviceaccount/token
```

---

## Referências

- https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/
- https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/
