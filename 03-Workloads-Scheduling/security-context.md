# Security Context

## O que é Security Context?

**Security Context** define configurações de segurança para Pods e containers, como:
- Qual usuário executa o processo
- Se o container pode escalar privilégios
- Capacidades Linux (Linux capabilities)
- Sistemas de arquivos read-only

Equivalente Docker:
```bash
docker run --user=1001 ubuntu sleep 3600
docker run --cap-add MAC_ADMIN ubuntu
```

---

## Nível de Pod vs Nível de Container

- Configurações no **Pod** se aplicam a todos os containers
- Configurações no **Container** sobrescrevem as do Pod

---

## Security Context no nível do Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  securityContext:
    runAsUser: 1000        # todos os containers rodam como UID 1000
    runAsGroup: 3000
    fsGroup: 2000          # grupo do sistema de arquivos dos volumes
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
```

---

## Security Context no nível do Container

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
```

---

## Linux Capabilities

Capabilities permitem conceder permissões específicas sem dar acesso root completo:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
spec:
  containers:
  - name: ubuntu
    image: ubuntu
    command: ["sleep", "3600"]
    securityContext:
      runAsUser: 1000
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]   # adicionar capabilities
        drop: ["ALL"]                     # remover todas e adicionar apenas as necessárias
```

> Capabilities só podem ser definidas no nível de **container**, não no nível de Pod.

---

## Campos Comuns do Security Context

| Campo | Nível | Descrição |
|-------|-------|-----------|
| `runAsUser` | Pod/Container | UID do processo |
| `runAsGroup` | Pod/Container | GID primário do processo |
| `runAsNonRoot` | Pod/Container | Impede execução como root |
| `fsGroup` | Pod | GID dos volumes montados |
| `allowPrivilegeEscalation` | Container | Permite `sudo` e `setuid` |
| `readOnlyRootFilesystem` | Container | Sistema de arquivos somente leitura |
| `privileged` | Container | Acesso completo ao host (evitar!) |
| `capabilities.add` | Container | Adicionar capabilities Linux |
| `capabilities.drop` | Container | Remover capabilities Linux |

---

## Exemplo Completo — Segurança em Produção

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: app
    image: minha-app:v1
    securityContext:
      allowPrivilegeEscalation: false
      readOnlyRootFilesystem: true
      capabilities:
        drop: ["ALL"]
    volumeMounts:
    - name: tmp
      mountPath: /tmp        # diretório gravável necessário
  volumes:
  - name: tmp
    emptyDir: {}
```

---

## Verificar usuário de um container

```bash
# Verificar qual usuário está rodando o processo
kubectl exec <pod> -- whoami
kubectl exec <pod> -- id
```

---

## Referências

- https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
- https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#securitycontext-v1-core
