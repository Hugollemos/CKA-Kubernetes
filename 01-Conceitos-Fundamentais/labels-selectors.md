# Labels e Selectors no Kubernetes

## O que são Labels e Selectors?

**Labels** são pares chave-valor anexados a objetos Kubernetes (Pods, Services, Deployments, etc.) para identificação e organização.

**Selectors** são filtros usados para selecionar objetos com base em suas labels.

---

## Como definir Labels em um Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: simple-webapp
  labels:
    app: App1
    function: front-end
    env: production
spec:
  containers:
  - name: simple-webapp
    image: simple-webapp
    ports:
    - containerPort: 8080
```

---

## Selecionar Pods por Label

```bash
# Selecionar pods com label específica
kubectl get pods --selector app=App1
kubectl get pods -l app=App1

# Múltiplos seletores (AND lógico)
kubectl get pods -l app=App1,env=production

# Ver labels dos pods
kubectl get pods --show-labels

# Filtrar todos os objetos
kubectl get all --selector app=App1
```

---

## Labels em ReplicaSet — Conexão entre objetos

O Kubernetes usa labels para conectar objetos. No ReplicaSet, existem **dois lugares** onde as labels aparecem:

1. `metadata.labels` — label do próprio ReplicaSet
2. `spec.template.metadata.labels` — labels dos Pods criados
3. `spec.selector.matchLabels` — define quais Pods o RS gerencia

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1              # label do próprio ReplicaSet
    function: front-end
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1            # seleciona Pods com essa label
  template:
    metadata:
      labels:
        app: App1          # label dos Pods criados
        function: front-end
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp
```

---

## Labels em Services

O Service usa `selector` para direcionar o tráfego para os Pods corretos:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: App1              # redireciona para Pods com essa label
  ports:
  - protocol: TCP
    port: 80
    targetPort: 9376
```

---

## Annotations

**Annotations** são similares às labels, mas usadas para armazenar **informações complementares** (não para seleção).

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: simple-webapp
  labels:
    app: App1
  annotations:
    buildversion: "1.34"
    owner: "time-backend"
    contact: "email@empresa.com"
spec:
  replicas: 3
  selector:
    matchLabels:
      app: App1
  template:
    metadata:
      labels:
        app: App1
    spec:
      containers:
      - name: simple-webapp
        image: simple-webapp
```

---

## Diferença: Labels vs Annotations

| | Labels | Annotations |
|--|--------|-------------|
| Propósito | Identificação e seleção | Informações complementares |
| Usado em selectors | Sim | Não |
| Consultável via kubectl | Sim (`-l`) | Não diretamente |
| Exemplos | `app=nginx`, `env=prod` | `buildVersion=1.2`, `owner=time-a` |

---

## Comandos Úteis

```bash
# Adicionar label a um recurso existente
kubectl label pod meu-pod env=production

# Remover label
kubectl label pod meu-pod env-

# Atualizar label
kubectl label pod meu-pod env=staging --overwrite

# Ver pods com labels
kubectl get pods --show-labels

# Filtrar por label
kubectl get pods -l 'env in (production, staging)'
kubectl get pods -l 'env notin (development)'
kubectl get pods -l env=production,app=nginx
```

---

## Referências

- https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/
- https://kubernetes.io/docs/concepts/overview/working-with-objects/annotations/
