# ConfigMaps, Secrets e Environment Variables

## 📋 O que são ConfigMaps e Secrets?

**ConfigMaps** e **Secrets** são objetos do Kubernetes usados para **separar configurações** do código da aplicação, seguindo o princípio de 12-factor apps.

### ConfigMaps
- ✅ Armazenam **dados de configuração não-sensíveis**
- ✅ Formato: chave-valor ou arquivos de configuração
- ✅ Exemplos: URLs, flags de features, configurações de aplicação

### Secrets
- ✅ Armazenam **dados sensíveis** (senhas, tokens, chaves)
- ✅ Codificados em **base64** (não criptografados por padrão!)
- ✅ Podem ser criptografados em repouso (encryption at rest)
- ✅ Mais protegidos que ConfigMaps (permissões RBAC)

## 🗂️ ConfigMaps

### Criar ConfigMaps

#### Método 1: Literal (chave-valor)

```bash
# Criar ConfigMap com valores literais
kubectl create configmap app-config \
  --from-literal=APP_ENV=production \
  --from-literal=LOG_LEVEL=info \
  --from-literal=MAX_CONNECTIONS=100

# Ver o ConfigMap
kubectl get configmap app-config -o yaml
```

#### Método 2: De arquivo

```bash
# Criar arquivo de configuração
cat > app.properties <<EOF
database.host=mysql.default.svc.cluster.local
database.port=3306
database.name=myapp
cache.enabled=true
cache.ttl=3600
EOF

# Criar ConfigMap do arquivo
kubectl create configmap app-config --from-file=app.properties

# Ver conteúdo
kubectl describe configmap app-config
```

#### Método 3: De diretório

```bash
# Criar diretório com múltiplos arquivos
mkdir config/
echo "server.port=8080" > config/server.conf
echo "debug=true" > config/app.conf

# Criar ConfigMap do diretório (todos os arquivos)
kubectl create configmap app-config --from-file=config/

# Cada arquivo vira uma chave no ConfigMap
```

#### Método 4: YAML declarativo

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: default
data:
  # Chave-valor simples
  APP_ENV: "production"
  LOG_LEVEL: "info"
  MAX_CONNECTIONS: "100"

  # Arquivo de configuração completo
  app.properties: |
    database.host=mysql.default.svc.cluster.local
    database.port=3306
    database.name=myapp
    cache.enabled=true
    cache.ttl=3600

  # JSON inline
  config.json: |
    {
      "server": {
        "port": 8080,
        "host": "0.0.0.0"
      },
      "features": {
        "auth": true,
        "cache": true
      }
    }
```

```bash
kubectl apply -f configmap.yaml
```

### Usar ConfigMaps em Pods

#### Como Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Injetar UMA chave do ConfigMap
    - name: APP_ENVIRONMENT
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV

    # Injetar TODAS as chaves do ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
```

#### Como Volume (arquivos)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config-volume
      mountPath: /etc/config    # Diretório onde será montado
      readOnly: true

  volumes:
  - name: config-volume
    configMap:
      name: app-config
      # Opcional: selecionar chaves específicas
      items:
      - key: app.properties
        path: application.properties    # Nome do arquivo
```

**Resultado:**
- Arquivo criado em: `/etc/config/application.properties`
- Conteúdo: valor da chave `app.properties` do ConfigMap

#### Exemplo Completo

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    server {
      listen 8080;
      server_name localhost;

      location / {
        root /usr/share/nginx/html;
        index index.html;
      }

      location /health {
        return 200 "OK\n";
        add_header Content-Type text/plain;
      }
    }
---
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: nginx-config-volume
      mountPath: /etc/nginx/conf.d/
      readOnly: true

  volumes:
  - name: nginx-config-volume
    configMap:
      name: nginx-config
      items:
      - key: nginx.conf
        path: default.conf
```

## 🔐 Secrets

### Tipos de Secrets

| Tipo | Uso |
|------|-----|
| **Opaque** | Genérico (padrão) |
| **kubernetes.io/service-account-token** | Token de ServiceAccount |
| **kubernetes.io/dockercfg** | Credenciais Docker (legado) |
| **kubernetes.io/dockerconfigjson** | Credenciais Docker (novo) |
| **kubernetes.io/basic-auth** | Autenticação básica |
| **kubernetes.io/ssh-auth** | Chaves SSH |
| **kubernetes.io/tls** | Certificados TLS |

### Criar Secrets

#### Método 1: Literal

```bash
# Criar Secret com valores literais
kubectl create secret generic app-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=secret123 \
  --from-literal=API_KEY=abc123xyz

# Ver Secret (values ficam ocultos)
kubectl get secret app-secret
kubectl get secret app-secret -o yaml

# Decodificar manualmente
echo "c2VjcmV0MTIz" | base64 --decode
# Output: secret123
```

#### Método 2: De arquivo

```bash
# Criar arquivo com dados sensíveis
echo -n "admin" > username.txt
echo -n "secret123" > password.txt

# Criar Secret dos arquivos
kubectl create secret generic db-credentials \
  --from-file=username=username.txt \
  --from-file=password=password.txt

# Limpar arquivos sensíveis
rm username.txt password.txt
```

#### Método 3: YAML declarativo

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
data:
  # Valores devem estar em base64
  DB_USER: YWRtaW4=           # "admin" em base64
  DB_PASSWORD: c2VjcmV0MTIz   # "secret123" em base64
  API_KEY: YWJjMTIzeHl6       # "abc123xyz" em base64
```

**Codificar em base64:**
```bash
echo -n "admin" | base64
# Output: YWRtaW4=

echo -n "secret123" | base64
# Output: c2VjcmV0MTIz
```

**IMPORTANTE**: base64 não é criptografia! Qualquer pessoa pode decodificar.

#### Método 4: stringData (mais fácil)

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
type: Opaque
stringData:
  # Valores em texto plano (Kubernetes converte para base64)
  DB_USER: admin
  DB_PASSWORD: secret123
  API_KEY: abc123xyz
```

### Secret Tipos Especiais

#### TLS Secret

```bash
# Criar certificado e chave (exemplo)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt -subj "/CN=example.com"

# Criar TLS Secret
kubectl create secret tls my-tls-secret \
  --cert=tls.crt \
  --key=tls.key

# Usar em Ingress
```

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: my-tls-secret
type: kubernetes.io/tls
data:
  tls.crt: LS0tLS1CRUdJTi...  # Certificado em base64
  tls.key: LS0tLS1CRUdJTi...  # Chave privada em base64
```

#### Docker Registry Secret

```bash
# Criar secret para pull de imagens privadas
kubectl create secret docker-registry regcred \
  --docker-server=myregistry.com \
  --docker-username=myuser \
  --docker-password=mypassword \
  --docker-email=myemail@example.com

# Usar em pod
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: private-app
spec:
  imagePullSecrets:
  - name: regcred    # ← Referencia o secret
  containers:
  - name: app
    image: myregistry.com/myapp:1.0
```

### Usar Secrets em Pods

#### Como Environment Variables

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Injetar UMA chave do Secret
    - name: DATABASE_PASSWORD
      valueFrom:
        secretKeyRef:
          name: app-secret
          key: DB_PASSWORD

    # Injetar TODAS as chaves do Secret
    envFrom:
    - secretRef:
        name: app-secret
```

#### Como Volume (arquivos)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: secret-volume
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: secret-volume
    secret:
      secretName: app-secret
      # Opcional: selecionar chaves específicas
      items:
      - key: DB_PASSWORD
        path: db-password    # Arquivo: /etc/secrets/db-password
      # Opcional: definir permissões do arquivo
      defaultMode: 0400      # Somente leitura pelo owner
```

## 🌍 Environment Variables

### Definir Environment Variables Diretamente

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    env:
    # Valor literal (hardcoded)
    - name: APP_NAME
      value: "MyApplication"

    # De um ConfigMap
    - name: APP_ENV
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: APP_ENV

    # De um Secret
    - name: DB_PASSWORD
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password

    # De um FieldRef (metadados do pod)
    - name: POD_NAME
      valueFrom:
        fieldRef:
          fieldPath: metadata.name

    - name: POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace

    - name: POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP

    # De um ResourceFieldRef (recursos do container)
    - name: CPU_REQUEST
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: requests.cpu

    - name: MEMORY_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: app
          resource: limits.memory
```

### Injetar Todas as Variáveis

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  containers:
  - name: app
    image: myapp:1.0
    # Injetar TODOS os valores de um ConfigMap
    envFrom:
    - configMapRef:
        name: app-config
    # Injetar TODOS os valores de um Secret
    - secretRef:
        name: app-secret
```

## 🔒 Encryption at Rest (Criptografia em Repouso)

Por padrão, Secrets são armazenados em **base64 no etcd**, **NÃO criptografados**!

### Habilitar Encryption at Rest

#### 1. Criar EncryptionConfiguration

```yaml
# encryption-config.yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  # Provedor primário (usado para criptografar novos dados)
  - aescbc:
      keys:
      - name: key1
        secret: <BASE64_ENCODED_32_BYTE_KEY>

  # Provedor secundário (usado para descriptografar dados antigos)
  - identity: {}    # Sem criptografia (fallback)
```

**Gerar chave de 32 bytes:**
```bash
# Gerar chave aleatória de 32 bytes
head -c 32 /dev/urandom | base64

# Output exemplo:
# K7VqNZj8JW3h9LHQ5kP2M8aB6T9cD1eF4gH5iJ6kL7m=
```

**Arquivo completo:**
```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
- resources:
  - secrets
  providers:
  - aescbc:
      keys:
      - name: key1
        secret: SzdxTjhKVzNoOUxIUTVrUDJNOGFCNlQ5Y0QxZUY0Z0g1aUo2a0w3bT0K
  - identity: {}
```

#### 2. Configurar kube-apiserver

```bash
# Copiar arquivo de encryption para o control plane
sudo cp encryption-config.yaml /etc/kubernetes/pki/

# Editar manifest do kube-apiserver
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

Adicionar flags:
```yaml
spec:
  containers:
  - command:
    - kube-apiserver
    - --encryption-provider-config=/etc/kubernetes/pki/encryption-config.yaml
    # ... outros parâmetros
    volumeMounts:
    - name: encryption-config
      mountPath: /etc/kubernetes/pki/encryption-config.yaml
      readOnly: true
  volumes:
  - name: encryption-config
    hostPath:
      path: /etc/kubernetes/pki/encryption-config.yaml
      type: File
```

#### 3. Reiniciar kube-apiserver

```bash
# Static pod é reiniciado automaticamente após editar o manifest
# Aguardar alguns segundos

# Verificar se API Server está rodando
kubectl get pods -n kube-system | grep apiserver
```

#### 4. Re-criptografar Secrets Existentes

```bash
# Secrets antigos ainda estão em base64!
# Precisa atualizar todos os secrets para serem criptografados

# Re-criptografar todos os secrets
kubectl get secrets --all-namespaces -o json | \
  kubectl replace -f -

# Verificar se foi criptografado
# Não deve mais ser legível em base64 no etcd
```

### Tipos de Provedores de Criptografia

| Provider | Descrição |
|----------|-----------|
| **identity** | Sem criptografia (base64) |
| **aescbc** | AES-CBC com chaves de 32 bytes (recomendado) |
| **aesgcm** | AES-GCM com chaves de 16, 24 ou 32 bytes |
| **secretbox** | XSalsa20 e Poly1305 (requer idade >= 1.27) |
| **kms** | KMS externo (AWS KMS, GCP KMS, Azure Key Vault) |

### Verificar se Secrets estão Criptografados

```bash
# Conectar ao etcd
ETCDCTL_API=3 etcdctl \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key \
  get /registry/secrets/default/my-secret

# Se CRIPTOGRAFADO, verá: k8s:enc:aescbc:v1:key1:<dados binários>
# Se NÃO criptografado, verá: k8s:enc:identity:<base64 legível>
```

## 🧪 Exemplos Práticos

### Exemplo 1: Aplicação com ConfigMap e Secret

```yaml
# 1. ConfigMap com configurações
apiVersion: v1
kind: ConfigMap
metadata:
  name: webapp-config
data:
  APP_NAME: "MyWebApp"
  LOG_LEVEL: "info"
  CACHE_ENABLED: "true"

  nginx.conf: |
    server {
      listen 8080;
      location / {
        proxy_pass http://localhost:3000;
      }
    }
---
# 2. Secret com credenciais
apiVersion: v1
kind: Secret
metadata:
  name: webapp-secret
type: Opaque
stringData:
  DB_USER: "admin"
  DB_PASSWORD: "SuperSecret123!"
  API_KEY: "abc123xyz789"
---
# 3. Deployment usando ConfigMap e Secret
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: webapp
  template:
    metadata:
      labels:
        app: webapp
    spec:
      containers:
      - name: app
        image: mywebapp:1.0
        ports:
        - containerPort: 3000

        # Environment variables de ConfigMap e Secret
        env:
        - name: APP_NAME
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: APP_NAME
        - name: LOG_LEVEL
          valueFrom:
            configMapKeyRef:
              name: webapp-config
              key: LOG_LEVEL
        - name: DB_USER
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: DB_USER
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: webapp-secret
              key: DB_PASSWORD

        # Volume com arquivo de configuração
        volumeMounts:
        - name: config-volume
          mountPath: /etc/nginx/

      volumes:
      - name: config-volume
        configMap:
          name: webapp-config
          items:
          - key: nginx.conf
            path: nginx.conf
```

### Exemplo 2: Secret montado como arquivo

```bash
# 1. Criar Secret com múltiplas chaves
kubectl create secret generic db-config \
  --from-literal=host=mysql.default.svc.cluster.local \
  --from-literal=port=3306 \
  --from-literal=database=myapp \
  --from-literal=username=admin \
  --from-literal=password=secret123

# 2. Pod que monta Secret como diretório
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app-with-db
spec:
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "ls -la /secrets && cat /secrets/* && sleep 3600"]
    volumeMounts:
    - name: db-secrets
      mountPath: /secrets
      readOnly: true

  volumes:
  - name: db-secrets
    secret:
      secretName: db-config
      defaultMode: 0400
EOF

# 3. Verificar arquivos criados
kubectl exec app-with-db -- ls -la /secrets
# Output:
# -r--------    1 root     root            35 Feb 26 12:00 database
# -r--------    1 root     root            35 Feb 26 12:00 host
# -r--------    1 root     root             9 Feb 26 12:00 password
# -r--------    1 root     root             4 Feb 26 12:00 port
# -r--------    1 root     root             5 Feb 26 12:00 username
```

## 📚 Comandos Úteis

### ConfigMaps

```bash
# Criar
kubectl create configmap <name> --from-literal=key=value
kubectl create configmap <name> --from-file=<file>
kubectl create configmap <name> --from-file=<dir>/

# Listar
kubectl get configmaps
kubectl get cm    # alias

# Ver detalhes
kubectl describe configmap <name>
kubectl get configmap <name> -o yaml

# Editar
kubectl edit configmap <name>

# Deletar
kubectl delete configmap <name>
```

### Secrets

```bash
# Criar
kubectl create secret generic <name> --from-literal=key=value
kubectl create secret generic <name> --from-file=<file>
kubectl create secret tls <name> --cert=cert.crt --key=cert.key
kubectl create secret docker-registry <name> --docker-server=<server>

# Listar
kubectl get secrets
kubectl get secret    # alias

# Ver detalhes (dados ficam ocultos)
kubectl describe secret <name>

# Ver YAML (dados em base64)
kubectl get secret <name> -o yaml

# Decodificar um valor
kubectl get secret <name> -o jsonpath='{.data.password}' | base64 --decode

# Editar
kubectl edit secret <name>

# Deletar
kubectl delete secret <name>
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Criar ConfigMap**
   ```bash
   kubectl create configmap app-config --from-literal=APP_ENV=prod
   ```

2. **Criar Secret**
   ```bash
   kubectl create secret generic db-secret --from-literal=password=secret123
   ```

3. **Usar ConfigMap em Pod (env)**
   ```yaml
   env:
   - name: APP_ENV
     valueFrom:
       configMapKeyRef:
         name: app-config
         key: APP_ENV
   ```

4. **Usar Secret em Pod (env)**
   ```yaml
   env:
   - name: DB_PASSWORD
     valueFrom:
       secretKeyRef:
         name: db-secret
         key: password
   ```

5. **Montar ConfigMap/Secret como volume**
   ```yaml
   volumeMounts:
   - name: config
     mountPath: /etc/config
   volumes:
   - name: config
     configMap:
       name: app-config
   ```

6. **Codificar/decodificar base64**
   ```bash
   echo -n "secret" | base64          # Codificar
   echo "c2VjcmV0" | base64 --decode  # Decodificar
   ```

### 🧪 Cenários típicos na prova:

> **"Crie um ConfigMap chamado 'app-config' com a chave 'database.host' e valor 'mysql.default.svc.cluster.local'. Depois crie um pod que use esse ConfigMap como variável de ambiente DB_HOST."**

```bash
# 1. Criar ConfigMap
kubectl create configmap app-config \
  --from-literal=database.host=mysql.default.svc.cluster.local

# 2. Criar Pod
kubectl run app --image=nginx --dry-run=client -o yaml > pod.yaml

# 3. Editar para adicionar env
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: nginx
    image: nginx
    env:
    - name: DB_HOST
      valueFrom:
        configMapKeyRef:
          name: app-config
          key: database.host
EOF

# 4. Verificar
kubectl exec app -- env | grep DB_HOST
```

> **"Crie um Secret chamado 'db-secret' com username='admin' e password='secret123'. Use esse Secret em um pod chamado 'webapp'."**

```bash
# 1. Criar Secret
kubectl create secret generic db-secret \
  --from-literal=username=admin \
  --from-literal=password=secret123

# 2. Criar Pod
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  containers:
  - name: app
    image: nginx
    env:
    - name: DB_USER
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: username
    - name: DB_PASS
      valueFrom:
        secretKeyRef:
          name: db-secret
          key: password
EOF

# 3. Verificar
kubectl exec webapp -- env | grep DB_
```

## 💡 Dicas para a Prova

1. **Use --from-literal para rapidez**
   ```bash
   kubectl create configmap <name> --from-literal=key=value
   ```

2. **stringData é mais fácil que data**
   ```yaml
   stringData:
     password: secret123    # ← Texto plano
   ```
   Ao invés de:
   ```yaml
   data:
     password: c2VjcmV0MTIz    # ← Base64
   ```

3. **envFrom injeta todas as chaves**
   ```yaml
   envFrom:
   - configMapRef:
       name: app-config
   ```

4. **Base64 não é criptografia!**
   - Qualquer pessoa com acesso ao Secret pode decodificar
   - Use RBAC para proteger Secrets
   - Considere encryption at rest

5. **Testar variáveis de ambiente**
   ```bash
   kubectl exec <pod> -- env
   kubectl exec <pod> -- printenv | grep <VAR>
   ```

---

⬅️ **Anterior**: [rolling-updates-rollbacks.md](./rolling-updates-rollbacks.md) | ➡️ **Próximo**: [daemonsets.md](./daemonsets.md)
