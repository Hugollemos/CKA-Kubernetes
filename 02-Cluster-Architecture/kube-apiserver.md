# Kube-API-Server - Guia Completo

## 1. O que é Kube-API-Server?

**Kube-API-Server** é o componente central do Kubernetes que funciona como a **porta de entrada principal** para toda comunicação com o cluster. É o único componente que interage diretamente com o ETCD.

### 1.1 Analogia

Pense no Kube-API-Server como um **porteiro de um banco**:

- Verifica sua identidade (autenticação)
- Valida se você tem permissão (autorização)
- Processa suas solicitações
- Guarda todos os dados (ETCD)

---

## 2. Responsabilidades Principais

### 2.1 Comunicação com o Cluster

- **Único ponto de contato** para interagir com o Kubernetes
- Recebe requisições HTTP/HTTPS de diversos clientes
- Responde com informações do cluster

### 2.2 Autenticação

- Verifica **quem você é**
- Valida credenciais (tokens, certificados, etc.)
- Rejeita requisições não autenticadas

### 2.3 Autorização

- Verifica **o que você pode fazer**
- Implementa políticas de acesso (RBAC)
- Controla permissões por usuário/serviço

### 2.4 Validação

- Valida formato das requisições
- Verifica se os dados estão corretos
- Rejeita requisições inválidas antes de processar

### 2.5 Recuperação e Atualização de Dados

- **Única interface** com o ETCD
- Recupera dados quando solicitado
- Atualiza dados e persiste no ETCD
- Garante consistência

---

## 3. Fluxo de uma Requisição

```
┌────────────────┐
│   Cliente      │
│  (kubectl)     │
└────────┬────────┘
         │ Requisição HTTPS
         ▼
┌───────────────────────────────────────────┐
│      Kube-API-Server                      │
│                                           │
│  1️⃣  Autenticação                        │
│      ├─ Verifica credenciais              │
│      └─ Identifica o usuário/serviço      │
│                                           │
│  2️⃣  Autorização                         │
│      ├─ Verifica permissões (RBAC)        │
│      └─ Permite/nega acesso               │
│                                           │
│  3️⃣  Validação                           │
│      ├─ Valida formato JSON/YAML          │
│      └─ Valida regras de negócio          │
│                                           │
│  4️⃣  Processamento                       │
│      ├─ Processa a solicitação            │
│      └─ Comunica com ETCD                 │
│                                           │
│  5️⃣  Resposta                            │
│      ├─ Retorna dados ou confirmação      │
│      └─ Retorna erros (se houver)         │
└───────────┬─────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────┐
│      ETCD (Banco de Dados)                │
│  (Armazena estado do cluster)             │
└───────────────────────────────────────────┘

```

---

## 4. Não é Necessário Usar Kube-API-Server Diretamente

### 4.1 Alternativa: Usar APIs do Cluster Diretamente

Embora o Kube-API-Server seja a interface padrão, você pode usar as **próprias APIs REST do Kubernetes** diretamente:

```bash
# Método tradicional (via kubectl)
kubectl create pod meu-pod --image=nginx

# Método direto via API REST
curl -X POST https://kubernetes-api.example.com/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @pod.json

```

### 4.2 Casos de Uso para API Direta

**Quando usar a API REST diretamente:**

- Integração em scripts ou aplicações
- Automação programática
- Ferramentas customizadas
- Integração CI/CD
- Monitoramento e observabilidade

### 4.3 Vantagens e Desvantagens

### ✅ Vantagens de Usar Diretamente a API

- Controle total e granular
- Sem dependência do kubectl
- Possibilidade de automação customizada
- Melhor integração com aplicações

### ❌ Desvantagens

- Precisa gerenciar autenticação (tokens/certificados)
- Requer conhecimento detalhado da API
- Sem validação do lado do cliente
- Mais complexo de depurar

---

## 5. Componentes do Kube-API-Server

### 5.1 Arquitetura Interna

```
┌─────────────────────────────────────────┐
│      Kube-API-Server                    │
│                                         │
│  ┌───────────────────────────────────┐ │
│  │  HTTP(S) Handler                  │ │
│  │  (Recebe requisições)             │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  Authentication Modules           │ │
│  │  ├─ Certificate Auth              │ │
│  │  ├─ Token Auth                    │ │
│  │  ├─ Basic Auth                    │ │
│  │  └─ OIDC                          │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  Authorization Modules (RBAC)     │ │
│  │  ├─ Role-Based Access Control     │ │
│  │  ├─ Attribute-Based Access Control│ │
│  │  └─ Webhooks                      │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  Admission Controllers            │ │
│  │  ├─ Validating Webhook            │ │
│  │  ├─ Mutating Webhook              │ │
│  │  └─ Validadores internos          │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  ETCD Interface                   │ │
│  │  └─ Persistência de dados         │ │
│  └─────────────────────────────────────┘ │
│                                         │
└─────────────────────────────────────────┘

```

---

## 6. Exemplos de Uso

### 6.1 Usando kubectl (Via Kube-API-Server)

```bash
# Lista pods
kubectl get pods

# Cria um pod
kubectl create pod meu-pod --image=nginx

# Atualiza um deployment
kubectl set image deployment/meu-app app=nginx:1.21

# Deleta um recurso
kubectl delete pod meu-pod

```

**Nos bastidores:**

- kubectl envia requisição HTTPS para Kube-API-Server
- Kube-API-Server autentica e autoriza
- Valida a solicitação
- Atualiza ETCD
- Retorna resposta

### 6.2 Usando API REST Diretamente

### Obter Token

```bash
# Para service account
kubectl create serviceaccount meu-usuario
TOKEN=$(kubectl describe secret $(kubectl get secret -o name | grep meu-usuario) | grep token: | awk '{print $2}')

```

### Listar Pods via API

```bash
curl -X GET https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  --cacert /etc/kubernetes/pki/ca.crt

```

### Criar Pod via API

```bash
curl -X POST https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --cacert /etc/kubernetes/pki/ca.crt \
  -d '{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "meu-pod",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "name": "nginx",
          "image": "nginx:latest"
        }
      ]
    }
  }'

```

### Atualizar Pod via API

```bash
curl -X PATCH https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods/meu-pod \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/strategic-merge-patch+json" \
  --cacert /etc/kubernetes/pki/ca.crt \
  -d '{
    "spec": {
      "containers": [
        {
          "name": "nginx",
          "image": "nginx:1.21"
        }
      ]
    }
  }'

```

### Deletar Pod via API

```bash
curl -X DELETE https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods/meu-pod \
  -H "Authorization: Bearer $TOKEN" \
  --cacert /etc/kubernetes/pki/ca.crt

```

---

## 7. Métodos HTTP Utilizados

| Método | Operação | Exemplo |
| --- | --- | --- |
| **GET** | Ler/Listar | `GET /api/v1/pods` |
| **POST** | Criar | `POST /api/v1/namespaces/default/pods` |
| **PUT** | Substituir completamente | `PUT /api/v1/namespaces/default/pods/meu-pod` |
| **PATCH** | Atualizar parcialmente | `PATCH /api/v1/namespaces/default/pods/meu-pod` |
| **DELETE** | Deletar | `DELETE /api/v1/namespaces/default/pods/meu-pod` |
| **WATCH** | Monitorar mudanças | `GET /api/v1/pods?watch=true` |

---

## 8. Segurança

### 8.1 Autenticação

**Métodos suportados:**

- **Client Certificates** (X.509)
- **Bearer Tokens** (JWT, OIDC)
- **Basic Authentication** (username:password)
- **Proxy Authentication**
- **OpenID Connect (OIDC)**

### 8.2 Autorização

**Modes:**

- **RBAC** (Role-Based Access Control) - Mais comum
- **ABAC** (Attribute-Based Access Control)
- **Webhook**
- **Node** (para kubelet)

### 8.3 Exemplo de RBAC

```yaml
# Define uma Role com permissões
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
# Associa a Role a um usuário
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: default

```

---

## 9. Interação com Outros Componentes

```
┌──────────────────────┐
│   Componentes        │
│   do Cluster         │
└──────┬───────────────┘
       │
       ├─ kubelet ─────────┐
       │                   │
       ├─ controller-manager
       │                   │
       ├─ scheduler        │
       │                   │
       └─ other clients ───┤
                           │
                    ┌──────▼────────────┐
                    │ Kube-API-Server   │
                    └──────┬────────────┘
                           │
                    ┌──────▼────────────┐
                    │      ETCD         │
                    └───────────────────┘

```

---

## 10. Configuração do Kube-API-Server

O Kube-API-Server pode ser configurado com várias flags:

```bash
kube-apiserver \
  --etcd-servers=https://127.0.0.1:2379 \
  --authorization-mode=RBAC \
  --admission-control=NamespaceLifecycle,LimitRanger \
  --client-ca-file=/etc/kubernetes/pki/ca.crt \
  --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
  --tls-private-key-file=/etc/kubernetes/pki/apiserver.key \
  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
  --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key

```

---

## 11. Troubleshooting

### Erro: Unauthorized

```
The connection to the server localhost:8080 was refused

```

**Causa**: Token inválido ou expirado
**Solução**: Renovar token ou certificado

### Erro: Forbidden

```
Error from server (Forbidden): pods is forbidden

```

**Causa**: Usuário não tem permissão
**Solução**: Verificar RBAC e RoleBindings

### Erro: Validation Error

```
error validating data: data[spec.containers[0].image]: required value

```

**Causa**: Dados enviados estão incompletos ou inválidos
**Solução**: Revisar formato do JSON/YAML

---

## 12. Resumo das Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Comunicação** | Único ponto de entrada para interagir com o cluster |
| **Autenticação** | Verifica identidade do cliente |
| **Autorização** | Verifica permissões (RBAC) |
| **Validação** | Valida formato e regras de negócio |
| **Persistência** | Interage com ETCD para armazenar dados |
| **Orquestração** | Comunica com outros componentes (scheduler, controller) |

---

## 13. Conclusão

- **Kube-API-Server** é o coração comunicativo do Kubernetes
- É responsável por **autenticação, autorização e validação**
- Comunica com **ETCD** para persistir dados
- Não é necessário usá-lo diretamente se preferir usar as **APIs REST do cluster**
- Implementa segurança através de **RBAC e autenticação**
- Todos os componentes do cluster o utilizam

---



## 🔗 Recursos Úteis

### Documentação Oficial

- 📖 [Kubernetes API Overview](https://kubernetes.io/docs/concepts/overview/kubernetes-api/) - Visão geral da API do Kubernetes
- 📖 [Accessing the API](https://kubernetes.io/docs/tasks/administer-cluster/access-cluster-api/)
- 📖 [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- 📖 [Using RBAC Authorization](https://kubernetes.io/docs/reference/access-authn-authz/rbac/)

### Blog Posts e Artigos

- 📝 [SIG Architecture: API Spotlight](https://kubernetes.io/blog/2026/02/12/sig-architecture-api-spotlight/) - Destaque sobre a API do Kubernetes

### Para Aprofundamento (não obrigatório para prova)

Recursos avançados sobre convenções e desenvolvimento da API:

- 🏗️ [API Conventions](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md) - Convenções de design da API
- 🏗️ [API Changes](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api_changes.md) - Como mudanças na API são feitas

**Nota**: Esses recursos sobre convenções e mudanças da API são úteis se você quiser entender melhor a arquitetura interna, mas **não são necessários para o exame CKA**.

---

⬅️ **Anterior**: [backup-restore.md](./backup-restore.md) | ➡️ **Próximo**: [admission-controllers.md](./admission-controllers.md)
