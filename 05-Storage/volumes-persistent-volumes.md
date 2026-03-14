# Volumes e Persistent Volumes no Kubernetes

## 📋 Índice

1. [Volumes no Kubernetes](#-volumes-no-kubernetes)
2. [Tipos de Volumes](#-tipos-de-volumes)
3. [Persistent Volumes (PV)](#-persistent-volumes-pv)
4. [Persistent Volume Claims (PVC)](#-persistent-volume-claims-pvc)
5. [Storage Classes](#-storage-classes)
6. [Container Storage Interface (CSI)](#-container-storage-interface-csi)
7. [Volume Modes](#-volume-modes)
8. [Access Modes](#-access-modes)
9. [Reclaim Policies](#-reclaim-policies)
10. [Dynamic Provisioning](#-dynamic-provisioning)
11. [Static Provisioning](#-static-provisioning)
12. [Exemplos Práticos](#-exemplos-práticos)
13. [Troubleshooting](#-troubleshooting)

## 📦 Volumes no Kubernetes

### O que são Volumes?

**Volumes** no Kubernetes permitem que containers tenham acesso a storage persistente ou compartilhado. Diferente de Docker volumes, volumes do Kubernetes têm ciclo de vida vinculado ao **Pod**, não ao container.

### Por que usar Volumes?

```
SEM VOLUMES:
┌─────────────────────────────────────────────────────────────┐
│  Pod                                                        │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Container                                             │ │
│  │  ├─ /var/lib/mysql  (dados)                          │ │
│  │  └─ Container reinicia → DADOS PERDIDOS! ❌          │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘

COM VOLUMES:
┌─────────────────────────────────────────────────────────────┐
│  Pod                                                        │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Container                                             │ │
│  │  └─ /var/lib/mysql → monta Volume                    │ │
│  └──────────────────────┬────────────────────────────────┘ │
│                         │                                   │
│  ┌──────────────────────▼────────────────────────────────┐ │
│  │ Volume (persiste enquanto Pod existir)               │ │
│  │  └─ Container reinicia → DADOS PRESERVADOS! ✅       │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Ciclo de Vida

| Tipo | Ciclo de Vida | Persistência |
|------|---------------|--------------|
| **Container filesystem** | Container | ❌ Perdido ao reiniciar container |
| **Volume (emptyDir)** | Pod | ⚠️ Perdido ao deletar pod |
| **Volume (hostPath)** | Node | ✅ Persiste no node |
| **PersistentVolume** | Cluster | ✅ Persiste independente de pods |

## 🗂️ Tipos de Volumes

### 1. emptyDir

**Diretório vazio** criado quando pod inicia. Deletado quando pod é removido.

**Use cases:**
- Cache temporário
- Compartilhar arquivos entre containers no mesmo pod
- Scratch space para computações

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-emptydir
spec:
  containers:
  - name: writer
    image: busybox
    command: ['sh', '-c', 'echo "Hello" > /data/hello.txt; sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data

  - name: reader
    image: busybox
    command: ['sh', '-c', 'cat /data/hello.txt; sleep 3600']
    volumeMounts:
    - name: shared-data
      mountPath: /data

  volumes:
  - name: shared-data
    emptyDir: {}
```

**emptyDir em memória (tmpfs):**

```yaml
volumes:
- name: cache-volume
  emptyDir:
    medium: Memory    # Usa RAM em vez de disco
    sizeLimit: 1Gi    # Limite de tamanho
```

### 2. hostPath

**Monta diretório ou arquivo do node** dentro do pod.

**⚠️ AVISO:** Não recomendado em produção! Use apenas para casos específicos.

**Use cases válidos:**
- DaemonSets que precisam acessar Docker socket (`/var/run/docker.sock`)
- Acessar logs do node (`/var/log`)
- Desenvolvimento local com Minikube/Kind

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-hostpath
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: host-volume
      mountPath: /usr/share/nginx/html

  volumes:
  - name: host-volume
    hostPath:
      path: /data/nginx    # Diretório no node
      type: DirectoryOrCreate
```

**Tipos de hostPath:**

| Type | Descrição |
|------|-----------|
| `DirectoryOrCreate` | Cria diretório se não existir |
| `Directory` | Deve existir, é um diretório |
| `FileOrCreate` | Cria arquivo se não existir |
| `File` | Deve existir, é um arquivo |
| `Socket` | Unix socket (ex: Docker socket) |
| `CharDevice` | Character device |
| `BlockDevice` | Block device |

### 3. configMap

**Monta ConfigMap** como volume (arquivos read-only).

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  config.json: |
    {
      "database": "postgres",
      "host": "db.example.com"
    }
  app.conf: |
    debug=true
    port=8080
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: config
      mountPath: /etc/config

  volumes:
  - name: config
    configMap:
      name: app-config
      # Resultado:
      # /etc/config/config.json
      # /etc/config/app.conf
```

**Montar apenas chaves específicas:**

```yaml
volumes:
- name: config
  configMap:
    name: app-config
    items:
    - key: config.json
      path: application.json    # Renomear arquivo
```

### 4. secret

**Monta Secret** como volume (similar a ConfigMap).

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-credentials
type: Opaque
data:
  username: YWRtaW4=       # base64("admin")
  password: cGFzc3dvcmQ=   # base64("password")
---
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: credentials
      mountPath: /etc/secrets
      readOnly: true

  volumes:
  - name: credentials
    secret:
      secretName: db-credentials
      # Resultado:
      # /etc/secrets/username (conteúdo: "admin")
      # /etc/secrets/password (conteúdo: "password")
```

### 5. persistentVolumeClaim

**Referencia um PersistentVolumeClaim** (mais detalhes abaixo).

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: myapp
    volumeMounts:
    - name: storage
      mountPath: /data

  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: my-pvc
```

### 6. nfs

**Monta volume NFS** diretamente.

```yaml
volumes:
- name: nfs-volume
  nfs:
    server: 192.168.1.100
    path: /exported/path
    readOnly: false
```

### 7. downwardAPI

**Expõe informações do pod** como arquivos.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: downward-api-pod
  labels:
    app: myapp
    env: production
spec:
  containers:
  - name: app
    image: busybox
    command: ['sh', '-c', 'cat /etc/podinfo/labels; sleep 3600']
    volumeMounts:
    - name: podinfo
      mountPath: /etc/podinfo

  volumes:
  - name: podinfo
    downwardAPI:
      items:
      - path: labels
        fieldRef:
          fieldPath: metadata.labels
      - path: annotations
        fieldRef:
          fieldPath: metadata.annotations
      - path: pod-name
        fieldRef:
          fieldPath: metadata.name
      - path: namespace
        fieldRef:
          fieldPath: metadata.namespace
```

### 8. projected

**Combina múltiplas fontes** em um único volume.

```yaml
volumes:
- name: all-in-one
  projected:
    sources:
    - secret:
        name: db-secret
    - configMap:
        name: app-config
    - downwardAPI:
        items:
        - path: pod-name
          fieldRef:
            fieldPath: metadata.name
```

## 💾 Persistent Volumes (PV)

### O que é um PersistentVolume?

**PersistentVolume (PV)** é um recurso de storage no cluster provisionado por um administrador ou dinamicamente via Storage Classes.

### Características:

- ✅ Independente do ciclo de vida de pods
- ✅ Pode ser provisionado estaticamente (admin) ou dinamicamente (StorageClass)
- ✅ Pode ser compartilhado entre pods
- ✅ Suporta diferentes backends (NFS, iSCSI, cloud disks, etc.)

### Exemplo de PersistentVolume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

### Campos Importantes

| Campo | Descrição |
|-------|-----------|
| `capacity.storage` | Tamanho do volume |
| `volumeMode` | `Filesystem` ou `Block` |
| `accessModes` | `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany` |
| `persistentVolumeReclaimPolicy` | `Retain`, `Recycle`, `Delete` |
| `storageClassName` | Classe de storage (para binding) |

## 📋 Persistent Volume Claims (PVC)

### O que é um PersistentVolumeClaim?

**PersistentVolumeClaim (PVC)** é uma **solicitação** de storage por um usuário. É como um "pedido" que será atendido por um PV disponível.

### PV vs PVC

```
┌─────────────────────────────────────────────────────────────┐
│  ADMINISTRADOR                                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ PersistentVolume (PV)                                 │ │
│  │  - Provisiona storage real                            │ │
│  │  - Define capacidade, access modes, backend           │ │
│  │  - "Oferta" de storage                                │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ Binding (automático)
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  DESENVOLVEDOR                                              │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ PersistentVolumeClaim (PVC)                           │ │
│  │  - Solicita storage                                   │ │
│  │  - Define quanto precisa e como quer acessar          │ │
│  │  - "Pedido" de storage                                │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

### Exemplo de PersistentVolumeClaim

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 5Gi
  storageClassName: nfs
```

### Binding Process

Kubernetes faz binding de PVC para PV baseado em:

1. **StorageClassName** (deve ser igual ou vazio)
2. **Access Modes** (PV deve suportar o que PVC pede)
3. **Capacity** (PV deve ter >= storage que PVC pede)

**Estados de um PVC:**

| Estado | Descrição |
|--------|-----------|
| `Pending` | Aguardando PV disponível |
| `Bound` | Vinculado a um PV |
| `Lost` | PV foi deletado mas PVC ainda existe |

**Estados de um PV:**

| Estado | Descrição |
|--------|-----------|
| `Available` | Disponível para binding |
| `Bound` | Vinculado a um PVC |
| `Released` | PVC foi deletado, mas ainda não foi reciclado |
| `Failed` | Erro ao reciclar |

## 🏷️ Storage Classes

### O que é uma StorageClass?

**StorageClass** define **tipos de storage** disponíveis no cluster e permite **provisionamento dinâmico** de PVs.

### Exemplo de StorageClass

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
reclaimPolicy: Delete
```

### Campos Importantes

| Campo | Descrição |
|-------|-----------|
| `provisioner` | CSI driver ou in-tree provisioner |
| `parameters` | Parâmetros específicos do provisioner |
| `volumeBindingMode` | `Immediate` ou `WaitForFirstConsumer` |
| `allowVolumeExpansion` | Permite expandir volumes |
| `reclaimPolicy` | `Delete` ou `Retain` |

### Volume Binding Modes

**1. Immediate (padrão):**
- PV é provisionado **imediatamente** quando PVC é criado
- Pode provisionar em zona errada (pod não consegue agendar)

**2. WaitForFirstConsumer (recomendado):**
- PV é provisionado apenas quando **primeiro pod usando o PVC é agendado**
- Garante que volume está na mesma zona do pod

```yaml
volumeBindingMode: WaitForFirstConsumer
```

### StorageClass Padrão

```bash
# Ver StorageClasses
kubectl get storageclass

# Marcar como padrão
kubectl patch storageclass <name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

# Remover padrão
kubectl patch storageclass <name> -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

## 🔌 Container Storage Interface (CSI)

### O que é CSI?

**Container Storage Interface (CSI)** é uma especificação padrão para expor sistemas de storage para workloads containerizadas.

### Por que CSI?

**Antes do CSI (In-tree volumes):**
- ❌ Drivers de storage dentro do código do Kubernetes
- ❌ Difícil adicionar/atualizar drivers
- ❌ Lançamento vinculado ao release do Kubernetes
- ❌ Código vendor-specific no core do Kubernetes

**Com CSI (Out-of-tree volumes):**
- ✅ Drivers de storage fora do código do Kubernetes
- ✅ Fácil adicionar/atualizar drivers
- ✅ Vendors podem lançar drivers independentemente
- ✅ Kubernetes core fica mais limpo

### Arquitetura CSI

```
┌─────────────────────────────────────────────────────────────┐
│  KUBERNETES CLUSTER                                         │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Kubernetes Control Plane                              │ │
│  │  ├─ kube-apiserver                                    │ │
│  │  ├─ kube-controller-manager                           │ │
│  │  │   └─ PersistentVolumeController                    │ │
│  │  └─ CSI Controller Plugins (DaemonSet/Deployment)     │ │
│  │       ├─ csi-provisioner (sidecar)                    │ │
│  │       ├─ csi-attacher (sidecar)                       │ │
│  │       ├─ csi-resizer (sidecar)                        │ │
│  │       ├─ csi-snapshotter (sidecar)                    │ │
│  │       └─ CSI Driver (vendor)                          │ │
│  └───────────────────────────────────────────────────────┘ │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ Worker Nodes                                          │ │
│  │  ├─ kubelet                                           │ │
│  │  └─ CSI Node Plugin (DaemonSet)                       │ │
│  │       ├─ node-driver-registrar (sidecar)              │ │
│  │       └─ CSI Driver (vendor)                          │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
                          │
                          │ gRPC
                          ↓
┌─────────────────────────────────────────────────────────────┐
│  STORAGE BACKEND                                            │
│  - AWS EBS, Azure Disk, GCP PD                             │
│  - NFS, iSCSI, Ceph, GlusterFS                             │
│  - Portworx, NetApp, Pure Storage, etc.                    │
└─────────────────────────────────────────────────────────────┘
```

### CSI Driver Components

#### 1. Controller Plugin (Control Plane)

Roda no control plane (Deployment ou DaemonSet).

**Responsabilidades:**
- **Provisioning**: Criar/deletar volumes
- **Attaching**: Atachar/detachar volumes de nodes
- **Snapshotting**: Criar/deletar snapshots
- **Resizing**: Expandir volumes

**Sidecars:**
- `csi-provisioner`: Provisiona volumes dinamicamente
- `csi-attacher`: Faz attach/detach de volumes
- `csi-resizer`: Expande volumes
- `csi-snapshotter`: Cria snapshots

#### 2. Node Plugin (Worker Nodes)

Roda em todos os nodes (DaemonSet).

**Responsabilidades:**
- **Mounting**: Montar volumes em pods
- **Unmounting**: Desmontar volumes
- **Volume health checks**

**Sidecars:**
- `node-driver-registrar`: Registra driver com kubelet

### CSI Drivers Populares

| Driver | Storage Backend | Cloud |
|--------|----------------|-------|
| **AWS EBS CSI** | Amazon Elastic Block Store | AWS |
| **Azure Disk CSI** | Azure Managed Disks | Azure |
| **GCE PD CSI** | Google Persistent Disk | GCP |
| **NFS CSI** | Network File System | Any |
| **Ceph CSI** | Ceph RBD / CephFS | Any |
| **Longhorn** | Distributed block storage | Any |
| **Portworx** | Enterprise storage platform | Any |
| **OpenEBS** | Container-native storage | Any |
| **NetApp Trident** | NetApp storage | Any |
| **Dell CSI** | Dell EMC storage | Any |

### Instalar CSI Driver (AWS EBS exemplo)

```bash
# 1. Adicionar repo Helm
helm repo add aws-ebs-csi-driver https://kubernetes-sigs.github.io/aws-ebs-csi-driver
helm repo update

# 2. Instalar driver
helm install aws-ebs-csi-driver aws-ebs-csi-driver/aws-ebs-csi-driver \
  --namespace kube-system

# 3. Verificar instalação
kubectl get pods -n kube-system -l app.kubernetes.io/name=aws-ebs-csi-driver

# 4. Verificar CSIDriver object
kubectl get csidriver
# Output:
# NAME              ATTACHREQUIRED   PODINFOONMOUNT   MODES        AGE
# ebs.csi.aws.com   true             false            Persistent   1m
```

### Criar StorageClass com CSI

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
provisioner: ebs.csi.aws.com    # CSI driver
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  fsType: ext4
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### CSI Features

| Feature | Descrição | Support |
|---------|-----------|---------|
| **Dynamic Provisioning** | Criar volumes automaticamente | ✅ Todos |
| **Volume Snapshots** | Criar snapshots de volumes | ⚠️ Depende do driver |
| **Volume Cloning** | Clonar volumes existentes | ⚠️ Depende do driver |
| **Volume Expansion** | Expandir volumes online | ⚠️ Depende do driver |
| **Topology** | Awareness de zona/região | ⚠️ Depende do driver |
| **Volume Health** | Monitorar saúde do volume | ⚠️ Depende do driver |

### Exemplo de Snapshot

```yaml
# 1. VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: csi-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Delete

---
# 2. VolumeSnapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: my-snapshot
spec:
  volumeSnapshotClassName: csi-snapshot-class
  source:
    persistentVolumeClaimName: my-pvc

---
# 3. Restaurar de snapshot
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: restored-pvc
spec:
  dataSource:
    name: my-snapshot
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

## 🔒 Access Modes

**Access Modes** definem como o volume pode ser montado.

| Access Mode | Abreviação | Descrição |
|-------------|------------|-----------|
| **ReadWriteOnce** | RWO | Leitura/escrita por **1 node** apenas |
| **ReadOnlyMany** | ROX | Leitura por **múltiplos nodes** |
| **ReadWriteMany** | RWX | Leitura/escrita por **múltiplos nodes** |
| **ReadWriteOncePod** | RWOP | Leitura/escrita por **1 pod** apenas (>= K8s 1.22) |

### Suporte por Storage Backend

| Storage | RWO | ROX | RWX |
|---------|-----|-----|-----|
| **AWS EBS** | ✅ | ✅ | ❌ |
| **Azure Disk** | ✅ | ✅ | ❌ |
| **GCE PD** | ✅ | ✅ | ❌ |
| **NFS** | ✅ | ✅ | ✅ |
| **CephFS** | ✅ | ✅ | ✅ |
| **GlusterFS** | ✅ | ✅ | ✅ |
| **Portworx** | ✅ | ✅ | ✅ |
| **hostPath** | ✅ | ✅ | ✅ |

## 🔄 Reclaim Policies

**Reclaim Policy** define o que acontece com o PV quando o PVC é deletado.

| Policy | Descrição |
|--------|-----------|
| **Retain** | PV **não é deletado**, fica `Released` |
| **Delete** | PV **é deletado** automaticamente |
| **Recycle** (deprecated) | Dados são limpos (`rm -rf`), PV volta a `Available` |

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-retain
spec:
  capacity:
    storage: 10Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain    # <--
  storageClassName: standard
  hostPath:
    path: /data/pv
```

### Comportamento de cada Policy

#### 1. Retain (Mais Seguro)

```bash
# 1. Criar PVC
kubectl apply -f pvc.yaml

# 2. PV é bound
kubectl get pv
# NAME   STATUS   CLAIM       STORAGECLASS   REASON   AGE
# pv-1   Bound    default/pvc standard                1m

# 3. Deletar PVC
kubectl delete pvc pvc

# 4. PV fica Released (não deletado!)
kubectl get pv
# NAME   STATUS     CLAIM       STORAGECLASS   REASON   AGE
# pv-1   Released   default/pvc standard                5m

# 5. Dados ainda estão no storage backend
# Admin precisa decidir: recuperar ou deletar manualmente
```

#### 2. Delete (Padrão para provisionamento dinâmico)

```bash
# 1. Criar PVC com StorageClass que provisiona dinamicamente
kubectl apply -f pvc.yaml

# 2. PV é criado automaticamente e bound
kubectl get pv

# 3. Deletar PVC
kubectl delete pvc pvc

# 4. PV é deletado automaticamente
kubectl get pv
# No resources found

# 5. Storage backend também é deletado (AWS EBS, Azure Disk, etc.)
```

## 📊 Volume Modes

**Volume Mode** define como o volume é apresentado ao pod.

| Mode | Descrição |
|------|-----------|
| **Filesystem** (padrão) | Volume montado como filesystem (ext4, xfs, etc.) |
| **Block** | Volume apresentado como raw block device |

### Filesystem Mode (padrão)

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-fs
spec:
  capacity:
    storage: 10Gi
  volumeMode: Filesystem    # <-- padrão
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /data/pv
```

**Pod usando Filesystem:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-fs
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: storage
      mountPath: /data    # Montado como diretório
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-fs
```

### Block Mode

**Usado para:** Bancos de dados que querem acesso direto ao dispositivo (sem overhead de filesystem).

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-block
spec:
  capacity:
    storage: 10Gi
  volumeMode: Block    # <-- raw block device
  accessModes:
  - ReadWriteOnce
  hostPath:
    path: /dev/sdb
```

**Pod usando Block:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-block
spec:
  containers:
  - name: database
    image: postgres
    volumeDevices:    # NÃO é volumeMounts!
    - name: storage
      devicePath: /dev/xvda    # Device path dentro do container
  volumes:
  - name: storage
    persistentVolumeClaim:
      claimName: pvc-block
```

## 🚀 Dynamic Provisioning

**Dynamic Provisioning** cria PVs automaticamente quando um PVC é criado.

### Como Funciona

```
1. Desenvolvedor cria PVC
┌─────────────────────────────────────────────────────────────┐
│  kubectl apply -f pvc.yaml                                  │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ PVC                                                   │ │
│  │  storageClassName: fast-ssd                           │ │
│  │  storage: 10Gi                                        │ │
│  └───────────────────────────────────────────────────────┘ │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
2. Kubernetes encontra StorageClass "fast-ssd"
┌─────────────────────────────────────────────────────────────┐
│  StorageClass: fast-ssd                                     │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ provisioner: ebs.csi.aws.com                          │ │
│  │ parameters: type=gp3, iops=3000                       │ │
│  └───────────────────────────────────────────────────────┘ │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
3. CSI Driver provisiona storage real
┌─────────────────────────────────────────────────────────────┐
│  AWS API                                                    │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ CreateVolume(size=10Gi, type=gp3, iops=3000)          │ │
│  └───────────────────────────────────────────────────────┘ │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
4. PV é criado automaticamente
┌─────────────────────────────────────────────────────────────┐
│  PersistentVolume (criado automaticamente)                  │
│  ┌───────────────────────────────────────────────────────┐ │
│  │ capacity: 10Gi                                        │ │
│  │ awsElasticBlockStore: vol-xxxxx                       │ │
│  └───────────────────────────────────────────────────────┘ │
└───────────────────────┬─────────────────────────────────────┘
                        │
                        ↓
5. PVC é bound ao PV
┌─────────────────────────────────────────────────────────────┐
│  PVC → Bound → PV                                           │
└─────────────────────────────────────────────────────────────┘
```

### Exemplo Completo

**1. StorageClass:**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

**2. PVC (sem especificar PV):**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: fast-ssd    # Usa StorageClass
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

**3. PV é criado automaticamente:**

```bash
kubectl apply -f pvc.yaml

# Aguardar provisionamento
kubectl get pvc
# NAME     STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc   Bound    pvc-12345678-1234-1234-1234-123456789012   10Gi       RWO            fast-ssd       30s

kubectl get pv
# NAME                                       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM            STORAGECLASS   REASON   AGE
# pvc-12345678-1234-1234-1234-123456789012   10Gi       RWO            Delete           Bound    default/my-pvc   fast-ssd                1m
```

## 📝 Static Provisioning

**Static Provisioning**: Admin cria PV manualmente, desenvolvedor cria PVC que faz bind ao PV.

### Exemplo Completo

**1. Admin cria PV:**

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-nfs-static
spec:
  capacity:
    storage: 20Gi
  volumeMode: Filesystem
  accessModes:
  - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: nfs-manual
  nfs:
    server: 192.168.1.100
    path: /exports/data
```

```bash
kubectl apply -f pv.yaml
kubectl get pv
# NAME            CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS      CLAIM   STORAGECLASS   REASON   AGE
# pv-nfs-static   20Gi       RWX            Retain           Available           nfs-manual              10s
```

**2. Desenvolvedor cria PVC:**

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-nfs
spec:
  storageClassName: nfs-manual    # Mesmo storageClassName do PV
  accessModes:
  - ReadWriteMany                 # Compatível com PV
  resources:
    requests:
      storage: 10Gi               # <= capacidade do PV
```

```bash
kubectl apply -f pvc.yaml

# PVC faz bind ao PV automaticamente
kubectl get pvc
# NAME      STATUS   VOLUME          CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# pvc-nfs   Bound    pv-nfs-static   20Gi       RWX            nfs-manual     5s
```

**3. Pod usa PVC:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:
  - name: app
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /data
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: pvc-nfs
```

## 📚 Exemplos Práticos

### Exemplo 1: PostgreSQL com PVC

```yaml
# 1. PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: fast-ssd

---
# 2. Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:14
        env:
        - name: POSTGRES_PASSWORD
          value: password
        volumeMounts:
        - name: data
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: postgres-pvc
```

### Exemplo 2: Expandir Volume

```yaml
# 1. StorageClass com allowVolumeExpansion
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: expandable
provisioner: ebs.csi.aws.com
allowVolumeExpansion: true    # <-- Permite expansão

---
# 2. PVC inicial (10Gi)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
  storageClassName: expandable
```

```bash
# 3. Expandir de 10Gi para 20Gi
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# 4. Verificar expansão
kubectl get pvc my-pvc
# NAME     STATUS   VOLUME     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# my-pvc   Bound    pvc-xxx    20Gi       RWO            expandable     5m

# 5. Para alguns drivers, é necessário reiniciar pod
kubectl rollout restart deployment my-app
```

### Exemplo 3: Shared Storage (NFS)

```yaml
# 1. PV NFS
apiVersion: v1
kind: PersistentVolume
metadata:
  name: nfs-shared
spec:
  capacity:
    storage: 50Gi
  accessModes:
  - ReadWriteMany    # Múltiplos pods podem escrever
  nfs:
    server: 192.168.1.100
    path: /shared

---
# 2. PVC
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-pvc
spec:
  accessModes:
  - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  volumeName: nfs-shared    # Bind explícito

---
# 3. Deployment com múltiplos pods compartilhando volume
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3    # 3 pods compartilhando mesmo volume!
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx
        volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
      volumes:
      - name: shared-data
        persistentVolumeClaim:
          claimName: shared-pvc
```

## 🐛 Troubleshooting

### Problema 1: PVC fica em Pending

**Sintomas:**
```bash
kubectl get pvc
# NAME   STATUS    VOLUME   CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# pvc    Pending                                      fast-ssd       5m
```

**Causas possíveis:**

#### 1. Nenhum PV disponível (static provisioning)

```bash
# Ver detalhes
kubectl describe pvc pvc

# Output:
# Events:
#   Type     Reason              Age   From                         Message
#   ----     ------              ----  ----                         -------
#   Warning  ProvisioningFailed  1m    persistentvolume-controller  no persistent volumes available

# Solução: Criar PV compatível
```

#### 2. StorageClass não existe

```bash
kubectl describe pvc pvc

# Output:
# Events:
#   Warning  ProvisioningFailed  1m  persistentvolume-controller  storageclass.storage.k8s.io "fast-ssd" not found

# Verificar StorageClasses
kubectl get storageclass

# Solução: Criar StorageClass ou usar uma existente
```

#### 3. CSI Driver não instalado

```bash
kubectl describe pvc pvc

# Output:
# Events:
#   Warning  ProvisioningFailed  1m  ebs.csi.aws.com  driver ebs.csi.aws.com not found

# Verificar CSI Driver
kubectl get csidriver

# Solução: Instalar CSI driver
```

#### 4. WaitForFirstConsumer (esperando pod)

```bash
kubectl describe pvc pvc

# Output:
# Events:
#   Normal  WaitForFirstConsumer  1m  persistentvolume-controller  waiting for first consumer to be created

# Isso é NORMAL! PVC ficará Pending até primeiro pod ser criado
# Solução: Criar pod que usa o PVC
```

### Problema 2: Pod não consegue montar volume

**Sintomas:**
```bash
kubectl get pod myapp
# NAME    READY   STATUS              RESTARTS   AGE
# myapp   0/1     ContainerCreating   0          5m
```

**Diagnosticar:**

```bash
kubectl describe pod myapp

# Events:
#   Warning  FailedMount  1m  kubelet  Unable to attach or mount volumes
```

**Causas possíveis:**

#### 1. Access Mode incompatível

```bash
# PV tem RWO, mas já está bound em outro node
# Solução: Usar RWX ou mover pod para mesmo node
```

#### 2. Volume ainda attached em outro node

```bash
# Ver VolumeAttachment
kubectl get volumeattachment

# Solução: Deletar pod antigo completamente
kubectl delete pod old-pod --grace-period=0 --force
```

#### 3. Permissões de arquivo

```bash
# Logs do kubelet
journalctl -u kubelet | grep -i "mount\|volume"

# Ver dentro do node
ls -la /var/lib/kubelet/pods/<pod-uid>/volumes/

# Solução: Ajustar fsGroup ou runAsUser no pod
```

### Problema 3: Volume não é deletado (Retain policy)

```bash
# Deletar PVC
kubectl delete pvc my-pvc

# PV fica Released
kubectl get pv
# NAME   STATUS     CLAIM          RECLAIM POLICY   STORAGECLASS   AGE
# pv-1   Released   default/mypvc  Retain           standard       10m
```

**Solução 1: Deletar PV manualmente**

```bash
kubectl delete pv pv-1
```

**Solução 2: Reciclar PV para reutilizar**

```bash
# 1. Editar PV e remover claimRef
kubectl edit pv pv-1

# Deletar seção:
# claimRef:
#   apiVersion: v1
#   kind: PersistentVolumeClaim
#   name: my-pvc
#   namespace: default
#   uid: xxxxx

# 2. Limpar dados no storage backend (se necessário)

# 3. PV volta para Available
kubectl get pv
# NAME   STATUS      CLAIM   RECLAIM POLICY   STORAGECLASS   AGE
# pv-1   Available           Retain           standard       15m
```

### Problema 4: Expansão de volume falha

```bash
# Tentar expandir
kubectl patch pvc my-pvc -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# PVC mostra FileSystemResizePending
kubectl get pvc my-pvc
# Conditions:
#   Type                      Status  Reason
#   ----                      ------  ------
#   FileSystemResizePending   True    WaitingForFileSystemResize
```

**Causas:**

#### 1. StorageClass não permite expansão

```bash
kubectl get storageclass <name> -o yaml

# Verificar:
# allowVolumeExpansion: false

# Solução: Não pode expandir com essa StorageClass
# Criar novo PVC com StorageClass que permite expansão
```

#### 2. Driver não suporta expansão online

```bash
# Precisa reiniciar pod
kubectl rollout restart deployment my-app

# Ou deletar e recriar pod
kubectl delete pod my-pod
```

### Problema 5: Performance ruim

**Sintomas:**
- I/O lento
- Latência alta
- Throughput baixo

**Diagnóstico:**

```bash
# 1. Ver metrics do volume (se CSI driver suporta)
kubectl get volumemetrics

# 2. Dentro do pod, testar I/O
kubectl exec -it my-pod -- bash
dd if=/dev/zero of=/data/testfile bs=1M count=1000

# 3. Ver tipo de disco usado
kubectl describe pv <pv-name>

# 4. Ver se há throttling (cloud providers)
# AWS: CloudWatch metrics de EBS
# Azure: Azure Monitor
```

**Soluções:**

```bash
# 1. Usar StorageClass com melhor performance
# Exemplo AWS: gp2 → gp3 ou io2

# 2. Aumentar IOPS/throughput
kubectl patch storageclass fast-ssd -p '{"parameters":{"iops":"10000"}}'

# 3. Usar local storage para performance máxima
# (mas sem HA)
```

## 💡 Dicas para a Prova CKA

### Comandos Rápidos

```bash
# === VOLUMES ===

# Ver PVs
kubectl get pv

# Ver PVCs
kubectl get pvc

# Ver detalhes de PV
kubectl describe pv <name>

# Ver detalhes de PVC
kubectl describe pvc <name>

# Ver StorageClasses
kubectl get storageclass
kubectl get sc

# === CRIAR RECURSOS ===

# Criar PVC rapidamente
kubectl create -f - <<EOF
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
EOF

# === TROUBLESHOOTING ===

# Ver eventos de PVC
kubectl describe pvc <name>

# Ver por que pod não sobe
kubectl describe pod <name>

# Ver volumes montados em pod
kubectl describe pod <name> | grep -A 10 Volumes

# Ver CSI Drivers instalados
kubectl get csidriver

# === EXPANSÃO ===

# Expandir PVC
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"10Gi"}}}}'

# === DELETAR ===

# Deletar PVC
kubectl delete pvc <name>

# Deletar PV
kubectl delete pv <name>

# Forçar deleção de PVC travado
kubectl patch pvc <name> -p '{"metadata":{"finalizers":null}}'
```

### Pontos Importantes

1. **PV vs PVC:**
   - PV = Oferta (admin provisiona)
   - PVC = Pedido (dev solicita)

2. **Access Modes:**
   - RWO = 1 node
   - ROX = N nodes (read-only)
   - RWX = N nodes (read-write)

3. **Reclaim Policy:**
   - Retain = PV não é deletado
   - Delete = PV é deletado automaticamente

4. **Dynamic vs Static:**
   - Dynamic = StorageClass provisiona automaticamente
   - Static = Admin cria PV manualmente

5. **CSI é o futuro:**
   - Drivers out-of-tree
   - Vendors independentes do Kubernetes
   - Features avançadas (snapshots, cloning, expansion)

## 🔗 Recursos Úteis

### Documentação Oficial
- 📖 [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
- 📖 [Persistent Volumes](https://kubernetes.io/docs/concepts/storage/persistent-volumes/)
- 📖 [Storage Classes](https://kubernetes.io/docs/concepts/storage/storage-classes/)
- 📖 [Container Storage Interface (CSI)](https://kubernetes-csi.github.io/docs/)
- 📖 [Volume Snapshots](https://kubernetes.io/docs/concepts/storage/volume-snapshots/)

---

⬅️ **Anterior**: [../Workloads/](../Workloads/) | ➡️ **Próximo**: [../06-Troubleshooting/](../06-Troubleshooting/)
