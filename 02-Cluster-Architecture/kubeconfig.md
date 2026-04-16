# KubeConfig

## O que é o KubeConfig?

O arquivo **kubeconfig** centraliza as configurações de acesso a clusters Kubernetes: certificados, credenciais, contextos e endpoints. Por padrão fica em `~/.kube/config`.

Sem ele, você precisaria passar credenciais manualmente em cada comando:

```bash
# Sem kubeconfig (tedioso)
kubectl get pods \
  --server=https://my-kube-apiserver:6443 \
  --client-certificate=admin.crt \
  --client-key=admin.key \
  --certificate-authority=ca.crt
```

Com kubeconfig, basta:

```bash
kubectl get pods
```

---

## Estrutura do KubeConfig

O arquivo tem **3 seções principais**:

```yaml
apiVersion: v1
kind: Config

# 1. CLUSTERS — endpoints dos clusters
clusters:
- name: my-kube-playground
  cluster:
    certificate-authority: /caminho/para/ca.crt
    # ou em base64:
    # certificate-authority-data: <base64>
    server: https://my-kube-playground:6443

- name: production
  cluster:
    certificate-authority: /caminho/para/ca-prod.crt
    server: https://prod-api:6443

# 2. USERS — credenciais dos usuários
users:
- name: my-kube-admin
  user:
    client-certificate: /caminho/para/admin.crt
    client-key: /caminho/para/admin.key

- name: dev-user
  user:
    client-certificate: /caminho/para/dev.crt
    client-key: /caminho/para/dev.key

# 3. CONTEXTS — combina cluster + usuário (+ namespace opcional)
contexts:
- name: my-kube-admin@my-kube-playground
  context:
    cluster: my-kube-playground
    user: my-kube-admin
    namespace: finance   # opcional

- name: dev-user@production
  context:
    cluster: production
    user: dev-user

# Contexto ativo por padrão
current-context: my-kube-admin@my-kube-playground
```

---

## Comandos Essenciais

```bash
# Ver o kubeconfig atual
kubectl config view

# Ver kubeconfig específico
kubectl config view --kubeconfig=meu-config

# Ver o contexto atual
kubectl config current-context

# Listar todos os contextos
kubectl config get-contexts

# Trocar de contexto
kubectl config use-context dev-user@production

# Definir namespace padrão no contexto atual
kubectl config set-context --current --namespace=dev

# Ajuda geral
kubectl config -h
```

---

## Múltiplos Arquivos KubeConfig

Você pode ter múltiplos arquivos kubeconfig e mesclá-los via variável de ambiente:

```bash
# Usar múltiplos arquivos
export KUBECONFIG=~/.kube/config:~/.kube/config-dev:~/.kube/config-prod

# Mesclar e salvar
kubectl config view --merge --flatten > ~/.kube/config-merged
```

Ou passar o arquivo diretamente:

```bash
kubectl get pods --kubeconfig=/caminho/para/meu-config
```

---

## Namespaces no KubeConfig

Contextos podem ter um namespace padrão:

```yaml
contexts:
- name: admin@production
  context:
    cluster: production
    user: admin
    namespace: kube-system   # namespace padrão para este contexto
```

---

## Certificados no KubeConfig

Os certificados podem ser referenciados por **caminho de arquivo** ou **embutidos em base64**:

```yaml
# Por caminho
clusters:
- name: my-cluster
  cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt

# Em base64 (mais portátil)
clusters:
- name: my-cluster
  cluster:
    certificate-authority-data: <conteudo-base64-do-ca.crt>
```

Para converter:
```bash
cat ca.crt | base64 -w 0
```

---

## Referências

- https://kubernetes.io/docs/tasks/access-application-cluster/configure-access-multiple-clusters/
- https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands#config
