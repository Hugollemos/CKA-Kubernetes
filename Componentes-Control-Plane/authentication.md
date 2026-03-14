# Authentication (Autenticação) no Kubernetes

## 📋 O que é Authentication?

**Authentication** (Autenticação) é o processo de **verificar a identidade** de quem está fazendo uma requisição ao cluster Kubernetes. É a primeira camada de segurança no acesso ao cluster.

### Fluxo de Segurança no Kubernetes

```
┌─────────────────────────────────────────────────────────────┐
│  kubectl get pods                                           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  1. AUTHENTICATION (Autenticação)                           │
│     ├─ Quem é você?                                         │
│     ├─ Certificados, Tokens, Senhas                         │
│     └─ Se falhar: 401 Unauthorized                          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  2. AUTHORIZATION (Autorização)                             │
│     ├─ O que você pode fazer?                               │
│     ├─ RBAC, ABAC, Node, Webhook                            │
│     └─ Se falhar: 403 Forbidden                             │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  3. ADMISSION CONTROL                                       │
│     ├─ Esta requisição é válida?                            │
│     ├─ Mutating + Validating                                │
│     └─ Se falhar: 400 Bad Request / 403 Forbidden           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  4. Persist to etcd                                         │
│     └─ Salvar no cluster                                    │
└─────────────────────────────────────────────────────────────┘
```

## 🎯 Tipos de Usuários no Kubernetes

Kubernetes distingue entre dois tipos de usuários:

### 1. Service Accounts (Contas de Serviço)
- **Gerenciados pelo Kubernetes** (objetos no cluster)
- Usados por **pods/aplicações** dentro do cluster
- Armazenados como secrets
- Exemplo: pod acessa Kubernetes API

### 2. Normal Users (Usuários Normais)
- **NÃO gerenciados pelo Kubernetes** (não existem como objetos)
- Pessoas ou processos externos
- Gerenciados por sistemas externos (certificados, LDAP, OIDC)
- Exemplo: administrador usando kubectl

### Comparação

| Característica | Service Accounts | Normal Users |
|----------------|------------------|--------------|
| **Gerenciado por** | Kubernetes | Sistema externo |
| **Existe como objeto?** | ✅ Sim (ServiceAccount) | ❌ Não |
| **Criação** | `kubectl create sa` | Certificado/OIDC/LDAP |
| **Escopo** | Namespace | Cluster-wide |
| **Usado por** | Pods, aplicações | Pessoas, ferramentas externas |
| **Token** | Montado em /var/run/secrets | Fornecido externamente |

## 🔐 Métodos de Authentication

Kubernetes suporta múltiplos métodos de autenticação:

### 1. Certificados X.509 (Client Certificates)
✅ **Método mais comum para administradores**

**Como funciona:**
- Cliente apresenta certificado assinado pela CA do cluster
- kube-apiserver valida certificado
- `CN` (Common Name) do certificado = username
- `O` (Organization) do certificado = group

**Configuração do kube-apiserver:**
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --client-ca-file=/etc/kubernetes/pki/ca.crt
    - --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
    - --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

**Criar certificado de usuário:**

```bash
# 1. Gerar chave privada
openssl genrsa -out john.key 2048

# 2. Criar Certificate Signing Request (CSR)
openssl req -new -key john.key -out john.csr -subj "/CN=john/O=developers"
# CN=john → username será "john"
# O=developers → grupo será "developers"

# 3. Assinar CSR com CA do cluster
openssl x509 -req -in john.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out john.crt \
  -days 365

# 4. Configurar kubectl com novo certificado
kubectl config set-credentials john \
  --client-certificate=john.crt \
  --client-key=john.key

kubectl config set-context john-context \
  --cluster=kubernetes \
  --user=john

kubectl config use-context john-context

# 5. Testar
kubectl get pods
# Se não tiver permissões RBAC: Error: forbidden
```

**Verificar certificado:**
```bash
openssl x509 -in john.crt -text -noout

# Ver CN e O:
# Subject: O = developers, CN = john
```

### 2. Bearer Tokens (Service Account Tokens)

✅ **Método padrão para Service Accounts**

**Como funciona:**
- Token JWT armazenado em Secret
- Enviado no header: `Authorization: Bearer <token>`
- Token montado automaticamente em pods

**Service Account padrão:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  serviceAccountName: default  # Adicionado automaticamente
  containers:
  - name: nginx
    image: nginx
```

**Token montado em:**
```bash
/var/run/secrets/kubernetes.io/serviceaccount/token
```

**Criar Service Account customizado:**

```bash
# 1. Criar ServiceAccount
kubectl create serviceaccount my-app

# 2. Usar em pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: my-app
spec:
  serviceAccountName: my-app
  containers:
  - name: app
    image: nginx
EOF

# 3. Ver token do ServiceAccount (Kubernetes < 1.24)
kubectl get secret $(kubectl get sa my-app -o jsonpath='{.secrets[0].name}') -o jsonpath='{.data.token}' | base64 -d

# 4. Para Kubernetes >= 1.24 (token de curta duração)
kubectl create token my-app
```

**Usar token manualmente:**

```bash
# 1. Obter token
TOKEN=$(kubectl create token my-app)

# 2. Usar com curl
curl -k https://kubernetes.default.svc/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN"

# 3. Configurar kubectl
kubectl config set-credentials my-app --token=$TOKEN
kubectl config set-context my-app-context --cluster=kubernetes --user=my-app
kubectl config use-context my-app-context
```

### 3. Static Token File

⚠️ **Não recomendado para produção (inseguro)**

**Como funciona:**
- Arquivo CSV com tokens estáticos
- Formato: `token,username,uid,group1,group2`

**Configuração:**

```bash
# 1. Criar arquivo de tokens
cat <<EOF > /etc/kubernetes/token-auth.csv
abc123,john,1001,developers,admins
def456,jane,1002,viewers
EOF

# 2. Configurar kube-apiserver
# Editar /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --token-auth-file=/etc/kubernetes/token-auth.csv
    volumeMounts:
    - name: token-file
      mountPath: /etc/kubernetes/token-auth.csv
      readOnly: true
  volumes:
  - name: token-file
    hostPath:
      path: /etc/kubernetes/token-auth.csv
```

**Usar:**
```bash
curl -k https://kubernetes:6443/api/v1/pods \
  -H "Authorization: Bearer abc123"
```

**Problemas:**
- ❌ Tokens nunca expiram
- ❌ Não pode rotacionar sem reiniciar API server
- ❌ Armazenados em texto plano
- ❌ Difícil de auditar

### 4. Static Password File

⚠️ **Não recomendado (muito inseguro)**

Similar ao token file, mas com senhas:

```csv
password1,john,1001,developers
password2,jane,1002,viewers
```

**Configuração:**
```yaml
- --basic-auth-file=/etc/kubernetes/basic-auth.csv
```

**Usar:**
```bash
curl -k https://kubernetes:6443/api/v1/pods \
  -u john:password1
```

**Problemas:** Mesmos do Static Token File + transmissão de senha

### 5. OpenID Connect (OIDC)

✅ **Método recomendado para empresas**

**Como funciona:**
- Integra com provedores OIDC (Google, Azure AD, Okta, Keycloak)
- Usuário autentica no provedor OIDC
- Provedor retorna JWT token
- Token enviado ao kube-apiserver
- API server valida token com provedor

**Configuração do kube-apiserver:**

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --oidc-issuer-url=https://accounts.google.com
    - --oidc-client-id=kubernetes
    - --oidc-username-claim=email
    - --oidc-groups-claim=groups
    - --oidc-ca-file=/etc/kubernetes/pki/oidc-ca.crt
```

**Fluxo:**

```
┌─────────────┐
│   Usuário   │
└──────┬──────┘
       │ 1. Login
       ↓
┌─────────────────┐
│ OIDC Provider   │  (Google, Azure AD, Okta)
│ (Google, Okta)  │
└──────┬──────────┘
       │ 2. ID Token (JWT)
       ↓
┌─────────────┐
│   kubectl   │
└──────┬──────┘
       │ 3. API request + Token
       ↓
┌──────────────────┐
│ kube-apiserver   │
│ ├─ Valida token  │
│ └─ Extrai user   │
└──────────────────┘
```

**Configurar kubectl com OIDC:**

```bash
kubectl config set-credentials john \
  --auth-provider=oidc \
  --auth-provider-arg=idp-issuer-url=https://accounts.google.com \
  --auth-provider-arg=client-id=kubernetes \
  --auth-provider-arg=client-secret=secret \
  --auth-provider-arg=refresh-token=xxxxx \
  --auth-provider-arg=id-token=yyyyy
```

### 6. Webhook Token Authentication

✅ **Autenticação customizada via webhook externo**

**Como funciona:**
- kube-apiserver envia token para webhook externo
- Webhook valida e retorna informações do usuário

**Configuração:**

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --authentication-token-webhook-config-file=/etc/kubernetes/webhook-config.yaml
    - --authentication-token-webhook-cache-ttl=30s
```

**Arquivo de configuração do webhook:**

```yaml
apiVersion: v1
kind: Config
clusters:
- name: auth-webhook
  cluster:
    server: https://auth-service.default.svc:8443/authenticate
    certificate-authority: /etc/kubernetes/pki/ca.crt
contexts:
- name: webhook
  context:
    cluster: auth-webhook
current-context: webhook
```

**Requisição ao webhook:**

```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "spec": {
    "token": "abc123xyz"
  }
}
```

**Resposta do webhook:**

```json
{
  "apiVersion": "authentication.k8s.io/v1",
  "kind": "TokenReview",
  "status": {
    "authenticated": true,
    "user": {
      "username": "john",
      "uid": "1001",
      "groups": ["developers", "admins"]
    }
  }
}
```

### 7. Authenticating Proxy

**Como funciona:**
- Proxy externo autentica usuários
- Passa headers para kube-apiserver
- API server confia no proxy

**Headers:**
- `X-Remote-User`: username
- `X-Remote-Group`: grupos
- `X-Remote-Extra-*`: atributos extras

**Configuração:**

```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --requestheader-client-ca-file=/etc/kubernetes/pki/proxy-ca.crt
    - --requestheader-username-headers=X-Remote-User
    - --requestheader-group-headers=X-Remote-Group
    - --requestheader-extra-headers-prefix=X-Remote-Extra-
```

## 📊 Comparação de Métodos

| Método | Segurança | Complexidade | Uso | Produção? |
|--------|-----------|--------------|-----|-----------|
| **X.509 Certificates** | ✅ Alta | Média | Admins, kubelets | ✅ Sim |
| **Service Account Tokens** | ✅ Alta | Baixa | Pods, apps | ✅ Sim |
| **OIDC** | ✅ Muito Alta | Alta | Empresas | ✅ Sim |
| **Webhook** | ✅ Alta | Alta | Custom | ✅ Sim |
| **Static Token File** | ❌ Baixa | Baixa | Testes | ❌ Não |
| **Static Password File** | ❌ Muito Baixa | Baixa | Nunca | ❌ Nunca |
| **Authenticating Proxy** | ✅ Alta | Muito Alta | Casos específicos | ⚠️ Depende |

## 🔐 Certificados TLS no Kubernetes

### O que são Certificados TLS?

**TLS (Transport Layer Security)** é um protocolo criptográfico que garante comunicação segura. No Kubernetes, certificados TLS são usados para:

1. **Autenticação**: Provar identidade (quem sou eu)
2. **Criptografia**: Proteger dados em trânsito
3. **Integridade**: Garantir que dados não foram alterados

### Por que Kubernetes usa TLS?

Kubernetes é um sistema distribuído onde vários componentes se comunicam pela rede:

```
┌──────────────┐                  ┌──────────────┐
│   kubectl    │─────TLS──────────│ kube-apiserver│
└──────────────┘                  └───────┬──────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    │                     │                     │
                   TLS                   TLS                   TLS
                    │                     │                     │
              ┌─────▼─────┐         ┌────▼────┐          ┌─────▼─────┐
              │   etcd    │         │ kubelet │          │ scheduler │
              └───────────┘         └─────────┘          └───────────┘
```

**Sem TLS**: Qualquer um poderia interceptar, modificar ou falsificar comunicações (ataque man-in-the-middle).

**Com TLS**: Todas as comunicações são criptografadas e autenticadas.

### Componentes dos Certificados TLS

#### 1. Certificate Authority (CA)

**CA** é a autoridade que assina e valida certificados.

- Kubernetes tem sua própria CA interna
- CA possui:
  - **Certificado CA** (`ca.crt`): certificado público da CA
  - **Chave Privada CA** (`ca.key`): chave secreta para assinar certificados

**Localização padrão:**
```bash
/etc/kubernetes/pki/ca.crt       # Certificado CA (público)
/etc/kubernetes/pki/ca.key       # Chave privada CA (SECRETO)
```

#### 2. Certificado (Certificate)

Certificado é um documento digital que contém:
- **Public Key** (chave pública): pode ser compartilhada
- **Identidade**: CN (Common Name), O (Organization)
- **Assinatura da CA**: prova que CA validou este certificado
- **Validade**: datas de início e expiração

#### 3. Chave Privada (Private Key)

- Chave secreta correspondente ao certificado
- NUNCA deve ser compartilhada
- Usada para descriptografar e assinar

### Certificados no Kubernetes

#### Certificados do Cluster (kubeadm)

Quando você cria um cluster com kubeadm, ele gera automaticamente todos os certificados:

```bash
/etc/kubernetes/pki/
├── ca.crt                          # CA do cluster
├── ca.key
├── apiserver.crt                   # Certificado do API Server
├── apiserver.key
├── apiserver-kubelet-client.crt    # API Server se conecta ao kubelet
├── apiserver-kubelet-client.key
├── front-proxy-ca.crt              # CA para aggregation layer
├── front-proxy-ca.key
├── front-proxy-client.crt
├── front-proxy-client.key
├── sa.pub                          # Chave pública para ServiceAccount tokens
├── sa.key                          # Chave privada para ServiceAccount tokens
└── etcd/
    ├── ca.crt                      # CA do ETCD
    ├── ca.key
    ├── server.crt                  # Certificado do servidor ETCD
    ├── server.key
    ├── peer.crt                    # Certificado para comunicação peer-to-peer
    ├── peer.key
    ├── healthcheck-client.crt      # Cliente para health checks
    └── healthcheck-client.key
```

#### Propósito de Cada Certificado

| Certificado | Usado Por | Propósito |
|-------------|-----------|-----------|
| **ca.crt/key** | Cluster CA | Assinar todos os certificados do cluster |
| **apiserver.crt/key** | kube-apiserver | Servidor TLS do API Server |
| **apiserver-kubelet-client.crt/key** | kube-apiserver | Cliente para se conectar ao kubelet |
| **kubelet.crt/key** | kubelet | Servidor TLS do kubelet |
| **front-proxy-ca.crt/key** | Aggregation Layer | CA para API Aggregation |
| **etcd/ca.crt/key** | ETCD CA | Assinar certificados do ETCD |
| **etcd/server.crt/key** | etcd | Servidor TLS do ETCD |
| **etcd/peer.crt/key** | etcd | Comunicação entre membros do ETCD cluster |

### Anatomia de um Certificado X.509

```bash
# Ver conteúdo de um certificado
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

**Saída (partes importantes):**

```
Certificate:
    Data:
        Version: 3 (0x2)
        Serial Number: 1234567890
    Signature Algorithm: sha256WithRSAEncryption
        Issuer: CN = kubernetes
        Validity
            Not Before: Jan  1 00:00:00 2024 GMT
            Not After : Dec 31 23:59:59 2025 GMT
        Subject: CN = kube-apiserver
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (2048 bit)
        X509v3 extensions:
            X509v3 Key Usage: critical
                Digital Signature, Key Encipherment
            X509v3 Extended Key Usage:
                TLS Web Server Authentication
            X509v3 Subject Alternative Name:
                DNS:kubernetes, DNS:kubernetes.default, DNS:kubernetes.default.svc,
                DNS:kubernetes.default.svc.cluster.local,
                IP Address:10.96.0.1, IP Address:192.168.1.100
```

**Campos importantes:**

- **Issuer**: Quem assinou (CA)
- **Subject**: Para quem é o certificado (CN = Common Name)
- **Validity**: Período de validade
- **Subject Alternative Name (SAN)**: DNS names e IPs adicionais
- **Key Usage**: Para que pode ser usado (server auth, client auth)

### Criar Certificados Manualmente

#### Método 1: OpenSSL (Tradicional)

**Passo 1: Criar CA (se não existir)**

```bash
# 1. Gerar chave privada da CA
openssl genrsa -out ca.key 2048

# 2. Criar certificado auto-assinado da CA
openssl req -x509 -new -nodes \
  -key ca.key \
  -subj "/CN=kubernetes-ca" \
  -days 3650 \
  -out ca.crt
```

**Passo 2: Criar certificado de usuário**

```bash
# 1. Gerar chave privada do usuário
openssl genrsa -out john.key 2048

# 2. Criar Certificate Signing Request (CSR)
openssl req -new \
  -key john.key \
  -subj "/CN=john/O=developers/O=admins" \
  -out john.csr
# CN=john → username será "john"
# O=developers → grupo "developers"
# O=admins → grupo "admins" (pode ter múltiplos)

# 3. Assinar CSR com CA do cluster
openssl x509 -req \
  -in john.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out john.crt \
  -days 365

# 4. Verificar certificado
openssl x509 -in john.crt -text -noout
```

#### Método 2: Kubernetes CertificateSigningRequest (Recomendado)

**Vantagens:**
- ✅ Não precisa acessar `ca.key` diretamente
- ✅ Aprovação controlada via RBAC
- ✅ Gerenciado pelo Kubernetes
- ✅ Auditável

**Processo completo:**

```bash
# 1. Gerar chave privada
openssl genrsa -out jane.key 2048

# 2. Criar CSR
openssl req -new \
  -key jane.key \
  -subj "/CN=jane/O=developers" \
  -out jane.csr

# 3. Codificar CSR em base64 (em uma linha)
cat jane.csr | base64 | tr -d '\n'

# 4. Criar CertificateSigningRequest no Kubernetes
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  request: $(cat jane.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  expirationSeconds: 31536000  # 1 ano (365 dias)
  usages:
  - client auth
EOF

# 5. Ver status do CSR
kubectl get csr jane

# 6. Aprovar CSR (requer permissão RBAC)
kubectl certificate approve jane

# 7. Obter certificado assinado
kubectl get csr jane -o jsonpath='{.status.certificate}' | base64 -d > jane.crt

# 8. Verificar certificado
openssl x509 -in jane.crt -text -noout

# 9. Configurar kubectl
kubectl config set-credentials jane \
  --client-certificate=jane.crt \
  --client-key=jane.key \
  --embed-certs=true

kubectl config set-context jane-context \
  --cluster=kubernetes \
  --user=jane

# 10. Usar
kubectl config use-context jane-context
```

### SignerNames no Kubernetes

Diferentes tipos de certificados usam diferentes **signers**:

| SignerName | Uso | CA |
|------------|-----|-----|
| `kubernetes.io/kube-apiserver-client` | Clientes do API Server (usuários) | `/etc/kubernetes/pki/ca.crt` |
| `kubernetes.io/kube-apiserver-client-kubelet` | Kubelet como cliente | `/etc/kubernetes/pki/ca.crt` |
| `kubernetes.io/kubelet-serving` | Kubelet como servidor | `/var/lib/kubelet/pki/kubelet-ca.crt` |
| `kubernetes.io/legacy-unknown` | Signer customizado | Configurável |

### Subject Alternative Names (SANs)

Para certificados de **servidor** (não clientes), SANs são críticos!

**Problema sem SANs:**
```bash
# API Server com certificado apenas para "kubernetes"
# Tentar acessar via IP
curl https://192.168.1.100:6443
# Erro: certificate is valid for kubernetes, not 192.168.1.100
```

**Solução: Adicionar SANs**

Criar arquivo `san.cnf`:
```ini
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
DNS.5 = my-cluster.example.com
IP.1 = 10.96.0.1
IP.2 = 192.168.1.100
```

Gerar certificado com SANs:
```bash
# Criar CSR com SANs
openssl req -new \
  -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -out apiserver.csr \
  -config san.cnf

# Assinar com SANs
openssl x509 -req \
  -in apiserver.csr \
  -CA ca.crt \
  -CAkey ca.key \
  -CAcreateserial \
  -out apiserver.crt \
  -days 365 \
  -extensions v3_req \
  -extfile san.cnf

# Verificar SANs
openssl x509 -in apiserver.crt -text -noout | grep -A 10 "Subject Alternative Name"
```

### Verificar e Validar Certificados

#### Ver informações do certificado

```bash
# Ver detalhes completos
openssl x509 -in cert.crt -text -noout

# Ver apenas subject
openssl x509 -in cert.crt -noout -subject
# Output: subject=CN = john, O = developers

# Ver apenas issuer
openssl x509 -in cert.crt -noout -issuer
# Output: issuer=CN = kubernetes-ca

# Ver apenas datas de validade
openssl x509 -in cert.crt -noout -dates
# Output:
# notBefore=Jan  1 00:00:00 2024 GMT
# notAfter=Dec 31 23:59:59 2025 GMT

# Ver apenas SANs
openssl x509 -in cert.crt -text -noout | grep -A 1 "Subject Alternative Name"
```

#### Verificar se certificado foi assinado pela CA

```bash
# Verificar cadeia de confiança
openssl verify -CAfile /etc/kubernetes/pki/ca.crt john.crt
# Output: john.crt: OK

# Se falhar:
# Output: error 20 at 0 depth lookup:unable to get local issuer certificate
```

#### Verificar se certificado e chave privada combinam

```bash
# Hash da chave pública no certificado
openssl x509 -noout -modulus -in cert.crt | openssl md5

# Hash da chave pública na chave privada
openssl rsa -noout -modulus -in key.key | openssl md5

# Se os hashes forem iguais, combinam!
```

#### Verificar expiração de todos os certificados

```bash
# Método 1: kubeadm (mais fácil)
kubeadm certs check-expiration

# Output:
# CERTIFICATE                EXPIRES                  RESIDUAL TIME   CERTIFICATE AUTHORITY   EXTERNALLY MANAGED
# admin.conf                 Dec 31, 2024 23:59 UTC   364d            ca                      no
# apiserver                  Dec 31, 2024 23:59 UTC   364d            ca                      no
# apiserver-kubelet-client   Dec 31, 2024 23:59 UTC   364d            ca                      no
# ...

# Método 2: Manualmente
for cert in /etc/kubernetes/pki/*.crt; do
  echo "=== $cert ==="
  openssl x509 -in $cert -noout -enddate
done
```

### Renovar Certificados

#### Renovar todos os certificados (kubeadm)

```bash
# 1. Backup dos certificados antigos
sudo cp -r /etc/kubernetes/pki /etc/kubernetes/pki.backup

# 2. Renovar todos os certificados
sudo kubeadm certs renew all

# 3. Reiniciar componentes do control plane
# (kubeadm reinicia automaticamente os static pods)

# 4. Verificar
kubeadm certs check-expiration

# 5. Atualizar kubeconfig
sudo cp /etc/kubernetes/admin.conf ~/.kube/config
```

#### Renovar certificado específico

```bash
# Renovar apenas certificado do API Server
sudo kubeadm certs renew apiserver

# Renovar apenas admin.conf
sudo kubeadm certs renew admin.conf

# Listar certificados que podem ser renovados
kubeadm certs renew --help
```

#### Renovar manualmente (sem kubeadm)

```bash
# 1. Gerar novo CSR
openssl req -new \
  -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -out apiserver.csr \
  -config san.cnf

# 2. Assinar com CA
openssl x509 -req \
  -in apiserver.csr \
  -CA /etc/kubernetes/pki/ca.crt \
  -CAkey /etc/kubernetes/pki/ca.key \
  -CAcreateserial \
  -out apiserver-new.crt \
  -days 365 \
  -extensions v3_req \
  -extfile san.cnf

# 3. Backup do certificado antigo
sudo mv /etc/kubernetes/pki/apiserver.crt /etc/kubernetes/pki/apiserver.crt.old

# 4. Mover novo certificado
sudo mv apiserver-new.crt /etc/kubernetes/pki/apiserver.crt

# 5. Reiniciar API Server
sudo kill -9 $(pidof kube-apiserver)
# kubelet reinicia automaticamente
```

### Troubleshooting de Certificados

#### Erro: "x509: certificate has expired"

```bash
# Verificar expiração
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Renovar certificados
sudo kubeadm certs renew all

# Reiniciar control plane
sudo systemctl restart kubelet
```

#### Erro: "x509: certificate signed by unknown authority"

```bash
# Problema: CA não é confiável

# Verificar qual CA assinou
openssl x509 -in cert.crt -noout -issuer

# Verificar se CA está correta
openssl verify -CAfile /etc/kubernetes/pki/ca.crt cert.crt

# Solução: Gerar novo certificado assinado pela CA correta
```

#### Erro: "x509: certificate is valid for X, not Y"

```bash
# Problema: SANs não incluem o hostname/IP usado

# Ver SANs atuais
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A 10 "Subject Alternative Name"

# Solução: Gerar novo certificado com SANs corretos
```

#### Erro: "tls: private key does not match public key in certificate"

```bash
# Problema: Certificado e chave privada não combinam

# Verificar hash do certificado
openssl x509 -noout -modulus -in cert.crt | openssl md5

# Verificar hash da chave
openssl rsa -noout -modulus -in key.key | openssl md5

# Se diferentes, use a chave privada correta
```

#### Verificar logs do API Server

```bash
# Ver logs relacionados a TLS
sudo journalctl -u kubelet -f | grep -i "tls\|certificate\|x509"

# Ver logs do pod do API Server
kubectl logs -n kube-system kube-apiserver-<node> | grep -i "tls\|certificate"
```

### Boas Práticas de Certificados

1. **Validade dos Certificados**
   - Certificados de produção: 1 ano (máximo)
   - Certificados de desenvolvimento: 90 dias
   - Renovar 30 dias antes de expirar

2. **Proteção da CA**
   - ⚠️ `ca.key` é CRÍTICO - se comprometido, todo cluster está em risco
   - Armazene `ca.key` offline após criar cluster
   - Use CertificateSigningRequest em vez de assinar diretamente

3. **Rotação de Certificados**
   - Configure renovação automática (cert-manager, kubeadm auto-renewal)
   - Monitore expiração com alertas

4. **SANs Completos**
   - Inclua todos os hostnames e IPs possíveis
   - Certificados de servidor SEMPRE precisam de SANs

5. **Backup**
   - Backup de `/etc/kubernetes/pki` regularmente
   - Armazene em local seguro e criptografado

6. **Auditoria**
   - Registre aprovações de CSR
   - Monitore criação de certificados

### Comandos Úteis de Certificados

```bash
# Ver expiração de todos os certificados
kubeadm certs check-expiration

# Renovar todos os certificados
sudo kubeadm certs renew all

# Ver certificado
openssl x509 -in cert.crt -text -noout

# Verificar certificado com CA
openssl verify -CAfile ca.crt cert.crt

# Ver datas de validade
openssl x509 -in cert.crt -noout -dates

# Ver subject e issuer
openssl x509 -in cert.crt -noout -subject -issuer

# Extrair certificado de servidor remoto
echo | openssl s_client -connect kubernetes.example.com:6443 -showcerts 2>/dev/null | openssl x509 -text

# Ver SANs
openssl x509 -in cert.crt -text -noout | grep -A 5 "Subject Alternative Name"
```

## 🔧 Configuração Prática

### Exemplo 1: Criar usuário com certificado

```bash
# 1. Gerar chave
openssl genrsa -out developer.key 2048

# 2. Criar CSR
openssl req -new -key developer.key -out developer.csr \
  -subj "/CN=developer/O=development"

# 3. Criar CertificateSigningRequest no Kubernetes
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: developer
spec:
  request: $(cat developer.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# 4. Aprovar CSR
kubectl certificate approve developer

# 5. Obter certificado
kubectl get csr developer -o jsonpath='{.status.certificate}' | base64 -d > developer.crt

# 6. Configurar kubectl
kubectl config set-credentials developer \
  --client-certificate=developer.crt \
  --client-key=developer.key \
  --embed-certs=true

kubectl config set-context developer-context \
  --cluster=kubernetes \
  --user=developer \
  --namespace=default

# 7. Usar
kubectl config use-context developer-context
kubectl get pods
# Error: forbidden (precisa de RBAC)
```

### Exemplo 2: ServiceAccount com RBAC

```bash
# 1. Criar ServiceAccount
kubectl create serviceaccount app-reader

# 2. Criar Role (permissões)
cat <<EOF | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
  namespace: default
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list", "watch"]
EOF

# 3. Criar RoleBinding (conectar SA com Role)
kubectl create rolebinding app-reader-binding \
  --role=pod-reader \
  --serviceaccount=default:app-reader \
  --namespace=default

# 4. Usar em pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app-reader
  containers:
  - name: app
    image: nginx
EOF

# 5. Testar de dentro do pod
kubectl exec app -- sh -c "
  TOKEN=\$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)
  curl -k https://kubernetes.default.svc/api/v1/namespaces/default/pods \
    -H 'Authorization: Bearer \$TOKEN'
"
```

### Exemplo 3: Bloquear acesso anônimo

```yaml
# Editar /etc/kubernetes/manifests/kube-apiserver.yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --anonymous-auth=false  # Desabilita acesso anônimo
```

**Testar:**
```bash
# Sem autenticação (deve falhar)
curl -k https://kubernetes:6443/api/v1/pods
# Error: Unauthorized
```

## 🛠️ Comandos Úteis

### Ver usuário atual

```bash
kubectl config view --minify

# Ver apenas username
kubectl auth whoami
```

### Ver métodos de auth configurados

```bash
# Ver configuração do kube-apiserver
kubectl -n kube-system get pod kube-apiserver-<node> -o yaml | \
  grep -E "client-ca-file|token-auth|oidc|webhook"
```

### Testar permissões (com auth-can-i)

```bash
# Posso criar pods?
kubectl auth can-i create pods

# Posso deletar deployments no namespace prod?
kubectl auth can-i delete deployments --namespace=prod

# Verificar para outro usuário (requer permissão de impersonate)
kubectl auth can-i get pods --as=developer --as-group=development
```

### Ver ServiceAccounts

```bash
# Listar ServiceAccounts
kubectl get serviceaccounts
kubectl get sa

# Ver detalhes (incluindo secrets)
kubectl describe sa default

# Ver token (K8s >= 1.24)
kubectl create token default

# Ver token (K8s < 1.24)
kubectl get secret $(kubectl get sa default -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{.data.token}' | base64 -d
```

### Verificar certificado

```bash
# Ver informações do certificado
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Verificar validade
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Ver CN e O
openssl x509 -in user.crt -noout -subject
```

### Impersonate (assumir identidade)

```bash
# Simular ser outro usuário (requer permissão)
kubectl get pods --as=developer --as-group=development

# Útil para testar RBAC
kubectl auth can-i get pods --as=developer
```

## 🔍 Troubleshooting

### Erro: "Unauthorized"

```bash
# 401 Unauthorized = falha na autenticação

# Verificar:
# 1. Certificado válido?
openssl x509 -in ~/.kube/client.crt -noout -dates

# 2. Certificado assinado pela CA correta?
openssl verify -CAfile /etc/kubernetes/pki/ca.crt ~/.kube/client.crt

# 3. Token válido?
kubectl create token default --duration=1h

# 4. Contexto correto?
kubectl config current-context
kubectl config view
```

### Erro: "Forbidden"

```bash
# 403 Forbidden = autenticou mas não tem permissão (problema de RBAC)

# Ver permissões do usuário
kubectl auth can-i --list

# Verificar RoleBindings
kubectl get rolebindings
kubectl get clusterrolebindings

# Ver detalhes de binding específico
kubectl describe rolebinding <name>
```

### ServiceAccount não tem permissões

```bash
# 1. Verificar se ServiceAccount existe
kubectl get sa <sa-name>

# 2. Verificar se há RoleBinding/ClusterRoleBinding
kubectl get rolebinding -A | grep <sa-name>
kubectl get clusterrolebinding | grep <sa-name>

# 3. Criar binding se necessário
kubectl create rolebinding <name> \
  --role=<role> \
  --serviceaccount=<namespace>:<sa-name>
```

### Token expirado (Kubernetes >= 1.24)

```bash
# Tokens de ServiceAccount expiram em 1h por padrão

# Criar novo token com duração customizada
kubectl create token my-app --duration=24h

# Para token que não expira (não recomendado), criar Secret manualmente
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: my-app-token
  annotations:
    kubernetes.io/service-account.name: my-app
type: kubernetes.io/service-account-token
EOF
```

### Certificado expirado

```bash
# Verificar validade
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# Renovar certificados (kubeadm)
kubeadm certs renew all

# Verificar após renovação
kubeadm certs check-expiration
```

## 📚 Authentication vs Authorization vs Admission

| Fase | Pergunta | Resposta em caso de falha | Exemplo |
|------|----------|---------------------------|---------|
| **Authentication** | Quem é você? | 401 Unauthorized | Certificado inválido |
| **Authorization** | O que você pode fazer? | 403 Forbidden | Sem permissão RBAC |
| **Admission Control** | Esta requisição é válida? | 403 Forbidden / 400 Bad Request | ResourceQuota excedida |

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Diferença entre Service Accounts e Normal Users**
   - Service Accounts: gerenciados pelo K8s, usados por pods
   - Normal Users: externos, não são objetos no cluster

2. **Criar usuário com certificado X.509**
   - Gerar chave privada
   - Criar CSR com CN=username e O=group
   - Assinar com CA do cluster (ou usar CertificateSigningRequest)
   - Configurar kubectl

3. **Service Accounts**
   - Criar: `kubectl create sa <name>`
   - Usar em pod: `serviceAccountName: <name>`
   - Token montado em: `/var/run/secrets/kubernetes.io/serviceaccount/token`

4. **Métodos de autenticação**
   - Certificados (X.509): admins, nodes
   - Tokens (Bearer): Service Accounts
   - OIDC: empresas
   - Webhook: customizado

5. **Comandos essenciais**
   - `kubectl auth whoami`: ver usuário atual
   - `kubectl auth can-i <verb> <resource>`: testar permissões
   - `kubectl config use-context`: trocar contexto
   - `kubectl create token`: criar token de SA

6. **Certificados TLS**
   - CN (Common Name) = username
   - O (Organization) = group (pode ter múltiplos)
   - Verificar expiração: `kubeadm certs check-expiration`
   - Renovar certificados: `kubeadm certs renew all`
   - Verificar certificado: `openssl x509 -in cert.crt -text -noout`
   - Localização: `/etc/kubernetes/pki/`

7. **CertificateSigningRequest**
   - Método recomendado para criar usuários
   - Não precisa acessar `ca.key` diretamente
   - Fluxo: criar CSR → aprovar → obter certificado
   - `kubectl certificate approve <csr-name>`

### 🧪 Cenários típicos na prova:

#### Cenário 1: Criar usuário com certificado

> **"Crie um usuário 'john' no grupo 'developers' usando certificado X.509."**

**Solução:**
```bash
# 1. Gerar chave
openssl genrsa -out john.key 2048

# 2. Criar CSR
openssl req -new -key john.key -out john.csr -subj "/CN=john/O=developers"

# 3. Criar CertificateSigningRequest
cat <<EOF | kubectl apply -f -
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: john
spec:
  request: $(cat john.csr | base64 | tr -d '\n')
  signerName: kubernetes.io/kube-apiserver-client
  usages:
  - client auth
EOF

# 4. Aprovar
kubectl certificate approve john

# 5. Obter certificado
kubectl get csr john -o jsonpath='{.status.certificate}' | base64 -d > john.crt

# 6. Configurar kubectl
kubectl config set-credentials john --client-certificate=john.crt --client-key=john.key
kubectl config set-context john-context --cluster=kubernetes --user=john
```

#### Cenário 2: ServiceAccount para aplicação

> **"Crie um ServiceAccount 'app-sa' e configure um pod para usá-lo."**

**Solução:**
```bash
# 1. Criar ServiceAccount
kubectl create serviceaccount app-sa

# 2. Criar pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  serviceAccountName: app-sa
  containers:
  - name: nginx
    image: nginx
EOF

# 3. Verificar
kubectl get pod app -o yaml | grep serviceAccountName
```

#### Cenário 3: Obter token de ServiceAccount

> **"Obtenha o token do ServiceAccount 'default' para usar em autenticação externa."**

**Solução:**
```bash
# Para Kubernetes >= 1.24
kubectl create token default

# Para Kubernetes < 1.24
kubectl get secret $(kubectl get sa default -o jsonpath='{.secrets[0].name}') \
  -o jsonpath='{.data.token}' | base64 -d
```

#### Cenário 4: Verificar e renovar certificados

> **"Verifique a expiração dos certificados do cluster e renove se necessário."**

**Solução:**
```bash
# 1. Verificar expiração de todos os certificados
kubeadm certs check-expiration

# 2. Se certificados estão perto de expirar, fazer backup
sudo cp -r /etc/kubernetes/pki /etc/kubernetes/pki.backup

# 3. Renovar todos os certificados
sudo kubeadm certs renew all

# 4. Verificar novamente
kubeadm certs check-expiration

# 5. Atualizar kubeconfig (se necessário)
sudo cp /etc/kubernetes/admin.conf ~/.kube/config

# 6. Reiniciar kubelet
sudo systemctl restart kubelet
```

#### Cenário 5: Troubleshooting de certificado expirado

> **"O API Server não está respondendo. Investigue se é problema de certificado."**

**Solução:**
```bash
# 1. Verificar expiração dos certificados
kubeadm certs check-expiration

# 2. Se expirado, verificar certificado específico
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -noout -dates

# 3. Ver logs do API Server
sudo journalctl -u kubelet | grep -i "certificate\|x509"

# 4. Se certificado expirado, renovar
sudo kubeadm certs renew apiserver

# 5. Reiniciar control plane
sudo systemctl restart kubelet

# 6. Verificar se API Server está funcionando
kubectl get nodes
```

#### Cenário 6: Verificar certificado de usuário

> **"Um usuário 'developer' não consegue se autenticar. Verifique o certificado."**

**Solução:**
```bash
# 1. Ver informações do certificado
openssl x509 -in developer.crt -text -noout

# 2. Verificar CN e O (username e grupo)
openssl x509 -in developer.crt -noout -subject
# Deve mostrar: subject=CN = developer, O = <grupo>

# 3. Verificar se foi assinado pela CA do cluster
openssl verify -CAfile /etc/kubernetes/pki/ca.crt developer.crt
# Deve mostrar: developer.crt: OK

# 4. Verificar validade
openssl x509 -in developer.crt -noout -dates
# Verificar se ainda está válido

# 5. Verificar se certificado e chave combinam
openssl x509 -noout -modulus -in developer.crt | openssl md5
openssl rsa -noout -modulus -in developer.key | openssl md5
# Hashes devem ser iguais
```

## 💡 Dicas para a Prova

1. **CN = username, O = group**
   - No certificado X.509, sempre use `/CN=<username>/O=<group>`

2. **CertificateSigningRequest vs assinatura manual**
   - CSR (preferido): `kubectl certificate approve`
   - Manual: `openssl x509 -req -CA ca.crt -CAkey ca.key`

3. **Service Account padrão**
   - Todo namespace tem SA `default`
   - Todo pod usa SA (se não especificar, usa `default`)

4. **Token de SA mudou no K8s 1.24+**
   - Antes: token permanente em Secret
   - Depois: token temporário via `kubectl create token`

5. **Authentication ≠ Authorization**
   - Authentication: quem é você (401 se falhar)
   - Authorization (RBAC): o que pode fazer (403 se falhar)

6. **Teste sempre com `kubectl auth can-i`**
   ```bash
   kubectl auth can-i create pods
   kubectl auth can-i get pods --as=developer
   ```

7. **Certificados TLS - Comandos críticos**
   - Verificar expiração: `kubeadm certs check-expiration`
   - Renovar todos: `sudo kubeadm certs renew all`
   - Ver certificado: `openssl x509 -in cert.crt -text -noout`
   - Verificar com CA: `openssl verify -CAfile ca.crt user.crt`

8. **Localização dos certificados**
   - Cluster CA: `/etc/kubernetes/pki/ca.crt`
   - API Server: `/etc/kubernetes/pki/apiserver.crt`
   - ETCD: `/etc/kubernetes/pki/etcd/`

9. **CertificateSigningRequest é o método preferido**
   - Não precisa acessar `ca.key` diretamente (mais seguro)
   - Use `signerName: kubernetes.io/kube-apiserver-client` para usuários
   - Aprovar com `kubectl certificate approve <name>`

10. **Certificado expirado = cluster quebrado**
    - Sempre verifique expiração periodicamente
    - Configure alertas para certificados que vão expirar
    - Renovar ANTES de expirar (30 dias de antecedência)

## 🔗 Recursos Úteis

### Documentação Oficial
- 📖 [Authenticating](https://kubernetes.io/docs/reference/access-authn-authz/authentication/)
- 📖 [Managing Service Accounts](https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/)
- 📖 [Certificate Signing Requests](https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/)
- 📖 [User impersonation](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#user-impersonation)
- 📖 [PKI certificates and requirements](https://kubernetes.io/docs/setup/best-practices/certificates/)
- 📖 [Certificate Management with kubeadm](https://kubernetes.io/docs/tasks/administer-cluster/kubeadm/kubeadm-certs/)
- 📖 [Manual certificate management](https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/)

### Comandos Rápidos de Revisão

```bash
# Ver usuário atual
kubectl auth whoami

# Criar ServiceAccount
kubectl create sa <name>

# Criar token de SA
kubectl create token <sa-name>

# Criar usuário com certificado (via CSR)
kubectl certificate approve <csr-name>

# Testar permissões
kubectl auth can-i <verb> <resource>
kubectl auth can-i --list

# Impersonate
kubectl get pods --as=<user> --as-group=<group>

# Ver configuração do kubectl
kubectl config view
kubectl config current-context

# === CERTIFICADOS TLS ===

# Verificar expiração de certificados
kubeadm certs check-expiration

# Renovar todos os certificados
sudo kubeadm certs renew all

# Renovar certificado específico
sudo kubeadm certs renew apiserver

# Ver informações de certificado
openssl x509 -in cert.crt -text -noout

# Ver apenas subject (CN e O)
openssl x509 -in cert.crt -noout -subject

# Ver apenas validade
openssl x509 -in cert.crt -noout -dates

# Verificar se certificado foi assinado pela CA
openssl verify -CAfile /etc/kubernetes/pki/ca.crt user.crt

# Verificar se certificado e chave combinam
openssl x509 -noout -modulus -in cert.crt | openssl md5
openssl rsa -noout -modulus -in key.key | openssl md5

# Criar CSR
openssl req -new -key user.key -subj "/CN=username/O=group" -out user.csr

# Aprovar CertificateSigningRequest
kubectl certificate approve <csr-name>

# Ver CSRs pendentes
kubectl get csr

# Obter certificado aprovado
kubectl get csr <name> -o jsonpath='{.status.certificate}' | base64 -d > user.crt
```

---

⬅️ **Anterior**: [kube-apiserver.md](./kube-apiserver.md) | ➡️ **Próximo**: [admission-controllers.md](./admission-controllers.md)
