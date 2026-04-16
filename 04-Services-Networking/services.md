# Services no Kubernetes - Resumo para Estudos CKA

## O que é um Service?

Um **Service** é um objeto que define como acessar Pods. Ele fornece uma abstração estável (IP e DNS) para um conjunto de Pods efêmeros. Sem Services, você não conseguiria acessar seus Pods de forma confiável, pois eles aparecem e desaparecem constantemente.

## Por Que Precisa de Services?

```
Problema:
- Pods têm IPs temporários
- Pods morrem e novos nascem com IPs diferentes
- Pods estão distribuídos em múltiplos nós
- Como conectar entre aplicações?

Solução:
- Service fornece IP fixo (Virtual IP - VIP)
- Service fornece DNS estável (nome.namespace.svc.cluster.local)
- Service distribui tráfego entre Pods (load balancing)
- Service abstrai os detalhes dos Pods

```

---

## Tipos de Services

### 1. ClusterIP (Padrão)

Expõe o Service apenas dentro do cluster. Outros Pods podem acessar via IP ou DNS, mas nada de fora do cluster consegue acessar.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP  # padrão, pode omitir
  selector:
    app: nginx      # seleciona Pods com esta label
  ports:
  - protocol: TCP
    port: 80        # porta do Service
    targetPort: 80  # porta do Pod

```

**Acesso:**

```bash
# De dentro do cluster
curl nginx-service         # mesmo namespace
curl nginx-service.default # namespace explícito
curl nginx-service.default.svc.cluster.local  # FQDN completo

# De outro namespace
curl nginx-service.default.svc.cluster.local

```

**Use quando:**

- Backend communication (Pod → Pod)
- Databases, caches, APIs internas
- Não precisa de acesso externo

---

### 2. NodePort

Expõe o Service em uma porta fixa em todos os nós do cluster. Tráfego externo pode acessar via `<node-ip>:<nodeport>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80        # porta interna do Service
    targetPort: 80  # porta do Pod
    nodePort: 30008 # porta no nó (30000-32767)
                    # se omitir, Kubernetes escolhe

```

**Fluxo de tráfego:**

```
Cliente externo (30.0.0.1:30008)
    ↓
Qualquer nó do cluster (node-1, node-2, node-3)
    ↓
Service NodePort
    ↓
Pods selecionados (pode ser em qualquer nó)

```

**Acesso:**

```bash
# De fora do cluster
curl node-1-ip:30008
curl node-2-ip:30008
curl node-3-ip:30008

# Qualquer nó funciona, mesmo se Pod não está lá
# (Kubernetes redireciona automaticamente)

```

**Use quando:**

- Precisa acessar serviço de fora do cluster
- Desenvolvimento/teste
- Sem load balancer externo disponível
- Porta fixa é importante

**Limitações:**

- Portas altas (30000-32767)
- Sem load balancing externo
- Expõe IPs dos nós

---

### 3. LoadBalancer

Expõe o Service via load balancer externo (AWS ELB, GCP Load Balancer, Azure LB, etc).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30008  # opcional

```

**Fluxo de tráfego:**

```
Cliente externo (internet)
    ↓
Load Balancer Externo (tem IP público)
    ↓
Qualquer nó do cluster
    ↓
Service LoadBalancer
    ↓
Pods selecionados

```

**Após criar:**

```bash
kubectl get svc nginx-service

# Output:
# NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
# nginx-service   LoadBalancer   10.0.1.100      a1b2c3d4e5f6.elb.amazonaws.com   80:30008/TCP

```

**Acesso:**

```bash
# Via LoadBalancer (recomendado)
curl a1b2c3d4e5f6.elb.amazonaws.com:80

# Via qualquer nó (também funciona)
curl node-ip:30008

```

**Use quando:**

- Produção com load balancer disponível
- Precisa de IP externo único
- Quer distribuição de tráfego externo
- Serviço é público/principal

**Custo:** Cada LoadBalancer é um recurso separado (pode ser caro)

---

### 4. ExternalName

Cria um alias DNS para um serviço externo. Redireciona tráfego para um serviço fora do cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com  # FQDN externo
  port: 5432

```

**Uso:**

```bash
# De dentro do cluster
kubectl run psql --image=postgres --rm -it \
  -- psql -h external-db -U user

# Internamente resolve para database.example.com

```

**Use quando:**

- Precisa conectar em serviços externos
- Quer usar DNS interno do Kubernetes
- Migrando de sistema externo para Kubernetes
- Abstratir localização do serviço externo

---

## Definição Completa de um Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
  labels:
    app: web
  annotations:
    description: "Service para aplicação web"
spec:
  type: ClusterIP
  clusterIP: 10.0.1.100  # IP específico (geralmente auto)
  selector:
    app: webapp           # seleciona Pods
    tier: frontend
  ports:
  - name: http           # nome da porta (opcional)
    protocol: TCP
    port: 80             # porta do Service
    targetPort: 8080     # porta do Pod
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  sessionAffinity: None   # ou ClientIP para sticky sessions
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600

```

---

## Service Discovery (Como Pods Encontram Services)

### Método 1: DNS (Recomendado)

```bash
# Dentro do cluster
curl http-service           # mesmo namespace
curl http-service.default   # outro namespace
curl http-service.default.svc.cluster.local  # FQDN

```

**DNS Format:**

```
<service-name>.<namespace>.svc.cluster.local

```

### Método 2: Environment Variables

```bash
# Kubernetes injeta automaticamente
# No Pod:
echo $NGINX_SERVICE_HOST      # IP do Service
echo $NGINX_SERVICE_PORT      # Porta do Service

# Convenção:
# <SERVICE_NAME>_SERVICE_HOST
# <SERVICE_NAME>_SERVICE_PORT

```

### Método 3: Selecionar por IP

```bash
# Não recomendado - mude para DNS se possível
curl 10.0.1.100:80

```

---

## Endpoints e seletor de Service

Um **Endpoint** conecta o Service aos Pods reais.

```bash
# Ver endpoints de um Service
kubectl get endpoints
kubectl get ep

# Descrever um Service (mostra endpoints)
kubectl describe svc nginx-service
# Endpoints: 10.244.1.10:80,10.244.1.11:80,10.244.1.12:80

# Os Pods com label app=nginx têm esses IPs
# Se adicionar/remover Pod, endpoints atualizam automaticamente

```

### Service sem Selector

```yaml
# Service sem selector (gerencia Endpoints manualmente)
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
---
# Endpoint manual
apiVersion: v1
kind: Endpoints
metadata:
  name: external-api
subsets:
- addresses:
  - ip: 192.168.1.100
  ports:
  - port: 8080

```

**Use quando:**

- Conectar em sistema legado/externo
- Controle total sobre Endpoints
- Múltiplos backends que não são Pods

---

## Comandos Essenciais

```bash
# Criar Service
kubectl create service clusterip web --tcp=80:8080
kubectl create service nodeport web --tcp=80:8080 --node-port=30008
kubectl create service loadbalancer web --tcp=80:8080

# Ou via arquivo YAML
kubectl create -f service.yaml
kubectl apply -f service.yaml

# Listar Services
kubectl get services
kubectl get svc
kubectl get svc -n default
kubectl get svc -o wide

# Descrever Service (muito importante!)
kubectl describe svc nginx-service

# Ver Endpoints
kubectl get endpoints
kubectl get ep
kubectl describe ep nginx-service

# Editar Service
kubectl edit svc nginx-service

# Expor um Deployment como Service
kubectl expose deployment nginx-dep --type=ClusterIP --port=80 --target-port=8080
kubectl expose deployment nginx-dep --type=NodePort --port=80 --target-port=8080

# Deletar Service
kubectl delete svc nginx-service
kubectl delete -f service.yaml

# Testar conectividade
kubectl run -it --rm debug --image=busybox -- wget -O- http://nginx-service:80

# Port forward (acesso local a um Service)
kubectl port-forward svc/nginx-service 8080:80
# Agora acessa via localhost:8080

```

---

## Estratégias de Load Balancing

### 1. Round-Robin (Padrão)

Distribui tráfego igualmente entre Pods.

```
Requisição 1 → Pod A
Requisição 2 → Pod B
Requisição 3 → Pod C
Requisição 4 → Pod A
Requisição 5 → Pod B
...

```

### 2. Session Affinity (Sticky Sessions)

Mantém cliente conectado ao mesmo Pod.

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # reset após 1 hora

```

**Use quando:**

- Aplicação precisa de estado (sessão)
- Evitar compartilhamento de sessão entre Pods

---

## Service Policy: externalTrafficPolicy

Controla como tráfego externo é roteado.

### Local (Sem saltos)

```yaml
spec:
  type: NodePort
  externalTrafficPolicy: Local  # rota para Pods no mesmo nó

```

```
Cliente → Node A:30008
    ↓
Se Pod está em Node A: acessa direto
Se Pod está em outro nó: conexão rejeitada/sem resposta

```

**Vantagem:** Preserva IP do cliente (source IP real)
**Desvantagem:** Distribuição desigual entre nós

### Cluster (Padrão)

```yaml
spec:
  type: NodePort
  externalTrafficPolicy: Cluster  # pode pular para outro nó

```

```
Cliente → Node A:30008
    ↓
Se Pod está em Node A: acessa direto
Se Pod está em outro nó: rota para lá (hop extra)

```

**Vantagem:** Distribuição igual entre Pods
**Desvantagem:** Source IP é mascarado (não é o cliente real)

---

## Comparação de Tipos de Services

| Tipo | Acesso Interno | Acesso Externo | IP Fixo | DNS | Use Case |
| --- | --- | --- | --- | --- | --- |
| **ClusterIP** | ✅ Sim | ❌ Não | ✅ Sim | ✅ Sim | Backend, APIs internas |
| **NodePort** | ✅ Sim | ✅ Sim (nó) | ✅ Sim | ✅ Sim | Dev/test, sem LB |
| **LoadBalancer** | ✅ Sim | ✅ Sim (LB) | ✅ Sim | ✅ Sim | Produção, serviços públicos |
| **ExternalName** | ✅ Sim | ⚠️ Via externo | ⚠️ Não | ✅ Sim | Redirect externo |

---

## Troubleshooting Services

### Service criado mas não funciona?

```bash
# 1. Verificar se Service existe
kubectl get svc nginx-service
kubectl describe svc nginx-service

# 2. Verificar Endpoints (Pods selecionados)
kubectl get ep nginx-service
# Se vazio: selector não match com nenhum Pod

# 3. Verificar labels dos Pods
kubectl get pods --show-labels
# Compare com selector do Service

# 4. Testar dentro do cluster
kubectl run -it --rm test --image=busybox -- sh
$ wget -O- http://nginx-service
$ nslookup nginx-service

# 5. Verificar iptables (kube-proxy)
kubectl describe node node-name
# Procure por portas e regras

# 6. Ver logs do kube-proxy
kubectl logs -n kube-system -l component=kube-proxy

# 7. Verificar conectividade de Pod para Pod
kubectl exec -it pod-name -- sh
$ wget http://nginx-service

```

### NodePort não responde?

```bash
# 1. Verificar porta
kubectl get svc nginx-service
# Procure por node-port na saída

# 2. Testar de dentro do cluster primeiro
kubectl run -it --rm test --image=busybox -- wget -O- http://nginx-service

# 3. Testar via IP do nó
kubectl get nodes -o wide
# Use NODE-EXTERNAL-IP:nodeport

# 4. Verificar firewall
# Porta nodeport (30000-32767) deve estar aberta

# 5. Verificar se Pod está rodando
kubectl get pods -o wide
# Procure por Running e Ready

```

### LoadBalancer não recebe IP externo?

```bash
# Pode levar alguns minutos
kubectl get svc -w
# Aguarde EXTERNAL-IP aparecer

# Se ficar como <pending>:
kubectl describe svc nginx-service
# Procure por Events e mensagens de erro

# Provedor de nuvem pode não estar configurado
# (verificar credenciais do cloud controller)

```

---

## Casos de Uso Comuns

### Caso 1: API Backend (ClusterIP)

```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: api:v1
        ports:
        - containerPort: 5000
---
# Service ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5000

```

**Acesso de outro Pod:**

```bash
curl http://api-service:80

```

---

### Caso 2: Aplicação Web Pública (LoadBalancer)

```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
# Service LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80

```

**Acesso externo:**

```bash
curl a1b2c3d4e5f6.elb.amazonaws.com

```

---

### Caso 3: Teste Local (NodePort)

```bash
# Criar deployment
kubectl create deployment test --image=nginx --replicas=2

# Expor como NodePort
kubectl expose deployment test --type=NodePort --port=80 --target-port=80

# Obter informações
kubectl get svc test

# Acessar
curl node-ip:node-port

```

---

## Pontos Importantes para Prova CKA

✅ **ClusterIP é o padrão** - use quando service é interno
✅ **Selector em Service** - deve match com labels dos Pods
✅ **Endpoints** - criados automaticamente quando selector match
✅ **port vs targetPort** - port é do Service, targetPort é do Pod
✅ **nodePort range** - 30000-32767
✅ **DNS interno** - `<service>.<namespace>.svc.cluster.local`
✅ **LoadBalancer** - precisa de cloud provider configurado
✅ **ExternalName** - redireciona para serviço externo
✅ **sessionAffinity** - sticky sessions com ClientIP
✅ **Verificar Endpoints** - se vazio, selector não match
✅ **kubectl expose** - cria Service de um Deployment
✅ **service discovery** - DNS é mais confiável que env vars
✅ **Load balancing padrão** - round-robin entre Pods
✅ **externalTrafficPolicy: Local** - sem hop extra, preserva source IP

---

## Quick Reference: Criar Services

```bash
# ClusterIP (padrão)
kubectl expose deployment nginx --type=ClusterIP --port=80

# NodePort
kubectl expose deployment nginx --type=NodePort --port=80 --node-port=30008

# LoadBalancer
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Com YAML
kubectl create -f service.yaml
kubectl apply -f service.yaml

# Descrever para debugging
kubectl describe svc nginx

```

---

## DNS Resolution Chain

```
Pod tenta: curl nginx-service

1. Procura em /etc/resolv.conf do Pod
   search default.svc.cluster.local svc.cluster.local cluster.local
   nameserver 10.0.0.10  (coredns)

2. Tenta: nginx-service.default.svc.cluster.local

3. CoreDNS resolve para IP do Service
   nginx-service → 10.0.1.100

4. Tráfego vai para 10.0.1.100:80
   (kube-proxy redireciona para Pods reais)

Se estiver em outro namespace:
curl nginx-service.other-namespace.svc.cluster.local

```

---

