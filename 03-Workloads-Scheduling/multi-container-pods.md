# Multi-Container Pods e Design Patterns

## 📋 O que são Multi-Container Pods?

**Multi-Container Pods** são pods que contêm **dois ou mais containers** que trabalham juntos como uma unidade lógica. Os containers compartilham o mesmo ciclo de vida, rede e volumes.

### Características principais:
- ✅ Containers **compartilham o mesmo IP** e namespace de rede
- ✅ Podem se comunicar via **localhost**
- ✅ Compartilham **volumes** (storage)
- ✅ São **agendados juntos** no mesmo nó
- ✅ Iniciam e param **juntos**
- ✅ Escalam **como uma unidade**

## 🎯 Por que usar Multi-Container Pods?

### Casos de Uso:

1. **Separação de responsabilidades**
   - Container principal: aplicação
   - Container auxiliar: funcionalidade de suporte

2. **Modularidade e reutilização**
   - Containers auxiliares podem ser reutilizados em diferentes pods
   - Ex: mesmo sidecar de logging para várias aplicações

3. **Acoplamento forte**
   - Quando dois processos precisam estar sempre juntos
   - Compartilham ciclo de vida

## 📐 Design Patterns de Multi-Container

### 1. Sidecar Pattern (Mais comum)

O container **sidecar** estende ou melhora a funcionalidade do container principal.

```
┌─────────────────────────────────────┐
│ Pod                                 │
│                                     │
│  ┌─────────────┐   ┌─────────────┐│
│  │   Main      │   │  Sidecar    ││
│  │ Container   │   │  Container  ││
│  │             │   │             ││
│  │ Web App     │   │   Logging   ││
│  │             │   │   Agent     ││
│  └─────────────┘   └─────────────┘│
│         │                 │        │
│         └────────┬────────┘        │
│         Shared Volume (logs)       │
└─────────────────────────────────────┘
```

**Exemplos:**
- Logging/monitoring agent (Fluentd, Filebeat)
- Service mesh proxy (Envoy, Linkerd)
- Configuration updater
- File sync service

**Exemplo YAML:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-logging
spec:
  containers:
  # Container principal
  - name: webapp
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx

  # Sidecar: streaming de logs
  - name: log-shipper
    image: fluent/fluentd:v1.14
    volumeMounts:
    - name: logs
      mountPath: /var/log/nginx
    env:
    - name: FLUENTD_CONF
      value: "fluent.conf"

  volumes:
  - name: logs
    emptyDir: {}
```

**Como funciona:**
1. Nginx escreve logs em `/var/log/nginx`
2. Volume `logs` é compartilhado
3. Fluentd lê os logs do volume compartilhado
4. Fluentd envia logs para destino (Elasticsearch, etc.)

### 2. Ambassador Pattern

O container **ambassador** age como um **proxy** entre o container principal e o mundo externo.

```
┌─────────────────────────────────────────┐
│ Pod                                     │
│                                         │
│  ┌──────────┐        ┌──────────────┐ │
│  │   Main   │  ←───→ │  Ambassador  │─┼──→ External Service
│  │Container │localhost│   Proxy      │ │
│  │          │        └──────────────┘ │
│  │  App     │                          │
│  └──────────┘                          │
└─────────────────────────────────────────┘
```

**Exemplos:**
- Proxy para serviços externos
- Cache local
- Circuit breaker
- Rate limiting

**Exemplo YAML:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-ambassador
spec:
  containers:
  # Container principal
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      value: "localhost"    # ← Conecta ao ambassador
    - name: DB_PORT
      value: "5432"

  # Ambassador: proxy para banco externo
  - name: db-ambassador
    image: postgres-proxy:1.0
    ports:
    - containerPort: 5432
    env:
    - name: UPSTREAM_DB_HOST
      value: "production-db.example.com"
    - name: UPSTREAM_DB_PORT
      value: "5432"
```

**Como funciona:**
1. App conecta em `localhost:5432`
2. Ambassador recebe conexão
3. Ambassador faz proxy para `production-db.example.com:5432`
4. Vantagem: trocar ambiente apenas mudando config do ambassador

### 3. Adapter Pattern

O container **adapter** transforma ou adapta dados do container principal para um formato padronizado.

```
┌─────────────────────────────────────────┐
│ Pod                                     │
│                                         │
│  ┌──────────┐        ┌──────────────┐  │
│  │   Main   │        │   Adapter    │──┼──→ Standard Format
│  │Container │  ───→  │  Container   │  │    (Prometheus metrics)
│  │          │ Custom │              │  │
│  │  App     │ Format └──────────────┘  │
│  └──────────┘                           │
└─────────────────────────────────────────┘
```

**Exemplos:**
- Converter logs para formato padrão
- Transformar métricas para Prometheus
- Normalizar dados de saída

**Exemplo YAML:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-adapter
spec:
  containers:
  # Container principal (expõe métricas em formato customizado)
  - name: app
    image: legacy-app:1.0
    ports:
    - containerPort: 8080
    volumeMounts:
    - name: metrics
      mountPath: /var/metrics

  # Adapter: converte para formato Prometheus
  - name: metrics-adapter
    image: metrics-adapter:1.0
    ports:
    - containerPort: 9090    # Prometheus endpoint
    volumeMounts:
    - name: metrics
      mountPath: /var/metrics

  volumes:
  - name: metrics
    emptyDir: {}
```

**Como funciona:**
1. App escreve métricas em formato customizado em `/var/metrics`
2. Adapter lê o arquivo
3. Adapter converte para formato Prometheus
4. Adapter expõe em `:9090/metrics` para Prometheus scrape

## 🔧 Init Containers

**Init Containers** são containers especiais que rodam **antes** dos containers principais e devem **completar com sucesso** antes dos containers principais iniciarem.

### Características:
- ✅ Rodam **sequencialmente** (um após o outro)
- ✅ Devem **completar** (exit 0) antes do próximo
- ✅ Se falharem, pod é **reiniciado** (respeitando restartPolicy)
- ✅ Úteis para **inicialização** e **setup**

### Casos de Uso:

1. **Esperar por dependências**
   - Aguardar serviço estar disponível
   - Aguardar banco de dados estar pronto

2. **Download de dados**
   - Baixar arquivos de configuração
   - Clonar repositório Git

3. **Setup de ambiente**
   - Criar diretórios
   - Definir permissões
   - Executar migrations

4. **Segurança**
   - Registrar com serviço externo
   - Obter certificados/secrets

### Exemplo: Aguardar Serviço

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-init
spec:
  # Init Container: aguarda banco estar pronto
  initContainers:
  - name: wait-for-db
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "Waiting for database..."
      until nslookup mysql.default.svc.cluster.local; do
        echo "Database not ready yet..."
        sleep 2
      done
      echo "Database is ready!"

  # Container principal: inicia apenas após init completar
  containers:
  - name: app
    image: myapp:1.0
    env:
    - name: DB_HOST
      value: "mysql.default.svc.cluster.local"
```

### Exemplo: Download de Configuração

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-config-download
spec:
  initContainers:
  # Init 1: Baixar configuração
  - name: download-config
    image: busybox:1.36
    command:
    - wget
    - "-O"
    - "/config/app-config.json"
    - "https://config-server.example.com/config.json"
    volumeMounts:
    - name: config
      mountPath: /config

  # Init 2: Validar configuração
  - name: validate-config
    image: busybox:1.36
    command:
    - sh
    - -c
    - |
      echo "Validating config..."
      if [ -f /config/app-config.json ]; then
        echo "Config file found!"
      else
        echo "Config file missing!"
        exit 1
      fi
    volumeMounts:
    - name: config
      mountPath: /config

  # Container principal
  containers:
  - name: app
    image: myapp:1.0
    volumeMounts:
    - name: config
      mountPath: /etc/app-config

  volumes:
  - name: config
    emptyDir: {}
```

## 🔄 Ordem de Execução

```
┌──────────────────────────────────────────────┐
│ Pod Lifecycle                                │
│                                              │
│  1. Init Container 1  ──→  ✓ Completo      │
│           ↓                                  │
│  2. Init Container 2  ──→  ✓ Completo      │
│           ↓                                  │
│  3. Init Container 3  ──→  ✓ Completo      │
│           ↓                                  │
│  4. Main Containers iniciam (em paralelo)   │
│     - Container A                           │
│     - Container B                           │
│     - Container C (sidecar)                 │
│                                              │
│  Se qualquer Init falhar → Pod reinicia     │
└──────────────────────────────────────────────┘
```

## 📊 Volumes Compartilhados

Containers em um pod podem compartilhar dados via volumes.

### Tipos de Volumes Comuns:

#### emptyDir (mais comum)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-volume-pod
spec:
  containers:
  - name: writer
    image: busybox
    command:
    - sh
    - -c
    - |
      while true; do
        echo "$(date): Hello from writer" >> /data/log.txt
        sleep 5
      done
    volumeMounts:
    - name: shared-data
      mountPath: /data

  - name: reader
    image: busybox
    command:
    - sh
    - -c
    - |
      tail -f /data/log.txt
    volumeMounts:
    - name: shared-data
      mountPath: /data

  volumes:
  - name: shared-data
    emptyDir: {}    # Volume temporário (deletado quando pod morre)
```

#### hostPath

```yaml
volumes:
- name: docker-socket
  hostPath:
    path: /var/run/docker.sock    # Acessa socket do Docker no host
    type: Socket
```

#### configMap / secret

```yaml
volumes:
- name: config
  configMap:
    name: app-config
- name: secrets
  secret:
    secretName: app-secret
```

## 🌐 Comunicação entre Containers

Containers no mesmo pod podem se comunicar de várias formas:

### 1. Via localhost (Network)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend-backend
spec:
  containers:
  # Backend: API
  - name: backend
    image: my-api:1.0
    ports:
    - containerPort: 8080

  # Frontend: Web UI
  - name: frontend
    image: my-frontend:1.0
    ports:
    - containerPort: 80
    env:
    - name: API_URL
      value: "http://localhost:8080"    # ← Comunica via localhost
```

### 2. Via Volume Compartilhado (Filesystem)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: producer-consumer
spec:
  containers:
  - name: producer
    image: busybox
    command: ["sh", "-c", "echo 'data' > /shared/file.txt"]
    volumeMounts:
    - name: shared
      mountPath: /shared

  - name: consumer
    image: busybox
    command: ["sh", "-c", "cat /shared/file.txt"]
    volumeMounts:
    - name: shared
      mountPath: /shared

  volumes:
  - name: shared
    emptyDir: {}
```

### 3. Via IPC (Inter-Process Communication)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ipc-pod
spec:
  shareProcessNamespace: true    # ← Permite compartilhar namespace de processos
  containers:
  - name: app1
    image: nginx
  - name: app2
    image: busybox
    command: ["sh", "-c", "ps aux"]    # Verá processos do nginx
```

## 🧪 Exemplos Práticos Completos

### Exemplo 1: Nginx + Fluentd (Sidecar)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-logging
  labels:
    app: nginx
spec:
  containers:
  # Container principal: Nginx
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: nginx-logs
      mountPath: /var/log/nginx

  # Sidecar: Fluentd para enviar logs
  - name: fluentd
    image: fluent/fluentd:v1.14
    volumeMounts:
    - name: nginx-logs
      mountPath: /var/log/nginx
      readOnly: true
    - name: fluentd-config
      mountPath: /fluentd/etc
    env:
    - name: FLUENTD_CONF
      value: "fluent.conf"

  volumes:
  - name: nginx-logs
    emptyDir: {}
  - name: fluentd-config
    configMap:
      name: fluentd-config
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentd-config
data:
  fluent.conf: |
    <source>
      @type tail
      path /var/log/nginx/access.log
      pos_file /var/log/nginx/access.log.pos
      tag nginx.access
      <parse>
        @type nginx
      </parse>
    </source>

    <match nginx.access>
      @type stdout
    </match>
```

### Exemplo 2: App + Redis Sidecar (Cache Local)

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-cache
spec:
  containers:
  # Container principal
  - name: app
    image: myapp:1.0
    ports:
    - containerPort: 8080
    env:
    - name: REDIS_HOST
      value: "localhost"    # ← Redis no mesmo pod
    - name: REDIS_PORT
      value: "6379"

  # Sidecar: Redis para cache
  - name: redis
    image: redis:7.0-alpine
    ports:
    - containerPort: 6379
    resources:
      limits:
        memory: "128Mi"
        cpu: "100m"
```

### Exemplo 3: Git Sync Sidecar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp-with-git-sync
spec:
  initContainers:
  # Init: Clone inicial
  - name: git-clone
    image: alpine/git:latest
    command:
    - git
    - clone
    - --branch=main
    - https://github.com/user/repo.git
    - /git
    volumeMounts:
    - name: git-repo
      mountPath: /git

  containers:
  # Container principal: Nginx servindo conteúdo do Git
  - name: nginx
    image: nginx:1.27
    ports:
    - containerPort: 80
    volumeMounts:
    - name: git-repo
      mountPath: /usr/share/nginx/html
      readOnly: true

  # Sidecar: Git sync (atualiza repositório a cada 60s)
  - name: git-sync
    image: k8s.gcr.io/git-sync/git-sync:v3.6.3
    env:
    - name: GIT_SYNC_REPO
      value: "https://github.com/user/repo.git"
    - name: GIT_SYNC_BRANCH
      value: "main"
    - name: GIT_SYNC_ROOT
      value: "/git"
    - name: GIT_SYNC_DEST
      value: "current"
    - name: GIT_SYNC_PERIOD
      value: "60s"
    volumeMounts:
    - name: git-repo
      mountPath: /git

  volumes:
  - name: git-repo
    emptyDir: {}
```

## 📚 Comandos Úteis

### Ver logs de containers específicos

```bash
# Logs do primeiro container (padrão)
kubectl logs <pod-name>

# Logs de um container específico
kubectl logs <pod-name> -c <container-name>

# Logs de todos os containers
kubectl logs <pod-name> --all-containers=true

# Follow logs de um container
kubectl logs <pod-name> -c <container-name> -f

# Logs do init container
kubectl logs <pod-name> -c <init-container-name>
```

### Exec em containers específicos

```bash
# Exec no primeiro container (padrão)
kubectl exec <pod-name> -- command

# Exec em container específico
kubectl exec <pod-name> -c <container-name> -- command

# Shell interativo em container específico
kubectl exec -it <pod-name> -c <container-name> -- /bin/sh
```

### Ver status dos containers

```bash
# Ver status de todos os containers
kubectl get pod <pod-name> -o jsonpath='{.status.containerStatuses[*].name}'

# Ver status detalhado
kubectl describe pod <pod-name>

# Ver init containers
kubectl get pod <pod-name> -o jsonpath='{.status.initContainerStatuses[*].name}'
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Criar pod com múltiplos containers**
   ```yaml
   spec:
     containers:
     - name: container1
       image: nginx
     - name: container2
       image: busybox
   ```

2. **Criar pod com init containers**
   ```yaml
   spec:
     initContainers:
     - name: init
       image: busybox
       command: ["sh", "-c", "sleep 10"]
     containers:
     - name: main
       image: nginx
   ```

3. **Compartilhar volume entre containers**
   ```yaml
   spec:
     containers:
     - name: c1
       volumeMounts:
       - name: shared
         mountPath: /data
     - name: c2
       volumeMounts:
       - name: shared
         mountPath: /data
     volumes:
     - name: shared
       emptyDir: {}
   ```

4. **Ver logs de container específico**
   ```bash
   kubectl logs <pod> -c <container>
   ```

5. **Exec em container específico**
   ```bash
   kubectl exec <pod> -c <container> -- command
   ```

### 🧪 Cenário típico na prova:

> **"Crie um pod chamado 'multi-app' com dois containers: 'nginx' (imagem nginx) e 'busybox' (imagem busybox que roda 'sleep 3600'). Ambos devem compartilhar um volume emptyDir montado em '/shared'."**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-app
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: shared
      mountPath: /shared

  - name: busybox
    image: busybox
    command: ["sleep", "3600"]
    volumeMounts:
    - name: shared
      mountPath: /shared

  volumes:
  - name: shared
    emptyDir: {}
```

```bash
# Aplicar
kubectl apply -f multi-app.yaml

# Verificar ambos os containers estão rodando
kubectl get pod multi-app

# Testar comunicação via volume compartilhado
kubectl exec multi-app -c nginx -- sh -c "echo 'hello from nginx' > /shared/test.txt"
kubectl exec multi-app -c busybox -- cat /shared/test.txt
```

## 💡 Dicas para a Prova

1. **Sempre especifique -c para containers específicos**
   ```bash
   kubectl logs <pod> -c <container>
   kubectl exec <pod> -c <container> -- command
   ```

2. **Init containers rodam sequencialmente**
   - Se um falha, pod não inicia
   - Útil para dependências

3. **emptyDir é temporário**
   - Deletado quando pod morre
   - Útil para compartilhamento temporário

4. **Containers compartilham IP**
   - Comunicação via localhost
   - Não podem usar mesma porta

5. **Use describe para debug**
   ```bash
   kubectl describe pod <name>
   ```
   Mostra status de init containers e main containers

---

⬅️ **Anterior**: [pods.md](./pods.md) | ➡️ **Próximo**: [resource-limits.md](./resource-limits.md)
