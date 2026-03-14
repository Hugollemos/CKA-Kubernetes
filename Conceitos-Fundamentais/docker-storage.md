# Docker Storage

## 📋 O que é Docker Storage?

**Docker Storage** refere-se ao sistema de armazenamento usado pelo Docker para gerenciar dados de containers e imagens. Entender storage é fundamental para trabalhar com containers e Kubernetes.

## 🗂️ Arquitetura de Storage do Docker

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Host                             │
│                                                             │
│  ┌───────────────────────────────────────────────────────┐ │
│  │              /var/lib/docker/                         │ │
│  │                                                        │ │
│  │  ├── containers/     # Dados de containers          │ │
│  │  ├── image/          # Camadas de imagens           │ │
│  │  ├── volumes/        # Docker volumes               │ │
│  │  ├── overlay2/       # Storage driver (overlay2)    │ │
│  │  ├── buildkit/       # Build cache                  │ │
│  │  └── network/        # Configuração de rede         │ │
│  └───────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

## 📦 Tipos de Storage no Docker

### 1. Volumes (Recomendado)

**Volumes** são o mecanismo preferido para persistir dados no Docker.

**Características:**
- ✅ Gerenciados pelo Docker
- ✅ Independentes do ciclo de vida do container
- ✅ Podem ser compartilhados entre containers
- ✅ Fáceis de fazer backup
- ✅ Funcionam em Linux e Windows

**Localização:** `/var/lib/docker/volumes/`

**Criar e usar volume:**

```bash
# Criar volume
docker volume create my-volume

# Listar volumes
docker volume ls

# Inspecionar volume
docker volume inspect my-volume

# Usar volume em container
docker run -d \
  --name nginx \
  -v my-volume:/usr/share/nginx/html \
  nginx

# Volume anônimo (criado automaticamente)
docker run -d \
  --name nginx \
  -v /usr/share/nginx/html \
  nginx

# Remover volume
docker volume rm my-volume

# Remover volumes não usados
docker volume prune
```

**Ver dados do volume:**

```bash
# Volumes ficam em:
sudo ls -l /var/lib/docker/volumes/

# Ver dados de um volume específico
sudo ls -l /var/lib/docker/volumes/my-volume/_data/
```

### 2. Bind Mounts

**Bind mounts** mapeiam um diretório do host para dentro do container.

**Características:**
- ⚠️ Dependem da estrutura de diretórios do host
- ✅ Performance alta
- ✅ Acesso direto aos arquivos do host
- ❌ Menos portável que volumes
- ⚠️ Processos fora do Docker podem modificar

**Uso:**

```bash
# Sintaxe curta (-v)
docker run -d \
  --name nginx \
  -v /home/user/html:/usr/share/nginx/html \
  nginx

# Sintaxe longa (--mount) - recomendada
docker run -d \
  --name nginx \
  --mount type=bind,source=/home/user/html,target=/usr/share/nginx/html \
  nginx

# Read-only bind mount
docker run -d \
  --name nginx \
  -v /home/user/html:/usr/share/nginx/html:ro \
  nginx
```

### 3. tmpfs Mounts (Apenas Linux)

**tmpfs mounts** armazenam dados em memória (RAM), não em disco.

**Características:**
- ✅ Muito rápido (memória RAM)
- ⚠️ Dados perdidos quando container para
- ✅ Ideal para dados temporários/sensíveis
- ❌ Só funciona em Linux

**Uso:**

```bash
# Criar tmpfs mount
docker run -d \
  --name nginx \
  --mount type=tmpfs,target=/app/cache \
  nginx

# Com opções de tamanho
docker run -d \
  --name nginx \
  --tmpfs /app/cache:rw,size=100m \
  nginx
```

## 📊 Comparação de Tipos de Storage

| Tipo | Gerenciado por | Persistência | Performance | Portabilidade | Uso |
|------|---------------|--------------|-------------|---------------|-----|
| **Volume** | Docker | ✅ Alta | Alta | ✅ Alta | **Produção** |
| **Bind Mount** | Host | ✅ Alta | Muito Alta | ⚠️ Baixa | **Desenvolvimento** |
| **tmpfs** | Memória | ❌ Temporária | Muito Alta | ⚠️ Baixa | **Cache/Temp** |

## 🔄 Layered Filesystem (Sistema de Camadas)

Docker usa um **sistema de arquivos em camadas** para construir imagens.

### Como Funciona

```
┌─────────────────────────────────────────────────────────────┐
│                 Container (Read-Write Layer)                │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Container Layer (Read-Write)                         │  │
│  │ - Arquivos modificados                               │  │
│  │ - Arquivos novos                                     │  │
│  │ - Arquivos deletados (whiteout)                      │  │
│  └──────────────────────────────────────────────────────┘  │
│                          ↑                                  │
│                          │ CoW (Copy-on-Write)              │
│                          │                                  │
│  ┌──────────────────────────────────────────────────────┐  │
│  │ Image Layers (Read-Only)                            │  │
│  │                                                      │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │ Layer 3: CMD ["nginx", "-g", "daemon off;"]   │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │ Layer 2: COPY ./app /usr/share/nginx/html     │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │ Layer 1: RUN apt-get update && apt-get ...    │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  │  ┌────────────────────────────────────────────────┐ │  │
│  │  │ Layer 0: FROM ubuntu:20.04 (Base Image)        │ │  │
│  │  └────────────────────────────────────────────────┘ │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

### Copy-on-Write (CoW)

**Copy-on-Write** é a estratégia que Docker usa para economizar espaço:

1. **Leitura**: Container lê diretamente das camadas da imagem (read-only)
2. **Escrita**:
   - Se arquivo existe na imagem: copia para layer do container, depois modifica
   - Se arquivo é novo: escreve direto na layer do container

**Exemplo:**

```bash
# Dockerfile
FROM ubuntu:20.04
RUN apt-get update
RUN apt-get install -y nginx
COPY ./app /var/www/html
CMD ["nginx", "-g", "daemon off;"]

# Cada comando RUN/COPY/ADD cria uma nova layer
```

**Ver layers de uma imagem:**

```bash
# Ver histórico de layers
docker history nginx:latest

# Ver detalhes das layers
docker inspect nginx:latest
```

## 🗄️ Storage Drivers

**Storage Drivers** controlam como as camadas de imagens e containers são armazenadas.

### Drivers Comuns

| Driver | Sistema | Uso | Performance |
|--------|---------|-----|-------------|
| **overlay2** | Linux (moderno) | ✅ **Recomendado** | ⚡ Muito Alta |
| **aufs** | Linux (antigo) | ⚠️ Legado | 🐌 Média |
| **devicemapper** | Linux (Red Hat) | ⚠️ Legado | 🐌 Baixa |
| **btrfs** | Linux (Btrfs) | ⚠️ Específico | ⚡ Alta |
| **zfs** | Linux (ZFS) | ⚠️ Específico | ⚡ Alta |
| **windowsfilter** | Windows | ✅ Padrão Windows | ⚡ Alta |

### Ver Storage Driver Atual

```bash
# Ver informações do Docker
docker info | grep "Storage Driver"

# Output:
# Storage Driver: overlay2
```

### Configurar Storage Driver

Editar `/etc/docker/daemon.json`:

```json
{
  "storage-driver": "overlay2"
}
```

Depois reiniciar Docker:

```bash
sudo systemctl restart docker
```

## 🔌 Volume Driver Plugins

**Volume Driver Plugins** permitem usar diferentes backends de storage para Docker volumes, indo além do storage local padrão.

### O que são Volume Drivers?

Volume drivers são plugins que estendem a capacidade do Docker de armazenar dados em diferentes sistemas de storage:

- **Local storage** (padrão)
- **Network storage** (NFS, CIFS/SMB)
- **Cloud storage** (AWS EBS, Azure Disk, GCE Persistent Disk)
- **Distributed storage** (GlusterFS, CephFS)
- **Storage appliances** (NetApp, Pure Storage, Dell EMC)

### Volume Drivers Built-in vs Third-party

| Tipo | Driver | Descrição | Instalação |
|------|--------|-----------|------------|
| **Built-in** | `local` | Storage local do host | ✅ Já incluído |
| **Third-party** | `nfs` | Network File System | Precisa plugin |
| **Third-party** | `azure` | Azure File Storage | Precisa plugin |
| **Third-party** | `aws-ebs` | Amazon Elastic Block Store | Precisa plugin |
| **Third-party** | `gce-pd` | Google Compute Engine Persistent Disk | Precisa plugin |
| **Third-party** | `glusterfs` | GlusterFS distribuído | Precisa plugin |
| **Third-party** | `ceph` | Ceph RBD/CephFS | Precisa plugin |
| **Third-party** | `netapp` | NetApp storage | Precisa plugin |
| **Third-party** | `portworx` | Portworx storage | Precisa plugin |
| **Third-party** | `convoy` | Backup e restore | Precisa plugin |

### Driver Padrão: `local`

O driver `local` é o padrão e armazena dados em `/var/lib/docker/volumes/`.

```bash
# Criar volume com driver local (padrão)
docker volume create my-volume

# Especificar explicitamente o driver local
docker volume create --driver local my-volume

# Ver driver usado
docker volume inspect my-volume

# Output:
# "Driver": "local"
```

### NFS Volume Driver

**NFS** (Network File System) permite compartilhar storage entre múltiplos hosts.

#### Configurar NFS Server (preparação)

```bash
# Em um servidor NFS (exemplo: 192.168.1.100)
sudo apt-get install nfs-kernel-server
sudo mkdir -p /srv/nfs/myshare
sudo chown nobody:nogroup /srv/nfs/myshare

# Configurar export
echo "/srv/nfs/myshare *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports

# Aplicar configuração
sudo exportfs -a
sudo systemctl restart nfs-kernel-server
```

#### Usar NFS Volume no Docker

```bash
# Método 1: Criar volume NFS
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/srv/nfs/myshare \
  nfs-volume

# Método 2: Mount inline no docker run
docker run -d \
  --name nginx \
  --mount type=volume,source=nfs-volume,target=/usr/share/nginx/html,volume-driver=local,volume-opt=type=nfs,volume-opt=o=addr=192.168.1.100,volume-opt=device=:/srv/nfs/myshare \
  nginx

# Usar volume criado
docker run -d \
  --name nginx \
  -v nfs-volume:/usr/share/nginx/html \
  nginx

# Inspecionar volume NFS
docker volume inspect nfs-volume
```

**Vantagens do NFS:**
- ✅ Compartilhamento entre múltiplos hosts
- ✅ Backup centralizado
- ✅ Escalabilidade

**Desvantagens:**
- ⚠️ Performance depende da rede
- ⚠️ Ponto único de falha (NFS server)

### Cloud Volume Drivers

#### AWS EBS (Elastic Block Store)

Usado em ambientes AWS EC2.

```bash
# Instalar plugin AWS EBS
docker plugin install rexray/ebs

# Criar volume EBS
docker volume create --driver rexray/ebs \
  --name ebs-volume \
  --opt size=10 \
  --opt volumetype=gp3

# Usar volume
docker run -d \
  --name app \
  -v ebs-volume:/data \
  myapp

# Volume EBS persiste mesmo se EC2 instance for terminada
```

**Características:**
- ✅ Alta performance (especialmente gp3, io2)
- ✅ Snapshots automáticos
- ✅ Encryption at rest
- ⚠️ Só funciona em AWS
- ⚠️ Vinculado a uma Availability Zone

#### Azure Disk

Usado em Azure VMs.

```bash
# Instalar plugin Azure
docker plugin install rexray/azuredisk

# Criar volume Azure Disk
docker volume create --driver rexray/azuredisk \
  --name azure-volume \
  --opt size=10 \
  --opt storageaccounttype=Premium_LRS

# Usar volume
docker run -d \
  --name app \
  -v azure-volume:/data \
  myapp
```

#### Google Compute Engine Persistent Disk

Usado em GCE VMs.

```bash
# Instalar plugin GCE
docker plugin install rexray/gcepd

# Criar volume GCE PD
docker volume create --driver rexray/gcepd \
  --name gce-volume \
  --opt size=10

# Usar volume
docker run -d \
  --name app \
  -v gce-volume:/data \
  myapp
```

### Distributed Storage Drivers

#### GlusterFS

Storage distribuído para alta disponibilidade.

```bash
# Instalar plugin GlusterFS
docker plugin install trajano/glusterfs-volume-plugin

# Criar volume GlusterFS
docker volume create --driver glusterfs \
  --name gluster-volume \
  --opt glusterserver=192.168.1.100 \
  --opt glustervolume=gv0

# Usar volume
docker run -d \
  --name app \
  -v gluster-volume:/data \
  myapp
```

**Vantagens:**
- ✅ Alta disponibilidade (replicação)
- ✅ Escalável horizontalmente
- ✅ Sem ponto único de falha

#### CephFS / Ceph RBD

Storage distribuído enterprise-grade.

```bash
# Instalar plugin Ceph
docker plugin install ceph/rbd

# Criar volume Ceph RBD
docker volume create --driver ceph/rbd \
  --name ceph-volume \
  --opt mon=192.168.1.100:6789 \
  --opt pool=docker-volumes \
  --opt image=myapp-vol

# Usar volume
docker run -d \
  --name app \
  -v ceph-volume:/data \
  myapp
```

### Instalar e Gerenciar Volume Plugins

#### Instalar Plugin

```bash
# Listar plugins disponíveis
docker plugin ls

# Instalar plugin
docker plugin install <plugin-name>

# Exemplo: REX-Ray (multi-cloud storage)
docker plugin install rexray/s3fs

# Instalar com configuração
docker plugin install rexray/s3fs \
  S3FS_REGION=us-east-1 \
  S3FS_ACCESSKEY=AKIAIOSFODNN7EXAMPLE \
  S3FS_SECRETKEY=wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY

# Listar plugins instalados
docker plugin ls

# Output:
# ID            NAME              ENABLED
# abc123        rexray/s3fs:latest    true
```

#### Gerenciar Plugins

```bash
# Habilitar plugin
docker plugin enable <plugin-id>

# Desabilitar plugin
docker plugin disable <plugin-id>

# Remover plugin (precisa estar desabilitado)
docker plugin disable <plugin-id>
docker plugin rm <plugin-id>

# Inspecionar plugin
docker plugin inspect <plugin-id>

# Ver logs do plugin
journalctl -u docker -f | grep <plugin-name>
```

### Comparação de Volume Drivers

| Driver | Use Case | Performance | HA | Custo | Cloud |
|--------|----------|-------------|----|----|------|
| **local** | Desenvolvimento, dados temporários | ⚡⚡⚡ Muito Alta | ❌ Não | 💰 Grátis | Qualquer |
| **nfs** | Compartilhamento multi-host | ⚡⚡ Alta | ⚠️ Depende | 💰 Baixo | Qualquer |
| **aws-ebs** | Produção AWS, performance | ⚡⚡⚡ Muito Alta | ⚠️ Single AZ | 💰💰 Médio | AWS |
| **azure-disk** | Produção Azure, performance | ⚡⚡⚡ Muito Alta | ⚠️ Single zone | 💰💰 Médio | Azure |
| **gce-pd** | Produção GCP, performance | ⚡⚡⚡ Muito Alta | ⚠️ Single zone | 💰💰 Médio | GCP |
| **glusterfs** | HA, multi-datacenter | ⚡⚡ Alta | ✅ Sim | 💰💰 Médio | Qualquer |
| **ceph** | Enterprise, HA, performance | ⚡⚡⚡ Muito Alta | ✅ Sim | 💰💰💰 Alto | Qualquer |
| **portworx** | Enterprise, HA, Kubernetes | ⚡⚡⚡ Muito Alta | ✅ Sim | 💰💰💰 Alto | Qualquer |

### Exemplo Completo: Multi-host Storage com NFS

**Cenário:** Três Docker hosts compartilhando dados via NFS.

```bash
# === NFS Server (192.168.1.100) ===
sudo apt-get install nfs-kernel-server
sudo mkdir -p /srv/nfs/shared
sudo chown nobody:nogroup /srv/nfs/shared
echo "/srv/nfs/shared *(rw,sync,no_subtree_check)" | sudo tee -a /etc/exports
sudo exportfs -a
sudo systemctl restart nfs-kernel-server

# === Docker Host 1 (192.168.1.101) ===
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/srv/nfs/shared \
  shared-data

docker run -d \
  --name writer \
  -v shared-data:/data \
  busybox \
  sh -c 'while true; do echo "Host1: $(date)" >> /data/log.txt; sleep 5; done'

# === Docker Host 2 (192.168.1.102) ===
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/srv/nfs/shared \
  shared-data

docker run -d \
  --name reader \
  -v shared-data:/data:ro \
  busybox \
  sh -c 'while true; do tail -f /data/log.txt; sleep 1; done'

# === Docker Host 3 (192.168.1.103) ===
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw \
  --opt device=:/srv/nfs/shared \
  shared-data

docker run -d \
  --name viewer \
  -v shared-data:/data:ro \
  nginx

# Todos os containers veem os mesmos dados!
```

### Troubleshooting de Volume Drivers

#### Problema 1: Plugin não inicia

```bash
# Ver logs do Docker
journalctl -u docker -f

# Ver status do plugin
docker plugin ls

# Reinstalar plugin
docker plugin disable <plugin-id>
docker plugin rm <plugin-id>
docker plugin install <plugin-name>
```

#### Problema 2: Volume não monta (NFS)

```bash
# Testar conexão NFS manualmente
sudo mount -t nfs 192.168.1.100:/srv/nfs/shared /mnt/test

# Verificar se NFS server está acessível
showmount -e 192.168.1.100

# Ver logs do container
docker logs <container>

# Inspecionar volume
docker volume inspect <volume-name>
```

#### Problema 3: Performance ruim (Network storage)

```bash
# Usar opções de mount otimizadas para NFS
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=192.168.1.100,rw,nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2 \
  --opt device=:/srv/nfs/shared \
  nfs-optimized

# Para cloud volumes, usar tipos de disco mais rápidos
# AWS: gp3, io2
# Azure: Premium_LRS, UltraSSD_LRS
# GCP: pd-ssd, pd-extreme
```

### Relação com Kubernetes Storage

Volume drivers no Docker são precursores dos **CSI (Container Storage Interface) Drivers** no Kubernetes:

| Docker Volume Driver | Kubernetes Equivalente |
|---------------------|------------------------|
| `local` | `hostPath` |
| `nfs` | NFS Provisioner / `nfs-client` |
| `aws-ebs` | AWS EBS CSI Driver |
| `azure-disk` | Azure Disk CSI Driver |
| `gce-pd` | GCE PD CSI Driver |
| `glusterfs` | GlusterFS CSI Driver |
| `ceph` | Ceph RBD/CephFS CSI Driver |
| `portworx` | Portworx CSI Driver |

**Exemplo de StorageClass no Kubernetes (AWS EBS):**

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-ssd
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer
allowVolumeExpansion: true
```

### Quando Usar Cada Volume Driver?

#### 1. **Desenvolvimento Local** → `local` driver
```bash
docker volume create dev-data
```

#### 2. **Multi-host sem HA** → NFS
```bash
docker volume create --driver local \
  --opt type=nfs \
  --opt o=addr=nfs-server,rw \
  --opt device=:/export/data \
  shared
```

#### 3. **Produção em AWS** → AWS EBS
```bash
docker plugin install rexray/ebs
docker volume create --driver rexray/ebs production-db
```

#### 4. **Alta Disponibilidade Enterprise** → Ceph ou Portworx
```bash
docker plugin install ceph/rbd
docker volume create --driver ceph/rbd ha-data
```

#### 5. **Backup e Disaster Recovery** → Convoy ou Velero
```bash
docker plugin install rancher/convoy
docker volume create --driver convoy backup-vol
```

## 📁 Estrutura de Diretórios do Docker

```bash
/var/lib/docker/
├── buildkit/           # Build cache e dados do BuildKit
├── containers/         # Dados específicos de cada container
│   └── <container-id>/
│       ├── config.v2.json
│       ├── hostconfig.json
│       └── hostname
├── image/              # Metadados de imagens
│   └── overlay2/
│       ├── distribution/
│       ├── imagedb/    # Database de imagens
│       └── layerdb/    # Database de layers
├── network/            # Configurações de rede
│   └── files/
├── overlay2/           # Layers do filesystem (overlay2 driver)
│   ├── <layer-id>/
│   │   ├── diff/       # Conteúdo da layer
│   │   ├── link        # Link simbólico
│   │   └── work/       # Diretório de trabalho
│   └── l/              # Links curtos para layers
├── plugins/            # Plugins do Docker
├── runtimes/           # Runtimes (runc, etc)
├── swarm/              # Dados do Docker Swarm
├── tmp/                # Arquivos temporários
└── volumes/            # Docker volumes
    └── <volume-name>/
        └── _data/      # Dados do volume
```

## 🔍 Comandos Úteis de Storage

### Volumes

```bash
# Criar volume
docker volume create my-vol

# Listar volumes
docker volume ls

# Inspecionar volume
docker volume inspect my-vol

# Ver uso de espaço
docker volume ls --format "table {{.Name}}\t{{.Driver}}"

# Remover volume
docker volume rm my-vol

# Remover volumes não usados
docker volume prune

# Remover TODOS volumes (cuidado!)
docker volume prune -a
```

### Informações de Storage

```bash
# Ver uso de disco total
docker system df

# Output:
# TYPE            TOTAL     ACTIVE    SIZE      RECLAIMABLE
# Images          5         2         2.5GB     1.8GB (72%)
# Containers      3         1         150MB     100MB (66%)
# Local Volumes   10        2         500MB     400MB (80%)
# Build Cache     20        0         1.5GB     1.5GB (100%)

# Ver detalhes
docker system df -v

# Limpar espaço não usado
docker system prune

# Limpar tudo (incluindo volumes)
docker system prune -a --volumes
```

### Inspecionar Container Storage

```bash
# Ver mounts de um container
docker inspect my-container | grep -A 20 "Mounts"

# Ver volumes de um container
docker inspect -f '{{ .Mounts }}' my-container

# Ver layer do container
docker inspect -f '{{ .GraphDriver.Data.UpperDir }}' my-container
```

## 📚 Exemplos Práticos

### Exemplo 1: Persistir dados de banco de dados

```bash
# Criar volume para PostgreSQL
docker volume create postgres-data

# Rodar PostgreSQL com volume
docker run -d \
  --name postgres \
  -e POSTGRES_PASSWORD=secret \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:13

# Dados persistem mesmo se container for removido
docker rm -f postgres

# Criar novo container com mesmo volume
docker run -d \
  --name postgres-new \
  -e POSTGRES_PASSWORD=secret \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:13
# Dados ainda estão lá!
```

### Exemplo 2: Desenvolvimento com bind mount

```bash
# Diretório local com código
mkdir -p ~/myapp
echo "Hello World" > ~/myapp/index.html

# Rodar nginx com bind mount
docker run -d \
  --name nginx-dev \
  -p 8080:80 \
  -v ~/myapp:/usr/share/nginx/html:ro \
  nginx

# Editar arquivo localmente
echo "Hello Docker!" > ~/myapp/index.html

# Mudanças aparecem imediatamente no container
curl http://localhost:8080
# Output: Hello Docker!
```

### Exemplo 3: Compartilhar volume entre containers

```bash
# Criar volume compartilhado
docker volume create shared-data

# Container 1: Escreve dados
docker run -d \
  --name writer \
  -v shared-data:/data \
  busybox \
  sh -c 'while true; do date >> /data/log.txt; sleep 5; done'

# Container 2: Lê dados
docker run -d \
  --name reader \
  -v shared-data:/data:ro \
  busybox \
  sh -c 'while true; do tail -f /data/log.txt; sleep 1; done'

# Ver logs do reader
docker logs -f reader
```

### Exemplo 4: Usar tmpfs para cache

```bash
# Rodar app com cache em memória
docker run -d \
  --name app \
  --mount type=tmpfs,target=/app/cache,tmpfs-size=100m \
  myapp:latest

# Cache é rápido mas não persiste
docker restart app
# Cache foi limpo!
```

## 🐛 Troubleshooting de Storage

### Problema 1: "No space left on device"

```bash
# Verificar uso de disco
docker system df

# Ver detalhes
docker system df -v

# Limpar imagens não usadas
docker image prune -a

# Limpar containers parados
docker container prune

# Limpar volumes não usados
docker volume prune

# Limpar tudo
docker system prune -a --volumes
```

### Problema 2: Container não consegue escrever

```bash
# Verificar permissões do volume
docker volume inspect my-volume

# Ver permissões dentro do container
docker exec my-container ls -la /data

# Rodar container com usuário específico
docker run -d \
  --name app \
  --user 1000:1000 \
  -v my-volume:/data \
  myapp

# Ou ajustar permissões do volume
sudo chown -R 1000:1000 /var/lib/docker/volumes/my-volume/_data/
```

### Problema 3: Volume não é removido

```bash
# Ver containers usando o volume
docker ps -a --filter volume=my-volume

# Remover containers primeiro
docker rm -f $(docker ps -a --filter volume=my-volume -q)

# Agora remover volume
docker volume rm my-volume
```

### Problema 4: Bind mount vazio no container

```bash
# ERRADO: Diretório não existe
docker run -v /path/nao/existe:/data nginx
# Container vê diretório vazio

# CORRETO: Criar diretório primeiro
mkdir -p /path/existe
docker run -v /path/existe:/data nginx

# Ou usar --mount (falha se não existir)
docker run --mount type=bind,source=/path,target=/data nginx
```

## 🔐 Segurança de Storage

### Boas Práticas

1. **Use volumes em vez de bind mounts em produção**
   ```bash
   # Preferir:
   docker run -v my-volume:/data nginx

   # Em vez de:
   docker run -v /host/path:/data nginx
   ```

2. **Use read-only quando possível**
   ```bash
   # Mount read-only
   docker run -v config:/etc/config:ro nginx
   ```

3. **Não exponha `/var/lib/docker`**
   ```bash
   # ❌ NUNCA faça isso:
   docker run -v /var/lib/docker:/host-docker nginx
   # Isso dá acesso total ao Docker host!
   ```

4. **Limite tamanho de tmpfs**
   ```bash
   # Sempre especifique tamanho
   docker run --tmpfs /app/cache:rw,size=100m nginx
   ```

5. **Faça backup de volumes importantes**
   ```bash
   # Backup de volume
   docker run --rm \
     -v my-volume:/data \
     -v $(pwd):/backup \
     busybox tar czf /backup/backup.tar.gz /data

   # Restore de volume
   docker run --rm \
     -v my-volume:/data \
     -v $(pwd):/backup \
     busybox tar xzf /backup/backup.tar.gz -C /
   ```

## 💡 Relação com Kubernetes

No Kubernetes, conceitos similares:

| Docker | Kubernetes |
|--------|------------|
| Volume | PersistentVolume (PV) |
| Volume mount | PersistentVolumeClaim (PVC) |
| tmpfs | emptyDir (com medium: Memory) |
| Bind mount | hostPath (não recomendado) |

**Exemplo em Kubernetes:**

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: data
      mountPath: /usr/share/nginx/html
  volumes:
  - name: data
    persistentVolumeClaim:
      claimName: nginx-pvc
```

## 📖 Comandos Rápidos de Referência

```bash
# === VOLUMES ===

# Criar volume
docker volume create <name>

# Listar volumes
docker volume ls

# Inspecionar volume
docker volume inspect <name>

# Remover volume
docker volume rm <name>

# Limpar volumes não usados
docker volume prune

# === USAR VOLUMES ===

# Volume nomeado
docker run -v <volume-name>:/path container

# Bind mount
docker run -v /host/path:/container/path container

# tmpfs mount
docker run --tmpfs /path container

# Mount syntax (recomendado)
docker run --mount type=volume,source=<name>,target=/path container

# === INFORMAÇÕES ===

# Ver uso de disco
docker system df

# Ver detalhes de uso
docker system df -v

# Limpar tudo
docker system prune -a --volumes

# === INSPEÇÃO ===

# Ver mounts de container
docker inspect <container> | grep -A 20 Mounts

# Ver storage driver
docker info | grep "Storage Driver"

# Ver layers de imagem
docker history <image>
```

## 🎯 Pontos Importantes para CKA

Embora Docker storage não seja testado diretamente no CKA, entender esses conceitos ajuda a compreender:

1. **Persistent Volumes no Kubernetes**
   - Similar a Docker volumes
   - Independentes do ciclo de vida dos pods

2. **EmptyDir volumes**
   - Similar a volumes anônimos do Docker
   - Temporários, duram apenas enquanto o pod existe

3. **HostPath volumes**
   - Similar a bind mounts
   - Montam diretório do node no pod
   - **Não recomendado em produção**

4. **Storage Classes**
   - Provisionamento dinâmico de storage
   - Equivalente a criar volumes automaticamente

## 🔗 Recursos Úteis

### Documentação Oficial
- 📖 [Docker Storage Overview](https://docs.docker.com/storage/)
- 📖 [Docker Volumes](https://docs.docker.com/storage/volumes/)
- 📖 [Bind Mounts](https://docs.docker.com/storage/bind-mounts/)
- 📖 [Storage Drivers](https://docs.docker.com/storage/storagedriver/)

---

⬅️ **Anterior**: [docker-containerd.md](./docker-containerd.md) | ➡️ **Próximo**: [../Componentes-Control-Plane/](../Componentes-Control-Plane/)
