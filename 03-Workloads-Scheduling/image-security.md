# Image Security — Segurança de Imagens

## Convenção de Nomes de Imagens

```
docker.io/library/nginx
│          │       │
│          │       └── nome da imagem
│          └────────── usuário/organização (library = imagens oficiais)
└───────────────────── registry (docker.io = Docker Hub)
```

Quando você escreve apenas `nginx`, o Kubernetes expande para:
```
docker.io/library/nginx:latest
```

---

## Registries Privados

Para usar imagens de um registro privado, é necessário autenticar.

### Passo 1 — Criar um Secret do tipo `docker-registry`

```bash
kubectl create secret docker-registry regcred \
  --docker-server=private-registry.io \
  --docker-username=usuario \
  --docker-password=senha123 \
  --docker-email=usuario@empresa.com
```

### Passo 2 — Referenciar o Secret no Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
spec:
  containers:
  - name: nginx
    image: private-registry.io/apps/internal-app
  imagePullSecrets:
  - name: regcred
```

---

## Usando Registries Privados em Deployments

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: minha-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: minha-app
  template:
    metadata:
      labels:
        app: minha-app
    spec:
      containers:
      - name: minha-app
        image: meu-registry.azurecr.io/minha-app:v1.2.3
      imagePullSecrets:
      - name: regcred-azure
```

---

## Registries Comuns

| Provedor | Endereço |
|----------|----------|
| Docker Hub | `docker.io` |
| Google Container Registry | `gcr.io` |
| Amazon ECR | `<account>.dkr.ecr.<region>.amazonaws.com` |
| Azure Container Registry | `<nome>.azurecr.io` |
| GitHub Container Registry | `ghcr.io` |

---

## Boas Práticas de Segurança

- **Sempre use tags específicas** — evitar `:latest` em produção
- **Escanear imagens** por vulnerabilidades antes do deploy
- **Usar registries privados** para imagens internas
- **Assinar imagens** com ferramentas como Cosign (Supply Chain Security)
- **ImagePullPolicy** controla quando a imagem é baixada:
  - `Always` — baixa sempre (padrão para `:latest`)
  - `IfNotPresent` — usa cache se existir
  - `Never` — nunca baixa (usa apenas cache local)

```yaml
spec:
  containers:
  - name: minha-app
    image: minha-app:v1.2.3
    imagePullPolicy: IfNotPresent  # recomendado para versões fixas
```

---

## Comandos Úteis

```bash
# Criar secret para registry privado
kubectl create secret docker-registry <nome-secret> \
  --docker-server=<servidor> \
  --docker-username=<usuario> \
  --docker-password=<senha> \
  --docker-email=<email>

# Ver secrets
kubectl get secrets
kubectl describe secret <nome-secret>

# Ver imagem sendo usada por um pod
kubectl get pod <nome-pod> -o jsonpath='{.spec.containers[*].image}'
```

---

## Referências

- https://kubernetes.io/docs/concepts/containers/images/
- https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/
