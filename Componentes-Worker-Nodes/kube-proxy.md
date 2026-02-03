# Kube Proxy - Guia Completo

## 1. O que é Kube Proxy?

**Kube Proxy** é um componente que roda em **cada nó do cluster Kubernetes**. É responsável por **manter regras de rede** que permitem comunicação entre pods e serviços. Ele implementa o modelo de rede do Kubernetes.

### 1.1 Analogia

Imagine o Kube Proxy como um **porteiro de switchboard de telefone**:

- 📞 Cada chamada (requisição) chega ao porteiro
- 📋 Ele consulta a lista de ramais (endpoints)
- 🔄 Redireciona a chamada para o ramal correto
- 🔁 Distribui chamadas entre múltiplos ramais
- ☎️ Se um ramal cair, redireciona para outro

---

## 2. Responsabilidades Principais

### 2.1 Implementação de Serviços

- **Criar regras de rede** que mapeiam Service para Pods
- **Permitir comunicação** entre pods
- **Expor serviços** dentro do cluster

### 2.2 Load Balancing

- **Distribuir tráfego** entre múltiplos pods
- **Balancear conexões** entre endpoints
- **Round-robin** ou outras estratégias

### 2.3 Network Translation

- **Traducir endereços IP** (SNAT/DNAT)
- **Reescrever portas** quando necessário
- **Manter estado de conexão** (stateful)

### 2.4 Service Discovery

- **Descobrir endpoints** (pods que implementam o service)
- **Monitorar mudanças** (pods adicionados/removidos)
- **Atualizar regras** dinamicamente

### 2.5 Roteamento de Tráfego

- **Rotear tráfego** para o pod correto
- **Garantir conectividade** dentro do cluster
- **Isolar tráfego** entre namespaces (via policy)

---

## 3. Problema que Kube Proxy Resolve

### 3.1 Sem Kube Proxy (Não Funciona)

```
┌─────────────────────────────────────────┐
│  Pod A (10.0.0.5)                       │
│                                         │
│  Quer conectar ao serviço "web"         │
│  que tem 3 pods:                        │
│  - Pod B (10.0.0.10)                    │
│  - Pod C (10.0.0.11)                    │
│  - Pod D (10.0.0.12)                    │
└────────────┬────────────────────────────┘
             │
    Pod A tenta conectar em 10.0.0.10
             │
    ✗ Problema: Qual IP usar?
    ✗ Se usar 10.0.0.10, sempre vai para Pod B
    ✗ Se Pod B cai, ninguém fica sabendo
    ✗ Sem abstração de serviço

```

### 3.2 Com Kube Proxy (Funciona)

```
┌─────────────────────────────────────────┐
│  Pod A (10.0.0.5)                       │
│                                         │
│  Quer conectar ao serviço "web"         │
│  (Service IP: 10.96.0.5)                │
└────────────┬────────────────────────────┘
             │
    Pod A conecta em 10.96.0.5 (Service IP)
             │
    ┌────────▼──────────────────────────┐
    │  Kube Proxy (no mesmo nó)         │
    │                                  │
    │  "10.96.0.5 é um serviço!"        │
    │  Encontra endpoints:              │
    │  - 10.0.0.10 (Pod B)              │
    │  - 10.0.0.11 (Pod C)              │
    │  - 10.0.0.12 (Pod D)              │
    │                                  │
    │  Round-robin:                     │
    │  1ª req → 10.0.0.10               │
    │  2ª req → 10.0.0.11               │
    │  3ª req → 10.0.0.12               │
    │  4ª req → 10.0.0.10 (repete)      │
    └────────┬──────────────────────────┘
             │
    ✓ Abstração de serviço
    ✓ Load balancing automático
    ✓ Tolerância a falhas
    ✓ Escalabilidade

```

---

## 4. Arquitetura do Kube Proxy

### 4.1 Componentes Internos

```
┌──────────────────────────────────────────────────────────┐
│              Kube Proxy (em cada nó)                     │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  1. API Server Watcher                             │ │
│  │     - Monitora Services                            │ │
│  │     - Monitora Endpoints (pods que implementam)    │ │
│  │     - Detecta mudanças                             │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  2. Service Manager                               │ │
│  │     - Rastreia serviços do cluster                │ │
│  │     - Mapeia Service → Endpoints                  │ │
│  │     - Configura load balancing                    │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  3. Proxier (Implementação de Proxy)              │ │
│  │                                                   │ │
│  │     Pode ser:                                    │ │
│  │     ├─ iptables (mais comum)                     │ │
│  │     ├─ ipvs (performance)                        │ │
│  │     ├─ kernelspace (eficiente)                   │ │
│  │     └─ userspace (legado, lento)                 │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  4. Conntrack Manager                             │ │
│  │     - Mantém estado de conexões                   │ │
│  │     - Rastreia fluxo de pacotes                   │ │
│  │     - Garbage collection de conexões velhas       │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  5. Network Rules Engine                          │ │
│  │     - iptables/ipvs rules                         │ │
│  │     - Firewall rules                              │ │
│  │     - Masquerading (SNAT)                         │ │
│  │     - Destination NAT (DNAT)                      │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
└──────────────────────────────────────────────────────────┘

```

---

## 5. Modos de Funcionamento do Kube Proxy

### 5.1 Userspace Mode (Legado, Lento)

```
Pod A ──┐
        │ Conexão
        ▼
    Kernel (iptables rule)
        │
        ▼
    Kube Proxy (userspace)  ← Processo em userspace
        │ Cópia de dados entre kernel e userspace
        │ (Lento!)
        ▼
    Kernel
        │
        ▼
Pod B (Endpoint)

❌ Lento: Cópia de dados kernel ↔ userspace
❌ Overhead: Mudança de contexto
✓ Compatibilidade: Funciona em qualquer SO

```

### 5.2 Iptables Mode (Padrão, Rápido)

```
Pod A ──┐
        │ Conexão
        ▼
    Kernel iptables rules  ← Tudo no kernel (Fast!)
    ├─ Match: destino 10.96.0.5?
    ├─ Sim: Load balancing
    ├─ Escolhe endpoint: 10.0.0.10
    ├─ DNAT (muda destino)
    │
        ▼
Pod B (10.0.0.10)

✓ Rápido: Tudo no kernel
✓ Sem userspace: Sem overhead
✓ Padrão em Kubernetes
❌ Escalabilidade: Muitas regras = lento

```

### 5.3 IPVS Mode (Melhor Performance)

```
Pod A ──┐
        │ Conexão
        ▼
    Kernel IPVS (Linux Virtual Server)
    ├─ Usa hash table (O(1))
    ├─ Load balancing eficiente
    ├─ Suporta algoritmos avançados
    ├─ Escalável para muitos serviços
    │
        ▼
Pod B (Endpoint)

✓ Muito Rápido: Hash table em kernel
✓ Escalável: Suporta milhares de serviços
✓ Algoritmos: Round-robin, LRU, etc
✓ Stateless: Não precisa de conntrack
❌ Requer IPVS kernel module

```

### 5.4 Comparação dos Modos

| Modo | Velocidade | Escalabilidade | Complexidade | Padrão |
| --- | --- | --- | --- | --- |
| **Userspace** | Lenta | Baixa | Simples | ❌ |
| **Iptables** | Média | Média | Média | ✅ |
| **IPVS** | Rápida | Alta | Alta | ⭐ Recomendado |

---

## 6. Como Kube Proxy Funciona: Fluxo de Tráfego

### 6.1 Cenário: Pod Acessando Service

```
Cluster:
├─ Pod A (10.0.0.5) - em Nó-1
├─ Pod B (10.0.0.10) - em Nó-2
├─ Pod C (10.0.0.11) - em Nó-2
└─ Pod D (10.0.0.12) - em Nó-3

Service "web":
├─ Service IP (ClusterIP): 10.96.0.5
├─ Port: 80
├─ Endpoints: [10.0.0.10, 10.0.0.11, 10.0.0.12]

```

### 6.2 Passo a Passo: Pod A Conectando ao Service

```
1️⃣  Pod A inicia requisição
    $ curl web  (resolve para 10.96.0.5:80)

2️⃣  Pacote sai do Pod A
    Source: 10.0.0.5:xxxxx
    Dest: 10.96.0.5:80

3️⃣  Chega no Kube Proxy (Nó-1)

    Kube Proxy vê: "10.96.0.5:80 é um serviço!"

    Consulta endpoints do serviço "web"
    Endpoints: [10.0.0.10, 10.0.0.11, 10.0.0.12]

    Load balancing (Round-robin):
    Escolhe: 10.0.0.10 (Pod B)

4️⃣  DNAT (Destination NAT)

    Pacote original:
    ┌─────────────────────────────┐
    │ Source: 10.0.0.5:xxxxx      │
    │ Dest: 10.96.0.5:80          │
    └─────────────────────────────┘
              ↓ (DNAT)
    Pacote transformado:
    ┌─────────────────────────────┐
    │ Source: 10.0.0.5:xxxxx      │
    │ Dest: 10.0.0.10:8080        │ ← Endpoint real
    └─────────────────────────────┘

5️⃣  Pacote chega em Pod B

    Pod B responde:
    Source: 10.0.0.10:8080
    Dest: 10.0.0.5:xxxxx

6️⃣  SNAT (Source NAT) - Retorno

    Resposta original:
    ┌─────────────────────────────┐
    │ Source: 10.0.0.10:8080      │
    │ Dest: 10.0.0.5:xxxxx        │
    └─────────────────────────────┘
              ↓ (SNAT)
    Resposta transformada:
    ┌─────────────────────────────┐
    │ Source: 10.96.0.5:80        │ ← Serviço
    │ Dest: 10.0.0.5:xxxxx        │
    └─────────────────────────────┘

7️⃣  Pod A recebe resposta

    Parece que veio de 10.96.0.5 (o serviço)
    Pod A não sabe que realmente veio de 10.0.0.10

```

---

## 7. Services e Endpoints

### 7.1 Service: Abstração de Rede

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP           # ← Tipo de serviço
  selector:
    app: web               # ← Seleciona pods com esta label
  ports:
  - port: 80               # ← Porta do serviço
    targetPort: 8080       # ← Porta no container
    protocol: TCP

```

**O que Kube Proxy faz:**

1. Obtém label selector `app=web`
2. Encontra todos os pods com esta label
3. Cria endpoint para cada pod encontrado
4. Configura regras de rede (DNAT/SNAT)

### 7.2 Endpoints: IPs dos Pods

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: web              # ← Nome do serviço
subsets:
- addresses:
  - ip: 10.0.0.10        # ← Pod B
    targetRef:
      kind: Pod
      name: pod-b
      namespace: default
  - ip: 10.0.0.11        # ← Pod C
    targetRef:
      kind: Pod
      name: pod-c
  - ip: 10.0.0.12        # ← Pod D
    targetRef:
      kind: Pod
      name: pod-d
  ports:
  - port: 8080           # ← Porta no container
    protocol: TCP

```

**Atualização Dinâmica:**

- Pod criado → Endpoint adicionado automaticamente
- Pod deletado → Endpoint removido automaticamente
- Pod falha → Removed from endpoints (readiness probe)

---

## 8. Tipos de Services

### 8.1 ClusterIP (Padrão)

```
┌────────────────────────────────────────┐
│  ClusterIP Service                     │
├────────────────────────────────────────┤
│ Service IP (Virtual): 10.96.0.5        │
│ Apenas acessível dentro do cluster     │
│                                        │
│ Pod dentro do cluster:                 │
│ $ curl web  (10.96.0.5:80)     ✓       │
│                                        │
│ Fora do cluster:                       │
│ $ curl 10.96.0.5  (impossível)  ✗      │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080

```

### 8.2 NodePort

```
┌────────────────────────────────────────┐
│  NodePort Service                      │
├────────────────────────────────────────┤
│ Service IP: 10.96.0.5                  │
│ Port: 80 (no service)                  │
│ NodePort: 30080 (em cada nó)           │
│                                        │
│ De dentro do cluster:                  │
│ $ curl web:80  ✓                       │
│ $ curl 10.96.0.5:80  ✓                 │
│                                        │
│ De fora do cluster:                    │
│ $ curl <node-ip>:30080  ✓              │
│ $ curl nó-1-ip:30080   (vai até pod)   │
│ $ curl nó-2-ip:30080   (vai até pod)   │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080    # ← Porta em cada nó (30000-32767)

```

### 8.3 LoadBalancer

```
┌────────────────────────────────────────┐
│  LoadBalancer Service                  │
├────────────────────────────────────────┤
│ Service IP: 10.96.0.5 (interno)        │
│ External IP: 203.0.113.5 (fornecedor)  │
│                                        │
│ De dentro:                             │
│ $ curl web:80  ✓                       │
│                                        │
│ De fora:                               │
│ $ curl 203.0.113.5:80  ✓               │
│                                        │
│ Fluxo:                                 │
│ Internet → Load Balancer (nuvem)       │
│           → NodePort em cada nó        │
│           → Service → Pods             │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080

```

### 8.4 ExternalName

```
┌────────────────────────────────────────┐
│  ExternalName Service                  │
├────────────────────────────────────────┤
│ Mapeia nome de serviço para DNS externo│
│                                        │
│ $ curl database (dentro do cluster)    │
│  → CNAME database.external.example.com │
│  → Conecta a banco externo             │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ExternalName
  externalName: database.external.example.com

```

---

## 9. Service Discovery

### 9.1 DNS (Recomendado)

Kubernetes inclui um CoreDNS que resolve nomes automaticamente:

```bash
# Pod dentro do cluster pode acessar:

# Serviço no mesmo namespace
curl web           # → 10.96.0.5

# Serviço em outro namespace
curl web.payments  # → 10.96.0.100

# Nome completo
curl web.payments.svc.cluster.local
# → 10.96.0.100

```

**Como funciona:**

```
1. Pod quer conectar em "web"
2. Kubelet injeta CoreDNS como resolver
3. Consulta CoreDNS (10.0.0.10:53)
4. CoreDNS consulta ETCD
5. Retorna IP: 10.96.0.5
6. Pod conecta em 10.96.0.5
7. Kube Proxy faz load balancing

```

### 9.2 Environment Variables (Legado)

Kubelet injeta variáveis de ambiente:

```bash
# Dentro do container
$ env | grep SERVICE

WEB_SERVICE_HOST=10.96.0.5
WEB_SERVICE_PORT=80

# Útil para apps legadas

```

---

## 10. Iptables Rules (Exemplo Detalhado)

### 10.1 Como Kube Proxy Usa Iptables

```
Quando você cria:
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080

Kube Proxy cria regras iptables:

# Cadeia de entrada
*nat
:PREROUTING ACCEPT
:KUBE-SERVICES - [0:0]

# Se pacote vai para Service IP
-A PREROUTING -m comment --comment "service"
  -j KUBE-SERVICES

# Service IP 10.96.0.5:80 → Endpoints
-A KUBE-SERVICES -d 10.96.0.5/32
  -m comment --comment "web"
  -m tcp -p tcp --dport 80
  -j KUBE-SVC-XXXXXXXXXXXX

# Endpoints (Round-robin com probability)
-A KUBE-SVC-XXXXXXXXXXXX -m statistic
  --mode random --probability 0.3333
  -j KUBE-SEP-YYYYYY  # Pod B (10.0.0.10:8080)
-A KUBE-SVC-XXXXXXXXXXXX -m statistic
  --mode random --probability 0.5000
  -j KUBE-SEP-ZZZZZZ  # Pod C (10.0.0.11:8080)
-A KUBE-SVC-XXXXXXXXXXXX
  -j KUBE-SEP-WWWWWW  # Pod D (10.0.0.12:8080)

# DNAT (muda destino)
-A KUBE-SEP-YYYYYY -m comment --comment "web"
  -m tcp -p tcp
  -j DNAT --to-destination 10.0.0.10:8080

```

### 10.2 Ver Regras Iptables

```bash
# Ver todas as regras NAT
sudo iptables -t nat -L -n -v

# Ver regras de um serviço específico
sudo iptables -t nat -L KUBE-SERVICES -n -v

# Ver estatísticas
sudo iptables -t nat -L -n -v -x

```

---

## 11. Configuração do Kube Proxy

### 11.1 Escolher Modo

```bash
kube-proxy \
  --kubeconfig=/etc/kubernetes/kube-proxy.conf \
  --proxy-mode=iptables \  # ou ipvs, userspace
  --cluster-cidr=10.0.0.0/8 \
  --masquerade-all=false \
  --v=2

```

### 11.2 Configuração via YAML

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.conf
clusterCIDR: 10.0.0.0/8
configSyncPeriod: 15m0s
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
detectLocalMode: ClusterCIDR
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
mode: iptables  # ou ipvs
nodePortAddresses: null
metricsBindAddress: 127.0.0.1:10249
udpIdleTimeout: 250ms

```

---

## 12. Monitoramento e Troubleshooting

### 12.1 Ver Status do Kube Proxy

```bash
# Ver pods do kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Ver logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Ver métricas
curl localhost:10249/metrics

```

### 12.2 Diagnosticar Problemas de Conectividade

### Problema 1: Não consegue Conectar ao Service

```bash
# Verificar se service existe
kubectl get svc web

# Ver endpoints do service
kubectl get endpoints web

# Verificar IP do service
kubectl get svc web -o jsonpath='{.spec.clusterIP}'

# Testar conectividade
kubectl run -it --rm test --image=curl --restart=Never \
  -- curl web:80

# Ver logs do service
kubectl describe svc web

```

### Problema 2: Kube Proxy Não Funciona

```bash
# Verificar se kube-proxy está rodando
kubectl get pods -n kube-system kube-proxy-<node>

# Ver logs
kubectl logs -n kube-system kube-proxy-<node>

# Reiniciar
kubectl delete pod -n kube-system kube-proxy-<node>

# Ver regras iptables
sudo iptables -t nat -L KUBE-SERVICES -n

```

### Problema 3: Service IP Não Resolve

```bash
# Verificar CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Testar DNS
kubectl run -it --rm test --image=alpine --restart=Never \
  -- nslookup web

# Ver logs do DNS
kubectl logs -n kube-system -l k8s-app=kube-dns

```

### 12.3 Comandos Úteis de Debug

```bash
# Entrar em pod e debugar
kubectl exec -it <pod> -- /bin/bash

# Ver conectividade de um pod
kubectl run -it --rm debug --image=alpine --restart=Never \
  -- sh

# Dentro do debug container:
# Testar DNS
nslookup web
nslookup web.default.svc.cluster.local

# Testar conectividade
curl -v web:80

# Ver iptables (se em host)
sudo iptables -t nat -L -n | grep SERVICE

```

---

## 13. Network Policies (Segurança)

### 13.1 Limitando Tráfego com NetworkPolicy

Kube Proxy respeita Network Policies para filtrar tráfego:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-frontend
spec:
  podSelector:
    matchLabels:
      app: web    # ← Aplica a pods com esta label

  policyTypes:
  - Ingress      # ← Tráfego de entrada

  ingress:
  - from:
        # Apenas de pods com label tier=frontend
        - podSelector:
            matchLabels:
              tier: frontend
    ports:
    - protocol: TCP
      port: 80

```

**Efeito:**

```
Tráfego permitido:
Frontend Pod → Web Pod ✓

Tráfego bloqueado:
Database Pod → Web Pod ✗
External → Web Pod ✗

```

---

## 14. Performance e Otimizações

### 14.1 IPVS vs Iptables

```
Iptables Mode:
┌──────────────────┐
│ 10 serviços      │ Rápido
│ 100 pods         │ Muitas regras (1000+)
└──────────────────┘

IPVS Mode:
┌──────────────────┐
│ 1000 serviços    │ Ainda rápido
│ 10000 pods       │ Hash table (O(1))
└──────────────────┘

```

### 14.2 Otimizações

```bash
# 1. Use IPVS para clusters grandes
kube-proxy --proxy-mode=ipvs

# 2. Ajuste conntrack
--conntrack-max=262144

# 3. Use session affinity se necessário
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP  # ← Mesmo cliente → mesmo pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800

```

---

## 15. Fluxo Completo do Cluster

### 15.1 Requisição de Pod A para Service

```
┌─────────────────────────────────────────────────────────┐
│  1. Pod A quer acessar service "web"                    │
│     curl web                                            │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  2. CoreDNS Resolve Nome                               │
│     web → 10.96.0.5 (Service IP)                       │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  3. Pod A Conecta em 10.96.0.5:80                      │
│     Pacote: Src=10.0.0.5, Dst=10.96.0.5:80            │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  4. Kube Proxy (iptables mode)                         │
│     ├─ Detecta: 10.96.0.5:80 é um serviço             │
│     ├─ Encontra endpoints: [10.0.0.10, 10.0.0.11, ...]│
│     ├─ Load balancing (round-robin)                    │
│     ├─ Escolhe: 10.0.0.10 (Pod B)                      │
│     └─ DNAT: Reescreve Dst para 10.0.0.10:8080        │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  5. Pacote Chega em Pod B                              │
│     Vê: Src=10.0.0.5, Dst=10.0.0.10:8080              │
│     Processa requisição                                 │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  6. Pod B Responde                                     │
│     Pacote: Src=10.0.0.10:8080, Dst=10.0.0.5          │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  7. Kube Proxy SNAT (Retorno)                          │
│     Reescreve Src: 10.96.0.5 (Service IP)             │
│     Pacote: Src=10.96.0.5:80, Dst=10.0.0.5            │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  8. Pod A Recebe Resposta                              │
│     Vê como se veio de 10.96.0.5 (o serviço)          │
│     Não sabe que realmente veio de 10.0.0.10           │
└─────────────────────────────────────────────────────────┘

```

---

## 16. Comparação: Service Discovery Methods

| Método | Velocidade | Flexibilidade | Uso |
| --- | --- | --- | --- |
| **DNS** | Rápida | Alta | Padrão, recomendado |
| **Env Vars** | Muito Rápida | Baixa | Apps legadas |
| **API Direct** | Média | Muito Alta | Casos especiais |

---

## 17. Resumo das Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Load Balancing** | Distribui tráfego entre endpoints |
| **Service IP** | Cria IP virtual para cada serviço |
| **DNS** | Resolve nomes de serviços |
| **NAT** | DNAT/SNAT para tradução de endereços |
| **Networking** | Conecta pods dentro do cluster |
| **Monitoramento** | Atualiza regras quando pods mudam |
| **Policy** | Respeita NetworkPolicies |

---

## 18. Conclusão

- **Kube Proxy** é o "porteiro de switchboard" do cluster
- **Implementa o modelo de rede** do Kubernetes
- **Cria abstrações de serviço** (Service IPs)
- **Faz load balancing** entre pods automaticamente
- **Usa NAT** (DNAT/SNAT) para tradução de endereços
- **3 modos**: Userspace (lento), Iptables (padrão), IPVS (rápido)
- **Trabalha com Kubelet** para descobrir endpoints
- É **essencial** para comunicação entre pods

---

