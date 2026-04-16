# ReplicaSet, ReplicaSet Controller e Deployments - Resumo para Estudos CKA

## Hierarquia de Objetos

```
Deployment (nível alto - recomendado)
    ↓
ReplicaSet (gerenciado pelo Deployment)
    ↓
Pods (gerenciados pelo ReplicaSet)

```

---

## ReplicaSet

### O que é?

Um **ReplicaSet** garante que um número especificado de réplicas de um Pod estejam rodando em qualquer momento. É o responsável por manter o número desejado de Pods.

### Definição em YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: meu-replicaset
  labels:
    app: nginx
spec:
  replicas: 3  # número de Pods desejados
  selector:
    matchLabels:
      app: nginx  # Pods com esta label serão gerenciados
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"

```

### Como Funciona

1. ReplicaSet monitora Pods com labels que correspondem ao `selector`
2. Se menos Pods existem que `replicas`: cria novos
3. Se mais Pods existem que `replicas`: deleta os extras
4. Se um Pod morre: cria um novo automaticamente
5. **Não faz rolling updates** (atualizações gradualmente)

### Seletor (Selector)

```yaml
selector:
  matchLabels:
    app: nginx
    version: v1

# Ou com matchExpressions (mais complexo)
selector:
  matchExpressions:
  - key: app
    operator: In
    values: [nginx, apache]
  - key: tier
    operator: NotIn
    values: [frontend]

```

### Comandos Essenciais

```bash
# Criar ReplicaSet
kubectl create -f replicaset.yaml

# Listar ReplicaSets
kubectl get replicaset
kubectl get rs  # forma curta

# Descrever ReplicaSet
kubectl describe rs meu-replicaset

# Escalar (mudar número de replicas)
kubectl scale rs meu-replicaset --replicas=5

# Editar ReplicaSet
kubectl edit rs meu-replicaset

# Deletar ReplicaSet (mantém Pods)
kubectl delete rs meu-replicaset
kubectl delete rs meu-replicaset --cascade=orphan

# Deletar ReplicaSet e Pods
kubectl delete rs meu-replicaset --cascade=background

```

---

## ReplicaSet Controller

### O que é?

O **ReplicaSet Controller** é um componente do Kubernetes (roda no control plane) que monitora continuamente os ReplicaSets e garante que o estado atual corresponda ao estado desejado.

### Responsabilidades

- **Monitorar Pods**: Verifica status de todos os Pods constantemente
- **Criar Pods**: Se houver menos Pods do que `replicas`, cria novos
- **Deletar Pods**: Se houver mais Pods, deleta o excesso
- **Recriar Pods**: Se um Pod falha, recria automaticamente
- **Lidar com Seletores**: Identifica quais Pods gerenciar via labels

### Como o Controller Sabe o que Fazer

```yaml
# O ReplicaSet define o estado desejado
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx

# O Controller:
# 1. Conta quantos Pods com label app=nginx existem
# 2. Compara com replicas: 3
# 3. Toma ação: criar/deletar Pods

```

### Fluxo de Reconciliação

```
Controller monitora
    ↓
Detecta mudança (Pod morreu, Deployment escalado, etc)
    ↓
Calcula: Pods atuais vs replicas desejadas
    ↓
Toma ação: criar ou deletar Pods
    ↓
Monitora novamente

```

### Status do ReplicaSet

```bash
kubectl describe rs meu-replicaset

# Saída:
# Replicas: 3 desired | 3 updated | 3 total | 3 available
#   desired: número definido no spec
#   updated: Pods com spec atualizado
#   total: número total de Pods
#   available: Pods prontos

```

---

## Deployments (RECOMENDADO!)

### O que é?

Um **Deployment** é um objeto de nível superior que gerencia ReplicaSets. Fornece atualizações declarativas para Pods e ReplicaSets com recursos como rolling updates, rollbacks e histórico.

### Quando Usar

- ✅ Quase sempre! (é a forma recomendada)
- ✅ Quando precisa de atualizações gradativas
- ✅ Quando quer manter histórico de versões
- ❌ Não use ReplicaSet diretamente na maioria dos casos

### Definição em YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate  # ou Recreate
    rollingUpdate:
      maxSurge: 1        # máximo de Pods além do desejado
      maxUnavailable: 1  # máximo de Pods indisponíveis
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      terminationGracePeriodSeconds: 30

```

### Estratégias de Deploy

### RollingUpdate (Padrão)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # máximo de Pods extras durante atualização
    maxUnavailable: 1     # máximo de Pods que podem estar down

```

Resultado: Pods são substituídos gradualmente, sem downtime

```
Atualização V1 → V2
V1: 3 Pods
→ Cria 1 Pod V2, deleta 1 Pod V1 (total: 3)
→ Cria 1 Pod V2, deleta 1 Pod V1 (total: 3)
→ Cria 1 Pod V2, deleta 1 Pod V1 (total: 3)
Resultado: 3 Pods V2

```

### Recreate

```yaml
strategy:
  type: Recreate

```

Resultado: Deleta todos os Pods antigos, depois cria novos (downtime!)

```
Atualização V1 → V2
→ Deleta todos os 3 Pods V1
→ Cria 3 Pods V2
(período com 0 Pods = DOWNTIME)

```

### Comandos Essenciais

```bash
# Criar Deployment
kubectl create -f deployment.yaml
kubectl create deployment nginx --image=nginx --replicas=3

# Listar Deployments
kubectl get deployments
kubectl get deploy

# Descrever Deployment
kubectl describe deploy nginx-deployment

# Ver status do rollout
kubectl rollout status deployment/nginx-deployment

# Ver histórico de rollouts
kubectl rollout history deployment/nginx-deployment

# Ver detalhes de uma versão específica
kubectl rollout history deployment/nginx-deployment --revision=2

# Escalar Deployment
kubectl scale deployment nginx-deployment --replicas=5

# Atualizar imagem (trigger de novo rollout)
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.16.0 --record

# Editar Deployment
kubectl edit deployment nginx-deployment

# Fazer rollback para versão anterior
kubectl rollout undo deployment/nginx-deployment

# Fazer rollback para versão específica
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pausar rollout
kubectl rollout pause deployment/nginx-deployment

# Resumir rollout
kubectl rollout resume deployment/nginx-deployment

# Deletar Deployment
kubectl delete deployment nginx-deployment

```

### Atualizando um Deployment

### Método 1: kubectl set image (Recomendado)

```bash
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.16.0 \
  --record

# Verificar status
kubectl rollout status deployment/nginx-deployment

```

### Método 2: Editar YAML

```bash
kubectl edit deployment nginx-deployment
# Muda a imagem no editor, salva e sai
# Kubernetes faz o deploy automaticamente

```

### Método 3: Aplicar novo arquivo

```bash
# Modifica deployment.yaml
kubectl apply -f deployment.yaml

```

### Histórico e Rollback

```bash
# Ver histórico
kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         kubectl create --record=true
# 2         kubectl set image deployment/nginx-deployment nginx=nginx:1.16.0

# Rollback para versão anterior
kubectl rollout undo deployment/nginx-deployment

# Rollback para versão específica
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Verificar mudanças
kubectl diff -f deployment.yaml

```

### Observar Mudanças em Tempo Real

```bash
# Watch dos Pods durante atualização
kubectl get pods -w

# Watch do Deployment
kubectl get deploy -w

# Logs de todos os Pods de um Deployment
kubectl logs -l app=nginx --tail=20 -f

```

---

## Comparação: ReplicaSet vs Deployment

| Aspecto | ReplicaSet | Deployment |
| --- | --- | --- |
| **Propósito** | Garante replicas | Gerencia ReplicaSets + updates |
| **Rolling Updates** | ❌ Não | ✅ Sim |
| **Rollback** | ❌ Não | ✅ Sim |
| **Histórico** | ❌ Não | ✅ Sim |
| **Escalabilidade** | ✅ Sim | ✅ Sim |
| **Recriar Pods** | ✅ Sim | ✅ Sim |
| **Uso em Produção** | ❌ Raramente | ✅ Sempre |
| **Quando Usar** | Casos especiais | 99% dos casos |

---

## Fluxo Completo: Deployment → ReplicaSet → Pods

```yaml
# 1. Você cria um Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14

```

```
O Kubernetes faz:

2. Deployment Controller cria ReplicaSet
   ReplicaSet: nginx-5d59d67564 (hash da spec)

3. ReplicaSet Controller cria 3 Pods
   Pod: nginx-5d59d67564-abc12
   Pod: nginx-5d59d67564-def45
   Pod: nginx-5d59d67564-ghi78

Você atualiza a imagem para nginx:1.16:

4. Deployment Controller cria novo ReplicaSet
   ReplicaSet: nginx-5d59d67564 (antiga)
   ReplicaSet: nginx-8f7b2c3d9e (nova)

5. Novo ReplicaSet cria Pods com nginx:1.16
   ReplicaSet nova: 1 Pod
   ReplicaSet antiga: 2 Pods (em destruição)

6. Continua até:
   ReplicaSet nova: 3 Pods
   ReplicaSet antiga: 0 Pods

7. Deployment mantém histórico:
   REVISION 1: ReplicaSet nginx-5d59d67564
   REVISION 2: ReplicaSet nginx-8f7b2c3d9e

Você faz rollback:

8. Deployment Controller recria ReplicaSet antiga
   ReplicaSet: nginx-5d59d67564 (volta a rodar)

9. Pods voltam para nginx:1.14

```

---

## Pontos Importantes para Prova CKA

✅ **Sempre use Deployment** - não ReplicaSet direto
✅ **RollingUpdate** é a estratégia padrão e recomendada
✅ **maxSurge** e **maxUnavailable** controlam a velocidade de atualização
✅ **--record** mantém histórico (use em set image)
✅ **kubectl rollout undo** desfaz atualizações
✅ **kubectl scale** muda número de replicas
✅ Deployment cria ReplicaSets automaticamente (não mexa neles)
✅ ReplicaSet Controller mantém o número exato de Pods rodando
✅ **Selector** usa labels para identificar quais Pods gerenciar
✅ Entender a diferença entre **desired**, **updated**, **total**, **available**
✅ Pods orphans podem ser adotados por novo ReplicaSet se labels correspondem
✅ **Recreate** causa downtime, evite em produção

---

## Casos de Uso Comuns

### Caso 1: Deploy Simples com 3 Replicas

```bash
kubectl create deployment web --image=nginx --replicas=3

```

### Caso 2: Atualizar Imagem

```bash
kubectl set image deployment/web nginx=nginx:1.20 --record
kubectl rollout status deployment/web

```

### Caso 3: Rollback se Algo Deu Errado

```bash
kubectl rollout undo deployment/web

```

### Caso 4: Escalar Rápido

```bash
kubectl scale deployment/web --replicas=10

```

### Caso 5: Pausar Atualização

```bash
kubectl rollout pause deployment/web
# Faz testes
kubectl rollout resume deployment/web

```

---

## Debug Comum

```bash
# Deployment não está fazendo rollout?
kubectl describe deploy nginx-deployment
# Verifique: replicas desejadas vs current

# Pods não estão criando?
kubectl describe rs nginx-deployment-XXX
# Verifique: events no final

# Imagem não atualizou?
kubectl get pods -o jsonpath='{.items[0].spec.containers[0].image}'

# Ver mudanças entre revisions
kubectl rollout history deployment/nginx --revision=1
kubectl rollout history deployment/nginx --revision=2

```

---

