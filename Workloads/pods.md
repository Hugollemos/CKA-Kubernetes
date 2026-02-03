# Pod no Kubernetes - Resumo para Estudos CKA

## O que é um Pod?

Um **Pod** é a menor unidade implantável no Kubernetes. É um wrapper ao redor de um ou mais containers que rodam juntos na mesma rede.

## Características Principais

**Containers por Pod**

- Geralmente um container por Pod (padrão recomendado)
- Pode ter múltiplos containers, mas eles compartilham recursos
- Containers no mesmo Pod se comunicam via `localhost`

**Networking**

- Cada Pod tem um IP único
- Compartilham o mesmo namespace de rede
- Comunicação via localhost entre containers

**Storage**

- Volumes podem ser montados em múltiplos containers
- Dados persistem enquanto o Pod existe

## Definição em YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: meu-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    env:
    - name: ENV_VAR
      value: "valor"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: meu-config

```

## Ciclo de Vida de um Pod

| Estado | Descrição |
| --- | --- |
| **Pending** | Pod foi criado mas não está rodando (esperando recursos) |
| **Running** | Containers estão rodando |
| **Succeeded** | Pod completou com sucesso (batch jobs) |
| **Failed** | Um ou mais containers falharam |
| **Unknown** | Estado desconhecido |

## Comandos Essenciais para CKA

```bash
# Criar um Pod
kubectl run nginx --image=nginx

# Criar e gerar YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml

# Listar Pods
kubectl get pods
kubectl get pods -n default
kubectl get pods -o wide  # mostra IPs e nós

# Descrever um Pod
kubectl describe pod nginx

# Ver logs
kubectl logs nginx
kubectl logs nginx -c container-name  # múltiplos containers

# Executar comando no Pod
kubectl exec -it nginx -- /bin/bash

# Editar Pod em tempo real
kubectl edit pod nginx

# Deletar Pod
kubectl delete pod nginx
kubectl delete pod nginx --grace-period=0 --force  # force delete

# Port forward
kubectl port-forward nginx 8080:80

```

## Multi-Container Pod

Usado quando containers precisam trabalhar juntos (sidecar pattern):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: app
    image: app:latest
    ports:
    - containerPort: 8080
  - name: sidecar
    image: sidecar:latest
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}

```

## Init Containers

Containers que rodam antes dos containers principais:

```yaml
spec:
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'echo Inicializando DB']
  containers:
  - name: app
    image: app:latest

```

## Resources (CPU/Memory)

```yaml
resources:
  requests:  # mínimo garantido
    memory: "64Mi"
    cpu: "250m"
  limits:    # máximo permitido
    memory: "128Mi"
    cpu: "500m"

```

## Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

```

---

