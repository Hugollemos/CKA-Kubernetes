# Rolling Updates and Rollbacks

## 📋 O que são Rolling Updates?

**Rolling Updates** é a estratégia padrão do Kubernetes para atualizar aplicações sem downtime. Os pods são atualizados gradualmente, garantindo que sempre haja pods disponíveis para servir requisições.

### Características:
- ✅ **Zero downtime**: Aplicação continua disponível durante a atualização
- ✅ **Gradual**: Pods são atualizados em lotes, não todos de uma vez
- ✅ **Reversível**: Pode fazer rollback se algo der errado
- ✅ **Controlado**: Define quantos pods podem estar indisponíveis
- ✅ **Automático**: Kubernetes gerencia o processo

## 🔄 Como Funciona um Rolling Update?

```
Estado Inicial:
┌─────────────────────────────────────────┐
│ Deployment: nginx (image: nginx:1.20)  │
│ ReplicaSet-v1: 3 pods                  │
└─────────────────────────────────────────┘
    Pod-v1-A  Pod-v1-B  Pod-v1-C
    nginx:1.20 nginx:1.20 nginx:1.20

Comando de Update:
kubectl set image deployment/nginx nginx=nginx:1.21

Fase 1: Criar novo ReplicaSet
┌─────────────────────────────────────────┐
│ Deployment: nginx (image: nginx:1.21)  │
│ ReplicaSet-v1: 3 pods (old)            │
│ ReplicaSet-v2: 0 pods (new)            │
└─────────────────────────────────────────┘

Fase 2: Escalar novo, desescalar velho
┌─────────────────────────────────────────┐
│ ReplicaSet-v1: 2 pods (old)            │
│ ReplicaSet-v2: 1 pod (new)             │
└─────────────────────────────────────────┘
    Pod-v1-A  Pod-v1-B  Pod-v2-A
    nginx:1.20 nginx:1.20 nginx:1.21

Fase 3: Continuar o processo
┌─────────────────────────────────────────┐
│ ReplicaSet-v1: 1 pod (old)             │
│ ReplicaSet-v2: 2 pods (new)            │
└─────────────────────────────────────────┘
    Pod-v1-A  Pod-v2-A  Pod-v2-B
    nginx:1.20 nginx:1.21 nginx:1.21

Fase 4: Completar
┌─────────────────────────────────────────┐
│ ReplicaSet-v1: 0 pods (old)            │
│ ReplicaSet-v2: 3 pods (new)            │
└─────────────────────────────────────────┘
    Pod-v2-A  Pod-v2-B  Pod-v2-C
    nginx:1.21 nginx:1.21 nginx:1.21
```

## 🚀 Realizando Rolling Updates

### Método 1: kubectl set image (mais rápido)

```bash
# Atualizar imagem de um deployment
kubectl set image deployment/nginx nginx=nginx:1.21

# Atualizar múltiplos containers
kubectl set image deployment/app \
  nginx=nginx:1.21 \
  redis=redis:7.0

# Atualizar com record (salva no histórico)
kubectl set image deployment/nginx nginx=nginx:1.21 --record

# Ver status do rollout
kubectl rollout status deployment/nginx
```

### Método 2: kubectl edit

```bash
# Editar deployment diretamente
kubectl edit deployment nginx

# Alterar a versão da imagem:
# spec:
#   template:
#     spec:
#       containers:
#       - name: nginx
#         image: nginx:1.21  # ← Alterar aqui

# Salvar e sair (:wq) - rollout inicia automaticamente
```

### Método 3: kubectl apply (declarativo)

```bash
# Editar arquivo YAML
cat deployment.yaml
# Alterar a versão da imagem no arquivo

# Aplicar mudanças
kubectl apply -f deployment.yaml

# Ver status
kubectl rollout status deployment/nginx
```

### Método 4: kubectl patch

```bash
# Atualizar via patch
kubectl patch deployment nginx -p \
  '{"spec":{"template":{"spec":{"containers":[{"name":"nginx","image":"nginx:1.21"}]}}}}'

# Ou via JSON file
cat patch.json
{
  "spec": {
    "template": {
      "spec": {
        "containers": [
          {
            "name": "nginx",
            "image": "nginx:1.21"
          }
        ]
      }
    }
  }
}

kubectl patch deployment nginx --patch-file patch.json
```

## 📊 Estratégias de Update

### RollingUpdate (Padrão)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 10
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 2      # Máximo de pods que podem estar indisponíveis
      maxSurge: 2            # Máximo de pods extras além do desired
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
```

**Parâmetros:**

**maxUnavailable**
- Número ou % de pods que podem estar indisponíveis durante update
- Exemplo: `maxUnavailable: 2` = No máximo 2 pods down ao mesmo tempo
- Padrão: 25%

**maxSurge**
- Número ou % de pods extras que podem ser criados além do desejado
- Exemplo: `maxSurge: 2` = Pode ter até 12 pods (10 + 2) temporariamente
- Padrão: 25%

### Recreate (Downtime)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  strategy:
    type: Recreate    # ← Deleta todos os pods antes de criar novos
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
```

**Comportamento:**
1. Deleta todos os pods da versão antiga
2. Aguarda todos serem terminados
3. Cria todos os pods da versão nova

**Quando usar:**
- Quando a aplicação não pode rodar versões diferentes simultaneamente
- Quando há incompatibilidade de schema de banco de dados
- Quando não há problema com downtime

## 📈 Monitorando Rolling Updates

### Ver status do rollout

```bash
# Ver progresso do rollout em tempo real
kubectl rollout status deployment/nginx

# Output:
# Waiting for deployment "nginx" rollout to finish: 2 out of 3 new replicas have been updated...
# Waiting for deployment "nginx" rollout to finish: 2 of 3 updated replicas are available...
# deployment "nginx" successfully rolled out

# Ver status com watch (atualiza automaticamente)
watch kubectl get pods

# Ver eventos
kubectl get events --sort-by='.lastTimestamp' | grep nginx
```

### Ver detalhes do deployment

```bash
# Ver informações gerais
kubectl get deployment nginx

# Output:
# NAME    READY   UP-TO-DATE   AVAILABLE   AGE
# nginx   3/3     3            3           5m

# Ver detalhes completos
kubectl describe deployment nginx

# Ver ReplicaSets (old e new)
kubectl get rs

# Output:
# NAME               DESIRED   CURRENT   READY   AGE
# nginx-7d8b49b9f    0         0         0       10m  ← Old ReplicaSet
# nginx-6d4cf56db9   3         3         3       2m   ← New ReplicaSet

# Ver pods por ReplicaSet
kubectl get pods --show-labels
```

## ⏸️ Pausar e Retomar Rollouts

### Pausar um rollout

```bash
# Pausar rollout em andamento
kubectl rollout pause deployment/nginx

# Fazer múltiplas mudanças enquanto pausado
kubectl set image deployment/nginx nginx=nginx:1.21
kubectl set resources deployment/nginx -c nginx --limits=cpu=200m,memory=512Mi

# Retomar rollout (aplica todas as mudanças de uma vez)
kubectl rollout resume deployment/nginx
```

**Uso prático:**
- Fazer múltiplas alterações sem disparar vários rollouts
- Mais eficiente: um único rollout com todas as mudanças

## 🔙 Rollbacks (Reverter Atualizações)

### Histórico de Rollouts

```bash
# Ver histórico de revisões
kubectl rollout history deployment/nginx

# Output:
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         kubectl set image deployment/nginx nginx=nginx:1.21 --record=true
# 3         kubectl set image deployment/nginx nginx=nginx:1.22 --record=true

# Ver detalhes de uma revisão específica
kubectl rollout history deployment/nginx --revision=2

# Output:
# deployment.apps/nginx with revision #2
# Pod Template:
#   Labels:       app=nginx
#   Containers:
#    nginx:
#     Image:      nginx:1.21
#     ...
```

### Fazer Rollback

```bash
# Rollback para revisão anterior (undo)
kubectl rollout undo deployment/nginx

# Rollback para uma revisão específica
kubectl rollout undo deployment/nginx --to-revision=2

# Ver status do rollback
kubectl rollout status deployment/nginx
```

### Exemplo Completo de Rollback

```bash
# 1. Deployment inicial
kubectl create deployment nginx --image=nginx:1.20 --replicas=3

# 2. Atualização 1 (com record para aparecer no histórico)
kubectl set image deployment/nginx nginx=nginx:1.21 --record

# 3. Atualização 2 (com bug!)
kubectl set image deployment/nginx nginx=nginx:1.99-buggy --record

# 4. Ver que os pods estão falhando
kubectl get pods
# NAME                     READY   STATUS             RESTARTS   AGE
# nginx-xxx                0/1     ImagePullBackOff   0          30s

# 5. Ver histórico
kubectl rollout history deployment/nginx

# 6. Fazer rollback para versão anterior
kubectl rollout undo deployment/nginx

# 7. Verificar que voltou ao normal
kubectl get pods
# NAME                     READY   STATUS    RESTARTS   AGE
# nginx-xxx                1/1     Running   0          10s
```

## ⚙️ Configurações Avançadas

### minReadySeconds

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  minReadySeconds: 10    # Aguarda 10s antes de considerar pod pronto
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 1
      maxSurge: 1
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
```

**Uso:**
- Aguarda X segundos após pod estar "Ready" antes de continuar
- Útil para garantir que pod está realmente estável
- Previne rollouts muito rápidos que podem não detectar problemas

### progressDeadlineSeconds

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  progressDeadlineSeconds: 600    # 10 minutos timeout
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
```

**Uso:**
- Timeout para o rollout completar
- Se não completar em X segundos, marca como "failed"
- Padrão: 600 segundos (10 minutos)

### revisionHistoryLimit

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  revisionHistoryLimit: 5    # Manter apenas 5 ReplicaSets antigos
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.20
```

**Uso:**
- Limita número de ReplicaSets antigos mantidos
- Economiza recursos do cluster
- Padrão: 10

## 🧪 Cenários Práticos

### Cenário 1: Update com Canary Deployment (manual)

```bash
# 1. Deployment inicial
kubectl create deployment nginx --image=nginx:1.20 --replicas=10

# 2. Pausar autoupdate
kubectl rollout pause deployment/nginx

# 3. Atualizar imagem (mas ainda pausado)
kubectl set image deployment/nginx nginx=nginx:1.21

# 4. Escalar para 1 replica da nova versão (canary)
kubectl scale deployment/nginx --replicas=11

# 5. Retomar (cria apenas 1 pod com nova versão)
kubectl rollout resume deployment/nginx

# 6. Monitorar a nova versão
kubectl get pods -l app=nginx -o wide

# 7. Se tudo ok, continuar o rollout completo
kubectl rollout resume deployment/nginx

# 8. Se problema, fazer rollback
kubectl rollout undo deployment/nginx
```

### Cenário 2: Blue-Green Deployment (manual)

```bash
# 1. Deployment "Blue" (versão atual)
kubectl create deployment nginx-blue --image=nginx:1.20 --replicas=3
kubectl expose deployment nginx-blue --port=80 --name=nginx-service

# 2. Criar deployment "Green" (nova versão)
kubectl create deployment nginx-green --image=nginx:1.21 --replicas=3

# 3. Testar deployment Green
kubectl port-forward deployment/nginx-green 8080:80

# 4. Se tudo ok, mudar o service para Green
kubectl set selector service/nginx-service app=nginx-green

# 5. Deletar Blue quando tiver certeza
kubectl delete deployment nginx-blue
```

### Cenário 3: Rollback após detecção de erro

```bash
# 1. Deploy funcionando
kubectl get deployment nginx
# nginx   3/3     3            3           10m

# 2. Atualizar para versão com bug
kubectl set image deployment/nginx nginx=nginx:bad-version --record

# 3. Monitorar - ver que está falhando
kubectl rollout status deployment/nginx
# Waiting for deployment "nginx" rollout to finish: 1 out of 3 new replicas have been updated...
# (stuck here)

# 4. Ver pods
kubectl get pods
# nginx-xxx   0/1   ImagePullBackOff

# 5. Rollback imediatamente
kubectl rollout undo deployment/nginx

# 6. Verificar que voltou ao normal
kubectl rollout status deployment/nginx
# deployment "nginx" successfully rolled out
```

## 🔍 Troubleshooting de Rollouts

### Rollout travado (stuck)

```bash
# Ver status
kubectl rollout status deployment/nginx

# Ver eventos
kubectl describe deployment nginx | grep -A 20 "Events:"

# Ver pods
kubectl get pods

# Causas comuns:
# 1. ImagePullBackOff - imagem não existe
kubectl describe pod <pod-name> | grep -i "image"

# 2. CrashLoopBackOff - aplicação falhando
kubectl logs <pod-name>

# 3. Insufficient resources - sem recursos
kubectl describe node | grep -A 5 "Allocated resources"

# 4. Readiness probe failing
kubectl describe pod <pod-name> | grep -i "readiness"
```

### Rollback não funciona

```bash
# Ver se há histórico de revisões
kubectl rollout history deployment/nginx

# Se não houver histórico:
# - Deployment foi criado sem --record
# - revisionHistoryLimit muito baixo

# Solução: fazer update manual para versão conhecida
kubectl set image deployment/nginx nginx=nginx:1.20
```

### Rollout muito lento

```bash
# Ver configuração do deployment
kubectl get deployment nginx -o yaml | grep -A 10 "strategy"

# Possíveis problemas:
# - maxUnavailable muito baixo (ex: 1)
# - minReadySeconds muito alto
# - Readiness probe com delay alto

# Ajustar:
kubectl patch deployment nginx -p \
  '{"spec":{"strategy":{"rollingUpdate":{"maxUnavailable":2,"maxSurge":2}}}}'
```

## 📚 Comandos Úteis - Resumo

### Realizar Updates

```bash
# Atualizar imagem
kubectl set image deployment/<name> <container>=<image>

# Editar deployment
kubectl edit deployment <name>

# Aplicar arquivo YAML
kubectl apply -f deployment.yaml

# Escalar replicas
kubectl scale deployment/<name> --replicas=5
```

### Monitorar Updates

```bash
# Ver status do rollout
kubectl rollout status deployment/<name>

# Ver histórico
kubectl rollout history deployment/<name>

# Ver detalhes de uma revisão
kubectl rollout history deployment/<name> --revision=2

# Ver ReplicaSets
kubectl get rs

# Ver pods
kubectl get pods -l app=<label>
```

### Controlar Rollouts

```bash
# Pausar rollout
kubectl rollout pause deployment/<name>

# Retomar rollout
kubectl rollout resume deployment/<name>

# Fazer rollback
kubectl rollout undo deployment/<name>

# Rollback para revisão específica
kubectl rollout undo deployment/<name> --to-revision=2
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **Realizar um rolling update**
   ```bash
   kubectl set image deployment/nginx nginx=nginx:1.21
   kubectl rollout status deployment/nginx
   ```

2. **Fazer rollback**
   ```bash
   kubectl rollout undo deployment/nginx
   ```

3. **Ver histórico de revisões**
   ```bash
   kubectl rollout history deployment/nginx
   kubectl rollout history deployment/nginx --revision=2
   ```

4. **Entender maxUnavailable e maxSurge**
   - maxUnavailable: máximo de pods down
   - maxSurge: máximo de pods extras

5. **Pausar e retomar rollouts**
   ```bash
   kubectl rollout pause deployment/nginx
   kubectl rollout resume deployment/nginx
   ```

6. **Troubleshoot rollouts travados**
   ```bash
   kubectl describe deployment <name>
   kubectl get pods
   kubectl logs <pod-name>
   ```

### 🧪 Cenários típicos na prova:

> **"Atualize o deployment 'webapp' para usar a imagem 'webapp:v2'. Se houver problemas, reverta para a versão anterior."**

```bash
# 1. Atualizar
kubectl set image deployment/webapp webapp=webapp:v2 --record

# 2. Monitorar
kubectl rollout status deployment/webapp

# 3. Se houver problema, fazer rollback
kubectl rollout undo deployment/webapp

# 4. Verificar
kubectl rollout status deployment/webapp
```

> **"O deployment 'api' está travado durante um update. Investigue e resolva o problema."**

```bash
# 1. Ver status
kubectl rollout status deployment/api

# 2. Ver pods
kubectl get pods -l app=api

# 3. Descrever pod com problema
kubectl describe pod <pod-name>

# 4. Ver logs
kubectl logs <pod-name>

# 5. Se for problema com imagem, fazer rollback
kubectl rollout undo deployment/api
```

## 💡 Dicas para a Prova

1. **Use --record para histórico**
   ```bash
   kubectl set image deployment/nginx nginx=nginx:1.21 --record
   ```
   Aparece em `kubectl rollout history`

2. **Rollout status é seu amigo**
   ```bash
   kubectl rollout status deployment/nginx
   ```
   Mostra progresso em tempo real

3. **Rollback é rápido**
   ```bash
   kubectl rollout undo deployment/nginx
   ```
   Não precisa especificar versão se quer voltar apenas uma

4. **Ver ReplicaSets ajuda**
   ```bash
   kubectl get rs
   ```
   Mostra versões old e new durante rollout

5. **Describe mostra eventos úteis**
   ```bash
   kubectl describe deployment nginx
   ```
   Ver últimos eventos ajuda no troubleshooting

---

⬅️ **Anterior**: [replicaset-deployments.md](./replicaset-deployments.md) | ➡️ **Próximo**: [daemonsets.md](./daemonsets.md)
