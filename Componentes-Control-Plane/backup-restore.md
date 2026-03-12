# Backup and Restore Methods (Métodos de Backup e Restore)

## 📋 O que é Backup no Kubernetes?

**Backup** no Kubernetes é o processo de salvar o estado do cluster para que possa ser **restaurado** em caso de falha, desastre ou migração. O componente mais crítico para backup é o **ETCD**, que armazena todo o estado do cluster.

## 🎯 O que precisa de Backup?

### 1. **ETCD** (CRÍTICO!)
- Armazena TODO o estado do cluster
- Configurações de todos os recursos
- Secrets, ConfigMaps, deployments, etc.

### 2. **Certificados e Configurações**
- `/etc/kubernetes/pki/` - Certificados do cluster
- `/etc/kubernetes/manifests/` - Manifests dos componentes do control plane
- `~/.kube/config` - Configurações do kubectl

### 3. **Persistent Volumes**
- Dados das aplicações
- Geralmente backup separado (storage provider)

### 4. **Resource Definitions**
- YAMLs dos recursos do cluster
- Pode ser versionado em Git

## 🚀 Guia Rápido de Referência

```bash
# 1. VERIFICAR VERSÃO
etcdctl version  # Verifica API version (deve ser 3.x)

# 2. CRIAR BACKUP (etcd rodando)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 3. VERIFICAR BACKUP
etcdctl snapshot status /backup/etcd.db --write-out=table

# 4. RESTAURAR BACKUP (etcd parado)
etcdutl snapshot restore /backup/etcd.db --data-dir=/var/lib/etcd-restored
```

## 🛠️ Trabalhando com ETCDCTL e ETCDUTL

### O que são ETCDCTL e ETCDUTL?

**etcdctl** é o cliente de linha de comando para interagir com o etcd (operações online).

**etcdutl** é uma ferramenta utilitária para operações offline de backup e restore do etcd (sem etcd rodando).

### Verificar Versão do ETCDCTL

```bash
# Verificar versão do etcdctl
etcdctl version

# Output esperado:
# etcdctl version: 3.5.16
# API version: 3.5
```

**IMPORTANTE:** Em todos os labs do Kubernetes, o ETCD roda como static pod no master e usa a versão v3. Sempre use `ETCDCTL_API=3` para garantir que está usando a API correta.

### Diferença entre etcdctl e etcdutl

| Ferramenta | Uso | ETCD precisa estar rodando? | Método |
|------------|-----|----------------------------|---------|
| **etcdctl snapshot save** | Criar snapshot de etcd live | ✅ Sim | Snapshot online |
| **etcdctl snapshot status** | Ver informações do snapshot | ❌ Não | Leitura de arquivo |
| **etcdutl snapshot restore** | Restaurar snapshot .db | ❌ Não | Restauração offline |
| **etcdutl backup** | Backup de arquivos raw (data + WAL) | ❌ Não | Cópia de arquivos |

## 🔄 Métodos de Backup

### Método 1: ETCD Snapshot com etcdctl (RECOMENDADO)

Este é o método **mais importante** e **mais cobrado na prova CKA**.

#### Como funciona:
```
┌──────────────────────────────────────────┐
│ ETCD Database                            │
│ - Todos os recursos do cluster           │
│ - Estado atual completo                  │
└──────────────┬───────────────────────────┘
               │
               │ etcdctl snapshot save
               ↓
┌──────────────────────────────────────────┐
│ Snapshot File (.db)                      │
│ - Cópia point-in-time do ETCD           │
│ - Pode ser restaurado em caso de falha  │
└──────────────────────────────────────────┘
```

#### Criar Snapshot do ETCD (etcdctl snapshot save)

**Usado para:** Criar backup de um etcd em execução (online backup)

```bash
# Localizar informações do ETCD
cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'advertise-client-urls|ca-file|cert-file|key-file'

# Criar snapshot
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# Output:
# Snapshot saved at /backup/etcd-snapshot.db
```

**Opções obrigatórias:**
- `--endpoints`: URL do ETCD server (padrão: localhost:2379)
- `--cacert`: Caminho para o certificado CA
- `--cert`: Caminho para o certificado do cliente
- `--key`: Caminho para a chave do cliente

**⚠️ IMPORTANTE:** Sempre use `ETCDCTL_API=3` antes do comando!

#### Verificar Snapshot (etcdctl snapshot status)

**Usado para:** Inspecionar metadados de um arquivo snapshot

```bash
# Ver status do snapshot (formato padrão)
etcdctl snapshot status /backup/etcd-snapshot.db

# Output mostra:
# Hash, Revision, Total Keys, Total Size

# Ver detalhes em formato table (RECOMENDADO)
etcdctl snapshot status /backup/etcd-snapshot.db --write-out=table

# Output:
# +----------+----------+------------+------------+
# |   HASH   | REVISION | TOTAL KEYS | TOTAL SIZE |
# +----------+----------+------------+------------+
# | 12345678 |   123456 |       1234 |    5.0 MB  |
# +----------+----------+------------+------------+
```

**Informações exibidas:**
- **HASH**: Hash do snapshot para verificar integridade
- **REVISION**: Número de revisão do etcd no momento do snapshot
- **TOTAL KEYS**: Número total de chaves armazenadas
- **TOTAL SIZE**: Tamanho do snapshot em MB

**✅ Benefício:** Útil para verificar a integridade do snapshot antes de fazer restore!

#### Restaurar ETCD Snapshot (etcdutl snapshot restore)

**Usado para:** Restaurar um arquivo snapshot .db para um novo data directory

**⚠️ IMPORTANTE:** A partir do Kubernetes 1.23+, use `etcdutl` (não `etcdctl`) para restore!

```bash
# 1. Parar o kube-apiserver (para evitar conflitos)
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Restaurar snapshot para novo diretório usando etcdutl
etcdutl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored

# Output:
# 2024-03-05 {"level":"info","ts":...,"msg":"restored snapshot"}
# 2024-03-05 {"level":"info","ts":...,"msg":"added member","member-id":"12345"}

# Comando alternativo com mais opções (se necessário)
etcdutl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-restored \
  --name=master \
  --initial-cluster=master=https://127.0.0.1:2380 \
  --initial-advertise-peer-urls=https://127.0.0.1:2380

# 3. Atualizar etcd.yaml para usar novo data-dir
vi /etc/kubernetes/manifests/etcd.yaml

# Modificar a seção de volumes:
# volumes:
# - hostPath:
#     path: /var/lib/etcd-restored    # ← Mudar de /var/lib/etcd
#     type: DirectoryOrCreate
#   name: etcd-data

# 4. Restaurar kube-apiserver
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Aguardar componentes voltarem
watch kubectl get pods -n kube-system

# 6. Verificar se restore funcionou
kubectl get all --all-namespaces
```

**📝 Notas importantes:**
- `etcdctl snapshot restore` foi substituído por `etcdutl snapshot restore` em versões recentes
- O ETCD não precisa estar rodando durante o restore
- Sempre restaure para um **novo** data-dir (não sobrescreva o existente diretamente)
- Use `etcdutl` para restore offline

### Método 1b: Backup File-Based com etcdutl (Offline)

**Usado para:** Backup offline do diretório de dados do etcd (sem etcd rodando)

Este método copia diretamente os arquivos do banco de dados backend e os arquivos WAL (Write-Ahead Log).

#### Criar Backup File-Based

```bash
# Backup offline dos arquivos do etcd
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-backup

# Isso copia:
# - Backend database files
# - WAL (Write-Ahead Log) files
```

**Quando usar:**
- ✅ Quando o etcd está parado (manutenção)
- ✅ Para backup de arquivos raw
- ✅ Para migração de dados entre servidores

#### Restaurar Backup File-Based

```bash
# Simplesmente copie os arquivos de volta
cp -r /backup/etcd-backup/* /var/lib/etcd/

# Reinicie o etcd
systemctl restart etcd

# Ou se for static pod, mova o manifest de volta
mv /tmp/etcd.yaml /etc/kubernetes/manifests/
```

### Método 2: Backup Declarativo (Resource Definitions)

Salvar as definições YAML de todos os recursos.

#### Backup de todos os recursos

```bash
# Criar diretório de backup
mkdir -p /backup/k8s-resources

# Backup de todos os namespaces
kubectl get namespaces -o yaml > /backup/k8s-resources/namespaces.yaml

# Backup de recursos em cada namespace
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  mkdir -p /backup/k8s-resources/$ns

  # Deployments
  kubectl get deployments -n $ns -o yaml > /backup/k8s-resources/$ns/deployments.yaml

  # Services
  kubectl get services -n $ns -o yaml > /backup/k8s-resources/$ns/services.yaml

  # ConfigMaps
  kubectl get configmaps -n $ns -o yaml > /backup/k8s-resources/$ns/configmaps.yaml

  # Secrets
  kubectl get secrets -n $ns -o yaml > /backup/k8s-resources/$ns/secrets.yaml

  # PersistentVolumeClaims
  kubectl get pvc -n $ns -o yaml > /backup/k8s-resources/$ns/pvcs.yaml
done

# Backup de recursos cluster-wide
kubectl get nodes -o yaml > /backup/k8s-resources/nodes.yaml
kubectl get persistentvolumes -o yaml > /backup/k8s-resources/pvs.yaml
kubectl get storageclasses -o yaml > /backup/k8s-resources/storageclasses.yaml
kubectl get clusterroles -o yaml > /backup/k8s-resources/clusterroles.yaml
kubectl get clusterrolebindings -o yaml > /backup/k8s-resources/clusterrolebindings.yaml
```

#### Backup Simplificado (todos os recursos)

```bash
# Backup de TODOS os recursos de uma vez
kubectl get all --all-namespaces -o yaml > /backup/k8s-all-resources.yaml

# Problema: "all" não inclui tudo (não pega secrets, configmaps, etc.)
```

#### Restaurar recursos declarativos

```bash
# Restaurar de um arquivo
kubectl apply -f /backup/k8s-resources/namespaces.yaml

# Restaurar todos os recursos de um namespace
kubectl apply -f /backup/k8s-resources/default/

# Restaurar recursivamente
kubectl apply -f /backup/k8s-resources/ --recursive
```

### Método 3: Backup de Certificados

```bash
# Criar diretório de backup
mkdir -p /backup/pki

# Backup dos certificados
cp -r /etc/kubernetes/pki /backup/
cp -r /etc/kubernetes/manifests /backup/

# Backup do kubeconfig
cp ~/.kube/config /backup/kubeconfig

# Verificar certificados
ls -la /backup/pki/
```

#### Restaurar certificados

```bash
# Restaurar certificados
cp -r /backup/pki/* /etc/kubernetes/pki/

# Restaurar manifests
cp -r /backup/manifests/* /etc/kubernetes/manifests/

# Reiniciar kubelet
systemctl restart kubelet
```

### Método 4: Velero (Backup Tool Completo)

**Velero** é uma ferramenta open-source da VMware para backup completo do Kubernetes.

#### Instalação do Velero

```bash
# Download do Velero
wget https://github.com/vmware-tanzu/velero/releases/download/v1.12.0/velero-v1.12.0-linux-amd64.tar.gz
tar -xvf velero-v1.12.0-linux-amd64.tar.gz
sudo mv velero-v1.12.0-linux-amd64/velero /usr/local/bin/

# Verificar instalação
velero version
```

#### Configurar Velero (exemplo com AWS S3)

```bash
# Criar arquivo de credentials
cat > credentials-velero <<EOF
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
EOF

# Instalar Velero no cluster
velero install \
  --provider aws \
  --plugins velero/velero-plugin-for-aws:v1.8.0 \
  --bucket my-backup-bucket \
  --secret-file ./credentials-velero \
  --backup-location-config region=us-east-1 \
  --snapshot-location-config region=us-east-1

# Verificar instalação
kubectl get pods -n velero
```

#### Criar Backup com Velero

```bash
# Backup de todo o cluster
velero backup create full-cluster-backup

# Backup de namespace específico
velero backup create app-backup --include-namespaces default

# Backup excluindo namespaces
velero backup create backup-1 --exclude-namespaces kube-system,kube-public

# Backup com schedule (diário às 2am)
velero schedule create daily-backup --schedule="0 2 * * *"

# Ver backups
velero backup get

# Descrever backup
velero backup describe full-cluster-backup
```

#### Restaurar com Velero

```bash
# Listar backups disponíveis
velero backup get

# Restaurar backup completo
velero restore create --from-backup full-cluster-backup

# Restaurar namespace específico
velero restore create --from-backup full-cluster-backup \
  --include-namespaces default

# Ver status do restore
velero restore get
velero restore describe <restore-name>

# Ver logs
velero restore logs <restore-name>
```

## 🔧 Script de Backup Automatizado

### Script: backup-etcd.sh

```bash
#!/bin/bash

# Configurações
BACKUP_DIR="/backup/etcd"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
SNAPSHOT_FILE="$BACKUP_DIR/etcd-snapshot-$TIMESTAMP.db"
RETENTION_DAYS=7

# Criar diretório se não existir
mkdir -p $BACKUP_DIR

# Certificados do ETCD
ETCD_ENDPOINTS="https://127.0.0.1:2379"
ETCD_CACERT="/etc/kubernetes/pki/etcd/ca.crt"
ETCD_CERT="/etc/kubernetes/pki/etcd/server.crt"
ETCD_KEY="/etc/kubernetes/pki/etcd/server.key"

# Criar snapshot
echo "$(date): Iniciando backup do ETCD..."
ETCDCTL_API=3 etcdctl snapshot save $SNAPSHOT_FILE \
  --endpoints=$ETCD_ENDPOINTS \
  --cacert=$ETCD_CACERT \
  --cert=$ETCD_CERT \
  --key=$ETCD_KEY

# Verificar se backup foi criado
if [ $? -eq 0 ]; then
  echo "$(date): Backup criado com sucesso: $SNAPSHOT_FILE"

  # Verificar integridade
  ETCDCTL_API=3 etcdctl snapshot status $SNAPSHOT_FILE --write-out=table

  # Remover backups antigos
  find $BACKUP_DIR -name "etcd-snapshot-*.db" -mtime +$RETENTION_DAYS -delete
  echo "$(date): Backups antigos removidos (mais de $RETENTION_DAYS dias)"
else
  echo "$(date): ERRO ao criar backup!"
  exit 1
fi

# Compactar backup (opcional)
gzip $SNAPSHOT_FILE
echo "$(date): Backup compactado: $SNAPSHOT_FILE.gz"

# Enviar para storage remoto (opcional - exemplo com AWS S3)
# aws s3 cp $SNAPSHOT_FILE.gz s3://my-backup-bucket/etcd-backups/

echo "$(date): Backup finalizado com sucesso!"
```

### Agendar backup automático com Cron

```bash
# Editar crontab
crontab -e

# Adicionar linha para backup diário às 2am
0 2 * * * /usr/local/bin/backup-etcd.sh >> /var/log/etcd-backup.log 2>&1

# Backup a cada 6 horas
0 */6 * * * /usr/local/bin/backup-etcd.sh >> /var/log/etcd-backup.log 2>&1

# Ver logs
tail -f /var/log/etcd-backup.log
```

## 📝 Resumo de Comandos ETCDCTL vs ETCDUTL

### Tabela de Referência Rápida

| Operação | Comando | Ferramenta | ETCD rodando? |
|----------|---------|------------|---------------|
| **Criar snapshot online** | `ETCDCTL_API=3 etcdctl snapshot save` | etcdctl | ✅ Sim |
| **Ver status do snapshot** | `etcdctl snapshot status --write-out=table` | etcdctl | ❌ Não |
| **Restaurar snapshot** | `etcdutl snapshot restore` | etcdutl | ❌ Não |
| **Backup de arquivos** | `etcdutl backup --data-dir --backup-dir` | etcdutl | ❌ Não |

### Comandos Essenciais

```bash
# 1. VERIFICAR VERSÃO
etcdctl version

# 2. CRIAR SNAPSHOT (online - etcd rodando)
ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 3. VERIFICAR SNAPSHOT
etcdctl snapshot status /backup/etcd.db --write-out=table

# 4. RESTAURAR SNAPSHOT (offline - etcd parado)
etcdutl snapshot restore /backup/etcd.db \
  --data-dir=/var/lib/etcd-restored

# 5. BACKUP DE ARQUIVOS (offline - etcd parado)
etcdutl backup \
  --data-dir /var/lib/etcd \
  --backup-dir /backup/etcd-files
```

### 🎯 Quando usar cada ferramenta

**Use etcdctl para:**
- ✅ Criar snapshots de etcd em execução (`snapshot save`)
- ✅ Verificar status de snapshots (`snapshot status`)
- ✅ Operações administrativas gerais do etcd

**Use etcdutl para:**
- ✅ Restaurar snapshots (`snapshot restore`)
- ✅ Backup offline de arquivos (`backup`)
- ✅ Operações que não requerem etcd rodando

## 📊 Comparação de Métodos

| Método | Vantagens | Desvantagens | Quando usar |
|--------|-----------|--------------|-------------|
| **ETCD Snapshot (etcdctl)** | ✅ Backup completo<br>✅ Rápido<br>✅ Point-in-time<br>✅ Etcd rodando | ❌ Requer certificados<br>❌ Downtime no restore | Backup diário do cluster |
| **File Backup (etcdutl)** | ✅ Backup raw de arquivos<br>✅ Não requer etcd rodando | ❌ Etcd deve estar parado<br>❌ Menos flexível | Backup durante manutenção |
| **Resource Definitions** | ✅ Versionável (Git)<br>✅ Portável<br>✅ Auditável | ❌ Não pega tudo<br>❌ Trabalhoso | GitOps, IaC |
| **Certificados** | ✅ Essencial para recovery<br>✅ Simples | ❌ Não é backup de dados | Antes de renovar certs |
| **Velero** | ✅ Completo<br>✅ Automático<br>✅ Storage remoto | ❌ Ferramenta adicional<br>❌ Complexidade | Produção enterprise |

## 🚨 Disaster Recovery Scenarios

### Cenário 1: Cluster totalmente perdido

```bash
# 1. Reinstalar Kubernetes com kubeadm
kubeadm init --config=/backup/kubeadm-config.yaml

# 2. Restaurar certificados
cp -r /backup/pki/* /etc/kubernetes/pki/

# 3. Restaurar ETCD snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd

# 4. Reiniciar control plane
systemctl restart kubelet

# 5. Verificar
kubectl get nodes
kubectl get all --all-namespaces
```

### Cenário 2: Namespace deletado acidentalmente

```bash
# Se tiver backup declarativo
kubectl apply -f /backup/k8s-resources/my-namespace/

# Se tiver Velero backup
velero restore create --from-backup full-cluster-backup \
  --include-namespaces my-namespace
```

### Cenário 3: ETCD corrompido

```bash
# 1. Parar ETCD
systemctl stop etcd

# 2. Remover dados corrompidos
rm -rf /var/lib/etcd/*

# 3. Restaurar snapshot
ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd

# 4. Reiniciar ETCD
systemctl start etcd

# 5. Verificar saúde
ETCDCTL_API=3 etcdctl endpoint health
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Verificar versão do etcdctl**
   ```bash
   etcdctl version
   # Sempre use ETCDCTL_API=3 !
   ```

2. **Criar snapshot do ETCD (etcdctl)**
   ```bash
   ETCDCTL_API=3 etcdctl snapshot save /backup/etcd.db \
     --endpoints=https://127.0.0.1:2379 \
     --cacert=/etc/kubernetes/pki/etcd/ca.crt \
     --cert=/etc/kubernetes/pki/etcd/server.crt \
     --key=/etc/kubernetes/pki/etcd/server.key
   ```

3. **Verificar snapshot (etcdctl)**
   ```bash
   etcdctl snapshot status /backup/etcd.db --write-out=table
   ```

4. **Restaurar snapshot (etcdutl)**
   ```bash
   # Versões recentes (1.23+): use etcdutl
   etcdutl snapshot restore /backup/etcd.db \
     --data-dir=/var/lib/etcd-restored

   # Versões antigas: etcdctl também funciona
   ETCDCTL_API=3 etcdctl snapshot restore /backup/etcd.db \
     --data-dir=/var/lib/etcd-restored
   ```

5. **Localizar certificados do ETCD**
   ```bash
   cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'cert-file|key-file|ca-file'
   # Ou:
   cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'advertise-client-urls|trusted-ca-file'
   ```

6. **Backup de recursos específicos**
   ```bash
   kubectl get all -n default -o yaml > backup.yaml
   ```

### 📌 Diferenças Importantes

**etcdctl vs etcdutl:**
- `etcdctl snapshot save` → Cria snapshot (etcd rodando) ✅
- `etcdctl snapshot status` → Verifica snapshot (arquivo) ✅
- `etcdutl snapshot restore` → Restaura snapshot (etcd parado) ✅
- `etcdutl backup` → Backup de arquivos raw (etcd parado) ✅

### 🧪 Cenário típico na prova:

> **"Faça backup do ETCD e salve em /opt/etcd-backup.db"**

```bash
# 1. Verificar versão
etcdctl version

# 2. Verificar localização dos certificados
cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'cert-file|key-file|trusted-ca-file'

# 3. Criar backup
ETCDCTL_API=3 etcdctl snapshot save /opt/etcd-backup.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key

# 4. Verificar (IMPORTANTE!)
etcdctl snapshot status /opt/etcd-backup.db --write-out=table
ls -lh /opt/etcd-backup.db
```

> **"Restaure o ETCD a partir do backup /opt/etcd-backup.db"**

```bash
# 1. Parar API server
mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/

# 2. Restaurar usando etcdutl (versão recente)
etcdutl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# OU usar etcdctl (versão antiga - ainda funciona)
ETCDCTL_API=3 etcdctl snapshot restore /opt/etcd-backup.db \
  --data-dir=/var/lib/etcd-restored

# 3. Atualizar etcd.yaml
vi /etc/kubernetes/manifests/etcd.yaml
# Mudar volumes.hostPath de /var/lib/etcd para /var/lib/etcd-restored

# Exemplo:
# volumes:
# - hostPath:
#     path: /var/lib/etcd-restored
#     type: DirectoryOrCreate
#   name: etcd-data

# 4. Restaurar API server
mv /tmp/kube-apiserver.yaml /etc/kubernetes/manifests/

# 5. Aguardar e verificar
watch kubectl get pods -n kube-system
kubectl get all
```

## 💡 Dicas para a Prova

1. **Verifique a versão do etcdctl primeiro**
   ```bash
   etcdctl version  # Verifica se é v3.x
   ```

2. **ETCDCTL_API=3 é obrigatório para snapshot save**
   - Sempre use `ETCDCTL_API=3` antes de `etcdctl snapshot save`
   - Exemplo: `ETCDCTL_API=3 etcdctl snapshot save ...`

3. **Use etcdutl para restore (versões recentes)**
   - `etcdutl snapshot restore` (Kubernetes 1.23+)
   - `etcdctl snapshot restore` ainda funciona em versões antigas

4. **Certificados estão em /etc/kubernetes/pki/etcd/**
   - ca.crt, server.crt, server.key
   - Verifique sempre em `/etc/kubernetes/manifests/etcd.yaml`

5. **Endpoint padrão: https://127.0.0.1:2379**
   - Verifique em /etc/kubernetes/manifests/etcd.yaml
   - Procure por `--advertise-client-urls`

6. **Sempre verifique o snapshot após criar**
   ```bash
   etcdctl snapshot status /backup/etcd.db --write-out=table
   ```

7. **Data-dir deve ser NOVO no restore**
   - Não restaure para /var/lib/etcd diretamente
   - Use /var/lib/etcd-restored, /var/lib/etcd-backup, etc.

8. **Pare o API server antes de restore**
   ```bash
   mv /etc/kubernetes/manifests/kube-apiserver.yaml /tmp/
   ```

9. **Não esqueça de atualizar etcd.yaml**
   - Altere `volumes.hostPath` para o novo data-dir

10. **Comandos estão na documentação**
    - https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/
    - Use Ctrl+F para buscar "backup" ou "snapshot"

### 🔑 Comandos que você DEVE memorizar

```bash
# Criar snapshot
ETCDCTL_API=3 etcdctl snapshot save <file> \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=<ca-cert> --cert=<cert> --key=<key>

# Verificar snapshot
etcdctl snapshot status <file> --write-out=table

# Restaurar snapshot
etcdutl snapshot restore <file> --data-dir=<new-dir>

# Localizar certificados
cat /etc/kubernetes/manifests/etcd.yaml | grep -E 'cert-file|key-file|ca-file'
```

## 📚 Checklist de Backup

**Diário:**
- [ ] Snapshot do ETCD
- [ ] Verificar integridade do snapshot
- [ ] Rotação de backups antigos

**Semanal:**
- [ ] Backup de certificados
- [ ] Backup de resource definitions
- [ ] Teste de restore em ambiente não-produção

**Mensal:**
- [ ] Disaster recovery drill completo
- [ ] Verificar todos os backups
- [ ] Atualizar documentação

**Antes de mudanças:**
- [ ] Snapshot do ETCD
- [ ] Backup de recursos afetados
- [ ] Documentar estado atual

## ⚠️ Notas Importantes sobre Versões

### etcdctl vs etcdutl - Qual usar?

A partir do **Kubernetes 1.23** e **etcd 3.5+**, a ferramenta `etcdutl` foi introduzida para separar as operações offline das operações online:

| Versão | Snapshot Save | Snapshot Status | Snapshot Restore |
|--------|---------------|-----------------|------------------|
| **etcd < 3.5** | `etcdctl` | `etcdctl` | `etcdctl` |
| **etcd >= 3.5** | `etcdctl` | `etcdctl` | `etcdutl` ✅ |

**Na prova CKA:**
- Se `etcdutl` estiver disponível, use para restore
- Se não estiver, use `etcdctl snapshot restore` (ainda funciona)
- Sempre use `ETCDCTL_API=3` para `etcdctl`

**Como verificar:**
```bash
# Verificar se etcdutl existe
which etcdutl

# Se existir, use etcdutl para restore
# Se não existir, use etcdctl para restore
```

### Mudanças importantes

1. **etcdctl snapshot save** → Continua igual ✅
2. **etcdctl snapshot status** → Continua igual ✅
3. **etcdctl snapshot restore** → Agora é **etcdutl snapshot restore** ⚠️
4. **etcdutl backup** → Nova funcionalidade para backup offline 🆕

## 🔗 Recursos Úteis

### Documentação Oficial

- 📖 [Kubernetes: Backing up an etcd cluster](https://kubernetes.io/docs/tasks/administer-cluster/configure-upgrade-etcd/#backing-up-an-etcd-cluster) - **Essencial para a prova!**
- 📖 [ETCD Official Documentation](https://etcd.io/docs/)
- 📖 [ETCD Recovery Guide (etcdctl vs etcdutl)](https://github.com/etcd-io/website/blob/main/content/en/docs/v3.5/op-guide/recovery.md) - Guia oficial sobre recovery
- 📖 [Kubernetes Backup Best Practices](https://kubernetes.io/docs/tasks/administer-cluster/backup-cluster-state/)

### Ferramentas

- 🛠️ [Velero Documentation](https://velero.io/docs/) - Backup enterprise para Kubernetes

### Vídeos Recomendados

- 🎥 [Kubernetes Backup and Restore Explained](https://www.youtube.com/watch?v=qRPNuT080Hk) - Tutorial prático de backup/restore
- 🎥 [ETCD Backup and Restore Deep Dive](https://www.youtube.com/watch?v=MTnQW9MxnRI) - Detalhes sobre ETCD

---

⬅️ **Anterior**: [etcd.md](./etcd.md) | ➡️ **Próximo**: [cluster-upgrade.md](./cluster-upgrade.md)
