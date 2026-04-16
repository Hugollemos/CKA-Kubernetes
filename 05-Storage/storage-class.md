# Storage Class — Provisionamento Dinâmico

## Problema: Provisionamento Estático

Sem Storage Classes, o fluxo é manual (estático):

1. Administrador cria manualmente um disco no provedor de nuvem (GCP, AWS, Azure)
2. Administrador cria um **PersistentVolume** apontando para esse disco
3. Usuário cria um **PersistentVolumeClaim**
4. K8s vincula o PVC ao PV

Isso é trabalhoso e não escala bem.

---

## Solução: Provisionamento Dinâmico com Storage Class

Com **Storage Class**, o PersistentVolume é criado **automaticamente** quando um PVC é criado.

```
PVC criado → Storage Class detecta → Cria disco no provedor → Cria PV → Vincula ao PVC
```

---

## Criar uma Storage Class

```yaml
# sc-definition.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: google-storage
provisioner: kubernetes.io/gce-pd     # provedor GCP
parameters:
  type: pd-standard                   # tipo de disco
  replication-type: none
```

```bash
kubectl create -f sc-definition.yaml

# Listar Storage Classes
kubectl get sc
kubectl get storageclass
```

---

## Provisioners disponíveis

| Provedor | Provisioner |
|----------|-------------|
| GCP | `kubernetes.io/gce-pd` |
| AWS | `kubernetes.io/aws-ebs` |
| Azure Disk | `kubernetes.io/azure-disk` |
| Azure File | `kubernetes.io/azure-file` |
| Local | `kubernetes.io/no-provisioner` |
| NFS | `nfs.csi.k8s.io` |

---

## PVC usando Storage Class

Ao especificar `storageClassName` no PVC, o Kubernetes cria o PV automaticamente:

```yaml
# pvc-definition.yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: myclaim
spec:
  accessModes:
  - ReadWriteOnce
  storageClassName: google-storage    # referencia a Storage Class
  resources:
    requests:
      storage: 500Mi
```

```bash
kubectl create -f pvc-definition.yaml

# Ver PVC e PV criado automaticamente
kubectl get pvc
kubectl get pv
```

---

## Pod usando o PVC

```yaml
# pod-definition.yaml
apiVersion: v1
kind: Pod
metadata:
  name: mypod
spec:
  containers:
  - name: frontend
    image: nginx
    volumeMounts:
    - mountPath: "/var/www/html"
      name: web
  volumes:
  - name: web
    persistentVolumeClaim:
      claimName: myclaim
```

---

## Storage Class com parâmetros avançados

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd                     # disco SSD
  replication-type: regional-pd   # replicação regional
reclaimPolicy: Retain              # manter disco ao deletar PVC (padrão: Delete)
volumeBindingMode: WaitForFirstConsumer  # aguarda Pod antes de criar disco
allowVolumeExpansion: true         # permitir expansão do volume
```

### Parâmetros importantes

| Parâmetro | Valores | Descrição |
|-----------|---------|-----------|
| `reclaimPolicy` | `Delete`, `Retain` | O que fazer com o disco ao deletar PVC |
| `volumeBindingMode` | `Immediate`, `WaitForFirstConsumer` | Quando criar o disco |
| `allowVolumeExpansion` | `true`, `false` | Permite aumentar o tamanho do PVC |

---

## Storage Class padrão

Você pode definir uma Storage Class como padrão com a annotation:

```yaml
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
```

Com uma SC padrão, PVCs sem `storageClassName` usam automaticamente essa SC.

---

## Fluxo Completo

```bash
# 1. Admin cria a Storage Class
kubectl apply -f storage-class.yaml

# 2. Dev cria o PVC
kubectl apply -f pvc.yaml

# 3. K8s cria PV automaticamente
kubectl get pv

# 4. PVC fica Bound
kubectl get pvc

# 5. Pod usa o PVC
kubectl apply -f pod.yaml
```

---

## Referências

- https://kubernetes.io/docs/concepts/storage/storage-classes/
- https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/
