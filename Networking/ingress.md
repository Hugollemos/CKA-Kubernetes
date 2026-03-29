# Ingress no Kubernetes

Este documento cobre **Ingress**, um recurso do Kubernetes para expor aplicações HTTP/HTTPS para o mundo externo com roteamento avançado.

## 📚 Conteúdo

1. [O que é Ingress?](#o-que-é-ingress)
2. [Ingress vs Service](#ingress-vs-service)
3. [Ingress Controllers](#ingress-controllers)
4. [Ingress Resources](#ingress-resources)
5. [Path-Based Routing](#path-based-routing)
6. [Host-Based Routing](#host-based-routing)
7. [TLS/HTTPS](#tlshttps)
8. [Annotations](#annotations)
9. [Troubleshooting](#troubleshooting)

---

## 🌐 O que é Ingress?

**Ingress** é um recurso Kubernetes que gerencia **acesso externo** a serviços via **HTTP/HTTPS**, fornecendo:
- **Roteamento baseado em URL** (path-based routing)
- **Roteamento baseado em host** (host-based routing / virtual hosting)
- **Terminação TLS/SSL**
- **Load balancing**

```
                        Internet
                           │
                           ▼
                   ┌───────────────┐
                   │   Ingress     │
                   │ (Layer 7 LB)  │
                   └───────┬───────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
      ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
      │Service A│     │Service B│     │Service C│
      │ClusterIP│     │ClusterIP│     │ClusterIP│
      └────┬────┘     └────┬────┘     └────┬────┘
           │               │               │
      ┌────▼────┐     ┌────▼────┐     ┌────▼────┐
      │ Pod A   │     │ Pod B   │     │ Pod C   │
      └─────────┘     └─────────┘     └─────────┘
```

### Por que usar Ingress?

**Sem Ingress:**
- Precisa expor cada app via `LoadBalancer` ou `NodePort`
- Cada `LoadBalancer` Service custa dinheiro (IP público em cloud)
- Difícil gerenciar SSL/TLS para múltiplas apps
- Sem roteamento avançado

**Com Ingress:**
- ✅ **Um único ponto de entrada** para múltiplas apps
- ✅ **Roteamento por URL**: `/app1` → Service A, `/app2` → Service B
- ✅ **Roteamento por hostname**: `app1.com` → Service A, `app2.com` → Service B
- ✅ **SSL/TLS centralizado**
- ✅ **Custo reduzido** (apenas um LoadBalancer)

---

## 🔄 Ingress vs Service

### Service (LoadBalancer/NodePort)

**Vantagens:**
- Simples de configurar
- Funciona com TCP/UDP (não só HTTP)
- Não precisa de Ingress Controller

**Desvantagens:**
- Um IP/porta por Service
- Sem roteamento avançado
- Caro em cloud (cada LoadBalancer = custo)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
spec:
  type: LoadBalancer  # $$$ Custa dinheiro em cloud
  ports:
  - port: 80
  selector:
    app: my-app
```

### Ingress

**Vantagens:**
- Um único LoadBalancer para múltiplas apps
- Roteamento avançado (path/host-based)
- SSL/TLS centralizado
- Rewrite URLs, redirects, etc

**Desvantagens:**
- Apenas HTTP/HTTPS (Layer 7)
- Precisa de Ingress Controller instalado
- Configuração mais complexa

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-app
            port:
              number: 80
```

### Comparação

| Feature | NodePort | LoadBalancer | Ingress |
|---------|----------|--------------|---------|
| **Layer** | L4 (TCP/UDP) | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| **Custo** | Grátis | $$$ Alto | $ Baixo |
| **Path routing** | ❌ Não | ❌ Não | ✅ Sim |
| **Host routing** | ❌ Não | ❌ Não | ✅ Sim |
| **SSL/TLS** | ❌ Manual | ❌ Manual | ✅ Automático |
| **Múltiplas apps** | Difícil | Caro | ✅ Fácil |

---

## 🎛️ Ingress Controllers

**Ingress resource sozinho não faz nada!** Você precisa de um **Ingress Controller** rodando no cluster.

### O que é Ingress Controller?

**Ingress Controller** é um **Pod** que:
1. Monitora Ingress resources via API Server
2. Configura um **reverse proxy** (nginx, Traefik, HAProxy, etc)
3. Roteia tráfego HTTP/HTTPS para Services

```
┌──────────────────────────────────────────────────────────┐
│                    Kubernetes Cluster                    │
│                                                          │
│  ┌────────────────────────────────────────────────┐     │
│  │         Ingress Controller (Pod)               │     │
│  │                                                │     │
│  │  ┌──────────────────────────────────────┐     │     │
│  │  │  Nginx / Traefik / HAProxy           │     │     │
│  │  │  (Reverse Proxy)                     │     │     │
│  │  └──────────────────────────────────────┘     │     │
│  │                                                │     │
│  │  Monitora API Server ──────────────────────►  │     │
│  │  para Ingress resources                       │     │
│  └────────────────────────────────────────────────┘     │
│                                                          │
│  ┌────────────┐    ┌────────────┐    ┌────────────┐    │
│  │ Ingress 1  │    │ Ingress 2  │    │ Ingress 3  │    │
│  │ (Resource) │    │ (Resource) │    │ (Resource) │    │
│  └────────────┘    └────────────┘    └────────────┘    │
└──────────────────────────────────────────────────────────┘
```

### Ingress Controllers Populares

| Controller | Descrição | Complexidade | Features |
|------------|-----------|--------------|----------|
| **NGINX Ingress** | Mais popular, mantido pela comunidade | ⭐⭐ Média | Completo, estável |
| **Traefik** | Moderno, dynamic config | ⭐⭐ Média | Auto-discovery, Let's Encrypt |
| **HAProxy Ingress** | Alta performance | ⭐⭐⭐ Alta | Muito configurável |
| **Contour** | Baseado em Envoy | ⭐⭐⭐ Alta | Moderno, extensível |
| **Kong** | API Gateway + Ingress | ⭐⭐⭐⭐ Muito Alta | Plugins, rate limiting |
| **AWS ALB Ingress** | Para AWS EKS | ⭐⭐ Média | Integração AWS |
| **GCE Ingress** | Para GKE (Google) | ⭐ Baixa | Integração GCP |

### Instalar NGINX Ingress Controller

**NGINX Ingress** é o mais usado e será usado nos exemplos.

```bash
# Método 1: kubectl apply (bare metal / cloud genérico)
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Método 2: Helm (mais flexível)
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx

# Verificar instalação
kubectl get pods -n ingress-nginx

# Output:
# NAME                                        READY   STATUS    RESTARTS   AGE
# ingress-nginx-controller-5d88495688-abc12   1/1     Running   0          2m
```

### Verificar Ingress Controller

```bash
# Ver pods
kubectl get pods -n ingress-nginx

# Ver service (LoadBalancer que recebe tráfego externo)
kubectl get svc -n ingress-nginx

# Output:
# NAME                                 TYPE           EXTERNAL-IP     PORT(S)
# ingress-nginx-controller             LoadBalancer   34.123.45.67    80:30080/TCP,443:30443/TCP

# Ver logs
kubectl logs -n ingress-nginx ingress-nginx-controller-xxxxx

# Ver configuração nginx gerada
kubectl exec -n ingress-nginx ingress-nginx-controller-xxxxx -- cat /etc/nginx/nginx.conf
```

---

## 📄 Ingress Resources

### Ingress Resource Básico

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: minimal-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx  # Qual Ingress Controller usar
  rules:
  - http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

```bash
# Criar Ingress
kubectl apply -f minimal-ingress.yaml

# Ver Ingress
kubectl get ingress

# Output:
# NAME              CLASS   HOSTS   ADDRESS         PORTS   AGE
# minimal-ingress   nginx   *       34.123.45.67    80      1m

# Descrever Ingress
kubectl describe ingress minimal-ingress

# Testar
curl http://34.123.45.67
```

### Campos Importantes

**`ingressClassName`**: qual Ingress Controller usar
```yaml
spec:
  ingressClassName: nginx  # ou "traefik", "haproxy", etc
```

**`rules`**: regras de roteamento
```yaml
spec:
  rules:
  - host: example.com  # Opcional (se omitido, aceita qualquer host)
    http:
      paths:
      - path: /app
        pathType: Prefix  # Prefix, Exact, ImplementationSpecific
        backend:
          service:
            name: my-service
            port:
              number: 80
```

**`pathType`**: como interpretar o path
- **Prefix**: `/app` match `/app`, `/app/`, `/app/foo`
- **Exact**: `/app` match APENAS `/app` (não `/app/`)
- **ImplementationSpecific**: depende do Ingress Controller

**`defaultBackend`**: fallback se nenhuma rule match (404 customizado)
```yaml
spec:
  defaultBackend:
    service:
      name: default-backend
      port:
        number: 80
```

---

## 🛤️ Path-Based Routing

Rotear tráfego baseado no **path da URL**.

### Exemplo: Múltiplas Apps por Path

```
http://example.com/app1  →  app1-service
http://example.com/app2  →  app2-service
http://example.com/api   →  api-service
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: path-based-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      # /app1/* → app1-service
      - path: /app1(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: app1-service
            port:
              number: 80

      # /app2/* → app2-service
      - path: /app2(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: app2-service
            port:
              number: 80

      # /api/* → api-service
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 80
```

**Explicação do rewrite:**
- Request: `http://example.com/app1/users`
- Nginx rewrite para: `http://app1-service/users` (remove `/app1`)
- Regex `(/|$)(.*)` captura tudo depois de `/app1`
- `/$2` substitui pelo conteúdo capturado

### Exemplo Simples (sem rewrite)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: simple-path-ingress
spec:
  ingressClassName: nginx
  rules:
  - http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80

      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

```bash
# Criar Services
kubectl create deployment web --image=nginx --port=80
kubectl expose deployment web --port=80 --name=web-service

kubectl create deployment api --image=hashicorp/http-echo --port=8080
kubectl expose deployment api --port=8080 --name=api-service

# Criar Ingress
kubectl apply -f simple-path-ingress.yaml

# Testar
curl http://<INGRESS-IP>/web
curl http://<INGRESS-IP>/api
```

---

## 🏠 Host-Based Routing

Rotear tráfego baseado no **hostname** (virtual hosting).

### Exemplo: Múltiplos Hosts

```
http://app1.example.com  →  app1-service
http://app2.example.com  →  app2-service
http://api.example.com   →  api-service
```

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: host-based-ingress
spec:
  ingressClassName: nginx
  rules:
  # app1.example.com
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80

  # app2.example.com
  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80

  # api.example.com
  - host: api.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

```bash
# Aplicar
kubectl apply -f host-based-ingress.yaml

# Testar (assumindo DNS está configurado ou usando /etc/hosts)
curl http://app1.example.com
curl http://app2.example.com
curl http://api.example.com
```

### Configurar DNS Local (/etc/hosts)

Para testar localmente sem DNS:

```bash
# Pegar IP do Ingress
kubectl get ingress host-based-ingress

# Output:
# NAME                  ADDRESS         PORTS   AGE
# host-based-ingress    34.123.45.67    80      1m

# Adicionar ao /etc/hosts
echo "34.123.45.67 app1.example.com app2.example.com api.example.com" | sudo tee -a /etc/hosts

# Testar
curl http://app1.example.com
curl http://app2.example.com
```

### Combinar Host + Path Routing

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: combined-ingress
spec:
  ingressClassName: nginx
  rules:
  # example.com/web → web-service
  # example.com/api → api-service
  - host: example.com
    http:
      paths:
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

  # admin.example.com/* → admin-service
  - host: admin.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: admin-service
            port:
              number: 3000
```

---

## 🔒 TLS/HTTPS

Ingress pode **terminar TLS/SSL**, permitindo HTTPS.

### Criar TLS Secret

```bash
# Gerar certificado self-signed (apenas para teste!)
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key \
  -out tls.crt \
  -subj "/CN=example.com/O=example"

# Criar Secret do tipo TLS
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key

# Verificar Secret
kubectl get secret example-tls
kubectl describe secret example-tls
```

### Ingress com TLS

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-ingress
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    - www.example.com
    secretName: example-tls  # Secret TLS criado acima
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
  - host: www.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```bash
# Aplicar
kubectl apply -f tls-ingress.yaml

# Testar HTTPS
curl -k https://example.com
# -k ignora certificado self-signed (não usar em produção!)
```

### Redirect HTTP → HTTPS

Annotation para forçar HTTPS:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: tls-redirect-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"  # Força redirect HTTP → HTTPS
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

```bash
# Testar HTTP (deve redirecionar para HTTPS)
curl -L http://example.com
# -L segue redirects
```

### Múltiplos TLS Certificates

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: multi-tls-ingress
spec:
  ingressClassName: nginx
  tls:
  # Certificado para app1.example.com
  - hosts:
    - app1.example.com
    secretName: app1-tls

  # Certificado para app2.example.com
  - hosts:
    - app2.example.com
    secretName: app2-tls

  rules:
  - host: app1.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app1-service
            port:
              number: 80

  - host: app2.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app2-service
            port:
              number: 80
```

---

## 🏷️ Annotations

**Annotations** customizam comportamento do Ingress Controller.

### Annotations Comuns (NGINX Ingress)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: annotated-ingress
  annotations:
    # Rewrite
    nginx.ingress.kubernetes.io/rewrite-target: /$2

    # SSL Redirect
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

    # Backend Protocol (para backends HTTPS)
    nginx.ingress.kubernetes.io/backend-protocol: "HTTPS"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, PUT, DELETE"
    nginx.ingress.kubernetes.io/cors-allow-origin: "*"

    # Rate Limiting
    nginx.ingress.kubernetes.io/limit-rps: "10"

    # Timeouts
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"

    # Auth Basic
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # Whitelist IP
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/24,192.168.1.0/24"

    # Client Body Size (upload limit)
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

### Exemplo: Basic Authentication

```bash
# Criar arquivo de senhas
htpasswd -c auth myuser
# Digite a senha quando solicitado

# Criar Secret
kubectl create secret generic basic-auth --from-file=auth

# Ingress com auth
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: auth-ingress
  annotations:
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"
spec:
  ingressClassName: nginx
  rules:
  - host: secure.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
EOF

# Testar (deve pedir usuário/senha)
curl http://secure.example.com
# Output: 401 Unauthorized

curl -u myuser:password http://secure.example.com
# ✅ Funciona
```

### Exemplo: URL Rewrite

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: rewrite-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /$2
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      # Request: /api/v1/users
      # Reescrito para: /users
      - path: /api(/|$)(.*)
        pathType: ImplementationSpecific
        backend:
          service:
            name: api-service
            port:
              number: 8080
```

---

## 🔧 Troubleshooting

### 1. Ingress não está funcionando

**Verificar:**

```bash
# 1. Ingress Controller está rodando?
kubectl get pods -n ingress-nginx

# Output deve ser Running:
# ingress-nginx-controller-xxxxx   1/1     Running

# 2. Service do Ingress Controller tem EXTERNAL-IP?
kubectl get svc -n ingress-nginx

# Output:
# NAME                       TYPE           EXTERNAL-IP     PORT(S)
# ingress-nginx-controller   LoadBalancer   34.123.45.67    80:30080/TCP,443:30443/TCP

# Se EXTERNAL-IP = <pending>, aguardar (ou usar NodePort em bare metal)

# 3. Ingress resource existe?
kubectl get ingress

# 4. Ingress tem ADDRESS?
kubectl get ingress my-ingress

# Output:
# NAME         CLASS   HOSTS           ADDRESS         PORTS   AGE
# my-ingress   nginx   example.com     34.123.45.67    80      5m

# Se ADDRESS vazio, Ingress Controller não encontrou o Ingress
```

### 2. Ingress retorna 404

**Causas comuns:**

**a) Path não match**

```bash
# Ver rules do Ingress
kubectl describe ingress my-ingress

# Verificar se path está correto
# Exemplo: Ingress tem path=/app mas você acessa /
```

**b) Host não match**

```bash
# Ingress tem host: example.com
# Mas você acessa via IP ou outro hostname

# Testar com header Host:
curl -H "Host: example.com" http://<INGRESS-IP>
```

**c) Service backend não existe**

```bash
# Ver Service referenciado no Ingress
kubectl get svc web-service

# Se não existir, criar:
kubectl expose deployment web --port=80 --name=web-service
```

### 3. Ingress retorna 503 (Service Unavailable)

**Causas:**

**a) Backend Pods não estão rodando**

```bash
# Ver Pods do Service
kubectl get pods -l app=web

# Se não há Pods, o Service não tem endpoints
kubectl get endpoints web-service

# Output:
# NAME          ENDPOINTS
# web-service   <none>  # ❌ Sem Pods

# Criar Deployment
kubectl create deployment web --image=nginx --replicas=2
```

**b) Porta errada no Service**

```bash
# Descrever Service
kubectl describe svc web-service

# Verificar se Port match com targetPort dos Pods
# Service port: 80
# Container port: 8080
# ❌ Mismatch!

# Corrigir Service:
kubectl delete svc web-service
kubectl expose deployment web --port=80 --target-port=8080
```

### 4. TLS/HTTPS não funciona

**Verificar:**

```bash
# 1. Secret TLS existe?
kubectl get secret example-tls

# 2. Secret tem tls.crt e tls.key?
kubectl describe secret example-tls

# Output deve ter:
# Data
# ====
# tls.crt:  1234 bytes
# tls.key:  1234 bytes

# 3. Ingress referencia o Secret correto?
kubectl get ingress my-ingress -o yaml | grep secretName

# 4. Hostname no certificado match com Host do Ingress?
openssl x509 -in tls.crt -text -noout | grep CN

# Output:
# Subject: CN = example.com
```

### 5. Ver Logs do Ingress Controller

```bash
# Ver logs
kubectl logs -n ingress-nginx ingress-nginx-controller-xxxxx

# Logs em tempo real
kubectl logs -n ingress-nginx ingress-nginx-controller-xxxxx -f

# Procurar erros
kubectl logs -n ingress-nginx ingress-nginx-controller-xxxxx | grep -i error
```

### 6. Ver Configuração Nginx Gerada

```bash
# Ver nginx.conf gerado pelo Ingress Controller
kubectl exec -n ingress-nginx ingress-nginx-controller-xxxxx -- cat /etc/nginx/nginx.conf

# Procurar por backend específico
kubectl exec -n ingress-nginx ingress-nginx-controller-xxxxx -- cat /etc/nginx/nginx.conf | grep -A 10 "server_name example.com"
```

### 7. Testar Conectividade para Backend

```bash
# Entrar no Pod do Ingress Controller
kubectl exec -it -n ingress-nginx ingress-nginx-controller-xxxxx -- sh

# Dentro do Pod, testar conectividade para Service
curl http://web-service.default.svc.cluster.local

# Se falhar, problema está no Service/Pods
# Se funcionar, problema está no Ingress configuration
```

---

## 📝 Resumo

### Ingress

**O que é:**
- Recurso Kubernetes para expor apps HTTP/HTTPS
- Roteamento Layer 7 (host/path-based)
- Terminação TLS/SSL
- Load balancing

**Componentes:**
1. **Ingress Controller**: Pod que implementa as rules (nginx, Traefik, etc)
2. **Ingress Resource**: YAML que define regras de roteamento
3. **Services**: Backends para onde tráfego é roteado

**Vantagens vs LoadBalancer:**
- ✅ Um único IP para múltiplas apps
- ✅ Roteamento avançado (path/host)
- ✅ SSL/TLS centralizado
- ✅ Custo reduzido (um LoadBalancer ao invés de vários)

### Instalação (NGINX Ingress Controller)

```bash
# Instalar
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.8.1/deploy/static/provider/cloud/deploy.yaml

# Verificar
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx
```

### Ingress Resource Básico

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  ingressClassName: nginx
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```

### Path-Based Routing

```yaml
rules:
- http:
    paths:
    - path: /app1
      pathType: Prefix
      backend:
        service:
          name: app1-service
          port:
            number: 80
    - path: /app2
      pathType: Prefix
      backend:
        service:
          name: app2-service
          port:
            number: 80
```

### Host-Based Routing

```yaml
rules:
- host: app1.example.com
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: app1-service
          port:
            number: 80
- host: app2.example.com
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: app2-service
          port:
            number: 80
```

### TLS/HTTPS

```bash
# Criar TLS Secret
kubectl create secret tls example-tls \
  --cert=tls.crt \
  --key=tls.key
```

```yaml
spec:
  tls:
  - hosts:
    - example.com
    secretName: example-tls
  rules:
  - host: example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

---

## 🔧 Comandos Essenciais

```bash
# Ver Ingress resources
kubectl get ingress
kubectl get ingress -A  # Todos namespaces

# Descrever Ingress
kubectl describe ingress my-ingress

# Ver YAML do Ingress
kubectl get ingress my-ingress -o yaml

# Ver Ingress Controller Pods
kubectl get pods -n ingress-nginx

# Ver Service do Ingress Controller
kubectl get svc -n ingress-nginx

# Ver Logs do Ingress Controller
kubectl logs -n ingress-nginx ingress-nginx-controller-xxxxx

# Ver configuração nginx gerada
kubectl exec -n ingress-nginx ingress-nginx-controller-xxxxx -- cat /etc/nginx/nginx.conf

# Criar TLS Secret
kubectl create secret tls my-tls --cert=tls.crt --key=tls.key

# Deletar Ingress
kubectl delete ingress my-ingress
```

---

## 💡 Dicas para o Exame CKA

1. **Ingress precisa de Controller**
   - Ingress resource sozinho não faz nada
   - Precisa ter Ingress Controller instalado (nginx, Traefik, etc)
   - Verificar: `kubectl get pods -n ingress-nginx`

2. **ingressClassName é obrigatório**
   - Especifica qual Ingress Controller usar
   - `spec.ingressClassName: nginx`

3. **pathType importa**
   - **Prefix**: match `/app` e `/app/*`
   - **Exact**: match APENAS `/app`
   - Use Prefix para maioria dos casos

4. **TLS Secret formato específico**
   - Tipo: `kubernetes.io/tls`
   - Campos: `tls.crt` e `tls.key`
   - Criar com: `kubectl create secret tls`

5. **Troubleshooting ordem:**
   ```bash
   # 1. Ingress Controller rodando?
   kubectl get pods -n ingress-nginx

   # 2. Ingress resource existe e tem ADDRESS?
   kubectl get ingress

   # 3. Service backend existe e tem Endpoints?
   kubectl get svc my-service
   kubectl get endpoints my-service

   # 4. Pods estão rodando?
   kubectl get pods -l app=my-app

   # 5. Ver logs do Ingress Controller
   kubectl logs -n ingress-nginx ingress-nginx-controller-xxxxx
   ```

6. **Annotations são específicas do Controller**
   - NGINX: `nginx.ingress.kubernetes.io/*`
   - Traefik: `traefik.ingress.kubernetes.io/*`
   - Ver docs do Controller usado

---

⬅️ **Anterior**: [services.md](./services.md) | ➡️ **Próximo**: [network-policies.md](./network-policies.md)
