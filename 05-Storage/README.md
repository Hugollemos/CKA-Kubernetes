# Storage (Armazenamento)

Esta pasta contém guias completos sobre Storage no Kubernetes, cobrindo Volumes, Persistent Volumes e Container Storage Interface (CSI).

## 📚 Conteúdo

### [storage-class.md](./storage-class.md)
**Provisionamento dinâmico de volumes**
- Provisionamento estático vs dinâmico
- Criar StorageClass com provisioners (GCP, AWS, Azure)
- PVC usando StorageClass (`storageClassName`)
- Parâmetros: reclaimPolicy, volumeBindingMode, allowVolumeExpansion
- Storage Class padrão (annotation `is-default-class`)

### [volumes-persistent-volumes.md](./volumes-persistent-volumes.md)
**Guia completo sobre storage no Kubernetes**

#### Volumes
- O que são Volumes e por que usar
- Tipos de volumes: emptyDir, hostPath, configMap, secret, nfs, downwardAPI, projected
- Ciclo de vida de volumes
- Volume vs Container filesystem

#### Persistent Volumes (PV)
- O que são PersistentVolumes
- Diferença entre PV e PVC
- Binding process (como PVC e PV se conectam)
- Estados de PV e PVC (Available, Bound, Released, Pending)
- Static Provisioning (admin cria PV manualmente)
- Dynamic Provisioning (StorageClass cria PV automaticamente)

#### Persistent Volume Claims (PVC)
- Como criar e usar PVCs
- Binding automático PVC → PV
- PVC em Pods e Deployments

#### Storage Classes
- O que são StorageClasses
- Provisionamento dinâmico de volumes
- Parâmetros e configurações
- Volume Binding Modes (Immediate vs WaitForFirstConsumer)
- allowVolumeExpansion (expansão de volumes)
- Marcar StorageClass como padrão

#### Container Storage Interface (CSI)
- O que é CSI e por que foi criado
- Arquitetura CSI (Controller Plugin vs Node Plugin)
- CSI vs In-tree volumes (drivers legados)
- CSI Drivers populares (AWS EBS, Azure Disk, GCE PD, NFS, Ceph, Portworx, etc.)
- Sidecars CSI (provisioner, attacher, resizer, snapshotter)
- Instalação de CSI drivers
- Volume Snapshots
- Volume Cloning
- Volume Expansion

#### Características de Volumes
- **Access Modes**: RWO, ROX, RWX, RWOP
- **Volume Modes**: Filesystem vs Block
- **Reclaim Policies**: Retain, Delete, Recycle

#### Exemplos Práticos
- PostgreSQL com PVC persistente
- Expandir volumes online
- Shared storage (NFS) com múltiplos pods

#### Troubleshooting
- PVC fica em Pending
- Pod não consegue montar volume
- Volume não é deletado (Released)
- Expansão de volume falha
- Performance ruim de I/O
- CSI driver não funciona

## 🎯 Importância para o Exame CKA

Storage representa **10% da prova** no domínio "Storage".

### ✅ No Escopo do CKA

É essencial entender:
- Diferença entre Volumes e PersistentVolumes
- Como criar e usar PVCs
- StorageClasses e provisionamento dinâmico
- Access Modes e quando usar cada um
- Troubleshooting de problemas de mounting
- Expandir volumes
- CSI drivers básicos (conceito, não implementação)

### ❌ Fora do Escopo do CKA

**NÃO será cobrado na prova:**
- StatefulSets (workload específico, não é testado no CKA)
- VolumeClaimTemplates (usado apenas em StatefulSets)
- Volume Snapshots (feature avançada)
- Volume Cloning (feature avançada)
- Custom CSI driver development (apenas conceitos básicos são cobrados)
- Advanced storage features (topology, raw block volumes detalhados)

## 💡 Dica de Prova

Na prova, você pode precisar:
- Criar PVC e montar em pod (MUITO COMUM!)
- Configurar StorageClass para provisionamento dinâmico
- Troubleshooting de PVC em Pending (COMUM!)
- Expandir volume existente
- Compartilhar volume entre múltiplos pods (NFS com ReadWriteMany)
- Configurar Reclaim Policy

### Comandos Essenciais

```bash
# Ver Persistent Volumes
kubectl get pv

# Ver Persistent Volume Claims
kubectl get pvc

# Ver Storage Classes
kubectl get storageclass
kubectl get sc

# Criar PVC
kubectl apply -f pvc.yaml

# Ver detalhes de PVC (troubleshooting)
kubectl describe pvc <name>

# Ver CSI Drivers instalados
kubectl get csidriver

# Expandir volume
kubectl patch pvc <name> -p '{"spec":{"resources":{"requests":{"storage":"20Gi"}}}}'

# Deletar PVC
kubectl delete pvc <name>
```

### Conceitos Críticos

1. **PV = Oferta, PVC = Pedido**
   - Admin cria PV (ou StorageClass provisiona automaticamente)
   - Desenvolvedor cria PVC
   - Kubernetes faz binding automaticamente

2. **Access Modes**
   - RWO (ReadWriteOnce): 1 node apenas - AWS EBS, Azure Disk, GCE PD
   - ROX (ReadOnlyMany): N nodes read-only
   - RWX (ReadWriteMany): N nodes read-write - NFS, CephFS, GlusterFS

3. **Reclaim Policy**
   - Retain: PV não é deletado quando PVC é removido (mais seguro)
   - Delete: PV é deletado automaticamente (padrão para dynamic provisioning)

4. **Static vs Dynamic Provisioning**
   - Static: Admin cria PV manualmente → Dev cria PVC → Binding
   - Dynamic: Dev cria PVC com StorageClass → PV criado automaticamente

5. **CSI é o padrão moderno**
   - In-tree volumes (hostPath, nfs, awsElasticBlockStore) estão deprecated
   - Use CSI drivers para novos deployments

## 🔗 Ordem de Estudo

1. **Volumes básicos** (emptyDir, hostPath, configMap)
2. **PersistentVolumes e PVCs** (conceitos fundamentais)
3. **Storage Classes** (provisionamento dinâmico)
4. **Access Modes e Reclaim Policies**
5. **CSI Drivers** (arquitetura moderna)
6. **Troubleshooting** (PVC pending, mount failures)

---

⬅️ **Anterior**: [04-Services-Networking](../04-Services-Networking/) | ➡️ **Próximo**: [06-Troubleshooting](../06-Troubleshooting/)
