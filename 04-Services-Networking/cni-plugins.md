# CNI - Container Network Interface

Este documento cobre CNI (Container Network Interface), os principais plugins CNI para Kubernetes, com foco especial em **Weave Net**.

## 📚 Conteúdo

1. [O que é CNI?](#o-que-é-cni)
2. [Como CNI Funciona](#como-cni-funciona)
3. [CNI Plugins Populares](#cni-plugins-populares)
4. [Weave Net](#weave-net)
5. [Instalação e Configuração](#instalação-e-configuração)
6. [Troubleshooting](#troubleshooting)

---

## 🔌 O que é CNI?

**CNI (Container Network Interface)** é uma **especificação** e um conjunto de **bibliotecas** para configurar interfaces de rede em containers Linux.

### Por que CNI existe?

Antes do CNI, cada orquestrador de containers (Kubernetes, Mesos, etc.) tinha sua própria forma de configurar redes. CNI padronizou isso.

**CNI define:**
- Como plugins de rede devem se comportar
- Como configurar redes de Pods
- Como alocar endereços IP
- Como configurar rotas

### Características do CNI

```
┌────────────────────────────────────────────────────────┐
│                    Kubernetes                          │
│                                                        │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐          │
│  │  Pod A   │   │  Pod B   │   │  Pod C   │          │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘          │
│       │              │              │                 │
│       └──────────────┼──────────────┘                 │
│                      │                                │
│              ┌───────▼────────┐                       │
│              │  CNI Plugin    │                       │
│              │  (Weave/       │                       │
│              │   Calico/      │                       │
│              │   Flannel)     │                       │
│              └────────────────┘                       │
└────────────────────────────────────────────────────────┘
```

**Responsabilidades do CNI:**
1. ✅ Criar interface de rede no container
2. ✅ Alocar endereço IP
3. ✅ Configurar rotas
4. ✅ Configurar DNS (opcional)

**NÃO é responsabilidade do CNI:**
- ❌ Port mapping (responsabilidade do runtime)
- ❌ Network policies (implementado por alguns plugins como feature adicional)

---

## ⚙️ Como CNI Funciona

### Workflow CNI

Quando um Pod é criado:

```
1. Kubelet detecta novo Pod
        ↓
2. Kubelet cria network namespace para o Pod
        ↓
3. Kubelet chama CNI plugin com comando ADD
        ↓
4. CNI plugin:
   - Cria veth pair
   - Move uma ponta para namespace do Pod
   - Aloca IP do range configurado
   - Configura rotas
   - Retorna informações (IP, gateway, etc)
        ↓
5. Pod está pronto para comunicar na rede
```

### Configuração CNI

CNI plugins são configurados via arquivos JSON em `/etc/cni/net.d/`.

**Exemplo de configuração:**

```bash
# Ver configurações CNI
ls -la /etc/cni/net.d/

# Conteúdo típico:
# 10-weave.conflist
# 10-flannel.conflist
# 10-calico.conflist
```

**Exemplo de arquivo de configuração (Weave):**

```json
{
  "cniVersion": "0.3.0",
  "name": "weave",
  "plugins": [
    {
      "type": "weave-net",
      "hairpinMode": true
    },
    {
      "type": "portmap",
      "capabilities": {
        "portMappings": true
      }
    }
  ]
}
```

### Binários CNI

CNI plugins são **binários executáveis** em `/opt/cni/bin/`.

```bash
# Ver binários CNI instalados
ls -la /opt/cni/bin/

# Binários típicos:
# bridge      - Bridge network plugin
# host-local  - IP allocation plugin
# loopback    - Loopback interface
# portmap     - Port mapping
# weave-net   - Weave Net plugin (se instalado)
# calico      - Calico plugin (se instalado)
# flannel     - Flannel plugin (se instalado)
```

### CNI Plugin Execution

Kubelet executa o plugin passando:

**Variáveis de ambiente:**
```bash
CNI_COMMAND=ADD           # Operação: ADD, DEL, CHECK, VERSION
CNI_CONTAINERID=abc123    # ID do container
CNI_NETNS=/var/run/netns/abc123  # Network namespace
CNI_IFNAME=eth0          # Nome da interface
CNI_PATH=/opt/cni/bin    # Caminho dos binários CNI
```

**Stdin (configuração JSON):**
```json
{
  "cniVersion": "0.3.0",
  "name": "weave",
  "type": "weave-net"
}
```

**Output (resultado):**
```json
{
  "cniVersion": "0.3.0",
  "interfaces": [
    {
      "name": "eth0",
      "mac": "02:42:ac:11:00:02"
    }
  ],
  "ips": [
    {
      "version": "4",
      "address": "10.32.0.5/12",
      "gateway": "10.32.0.1"
    }
  ]
}
```

---

## 🔧 CNI Plugins Populares

### Comparação de Plugins

| Plugin | Tipo | Encryption | Network Policy | Performance | Complexidade |
|--------|------|------------|----------------|-------------|--------------|
| **Flannel** | Overlay | ❌ Não | ❌ Não | ⭐⭐⭐ Alta | ⭐ Baixa |
| **Calico** | BGP/Overlay | ✅ Sim (WireGuard) | ✅ Sim | ⭐⭐⭐⭐ Muito Alta | ⭐⭐⭐ Alta |
| **Weave Net** | Overlay | ✅ Sim | ✅ Sim | ⭐⭐ Média | ⭐⭐ Média |
| **Cilium** | eBPF | ✅ Sim | ✅ Sim | ⭐⭐⭐⭐⭐ Excelente | ⭐⭐⭐⭐ Muito Alta |
| **Kindnet** | Bridge | ❌ Não | ❌ Não | ⭐⭐⭐ Alta | ⭐ Muito Baixa |

### 1. Flannel

**Tipo:** Overlay network (VXLAN)

**Características:**
- Simples e fácil de configurar
- Foca apenas em conectividade básica
- Não suporta Network Policies
- Popular para clusters simples

**Instalação:**
```bash
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

### 2. Calico

**Tipo:** BGP routing (pode usar overlay)

**Características:**
- Alta performance (roteamento direto via BGP)
- Suporta Network Policies avançadas
- Encryption via WireGuard
- Muito usado em produção

**Instalação:**
```bash
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### 3. Cilium

**Tipo:** eBPF-based

**Características:**
- Performance excepcional (usa eBPF no kernel)
- Network Policies avançadas
- Observabilidade integrada
- Requer kernel Linux moderno (≥4.9)

**Instalação:**
```bash
helm repo add cilium https://helm.cilium.io/
helm install cilium cilium/cilium --namespace kube-system
```

### 4. Kindnet

**Tipo:** Bridge (CNI bridge plugin)

**Características:**
- Muito simples (apenas para Kind - Kubernetes in Docker)
- Não funciona em clusters multi-node reais
- Sem Network Policies

**Uso:**
```bash
# Kindnet é instalado automaticamente pelo Kind
kind create cluster
```

---

## 🕸️ Weave Net

**Weave Net** é um CNI plugin que cria uma **overlay network** entre Pods em diferentes nodes.

### Como Weave Net Funciona

```
┌─────────────────────────────────────────────────────────────┐
│                        Node 1                               │
│                                                             │
│  ┌──────────┐   ┌──────────┐   ┌──────────┐               │
│  │  Pod A   │   │  Pod B   │   │  Pod C   │               │
│  │10.32.0.5 │   │10.32.0.6 │   │10.32.0.7 │               │
│  └────┬─────┘   └────┬─────┘   └────┬─────┘               │
│       │              │              │                      │
│       └──────────────┼──────────────┘                      │
│                      │                                     │
│               ┌──────▼──────┐                              │
│               │  weave net  │ (cria bridge virtual)        │
│               │   bridge    │                              │
│               └──────┬──────┘                              │
│                      │                                     │
│               ┌──────▼──────┐                              │
│               │ weave router│ (DaemonSet)                  │
│               │   (Pod)     │                              │
│               └──────┬──────┘                              │
│                      │                                     │
└──────────────────────┼─────────────────────────────────────┘
                       │
                       │ Encapsula tráfego em VXLAN/UDP
                       │
┌──────────────────────┼─────────────────────────────────────┐
│                      │           Node 2                    │
│               ┌──────▼──────┐                              │
│               │ weave router│                              │
│               │   (Pod)     │                              │
│               └──────┬──────┘                              │
│                      │                                     │
│               ┌──────▼──────┐                              │
│               │  weave net  │                              │
│               │   bridge    │                              │
│               └──────┬──────┘                              │
│                      │                                     │
│       ┌──────────────┼──────────────┐                      │
│       │              │              │                      │
│  ┌────▼─────┐   ┌────▼─────┐   ┌───▼──────┐               │
│  │  Pod D   │   │  Pod E   │   │  Pod F   │               │
│  │10.32.0.8 │   │10.32.0.9 │   │10.32.0.10│               │
│  └──────────┘   └──────────┘   └──────────┘               │
└─────────────────────────────────────────────────────────────┘
```

### Características do Weave Net

**Vantagens:**
- ✅ Fácil de instalar (um único comando)
- ✅ Suporta **encryption** (IPsec/NaCl)
- ✅ Suporta **Network Policies**
- ✅ Descoberta automática de peers (nodes)
- ✅ Não requer configuração externa (etcd, Consul, etc)
- ✅ Suporta **multicast**

**Desvantagens:**
- ❌ Performance menor que BGP routing (Calico)
- ❌ Overhead de encapsulamento (VXLAN)

### Arquitetura do Weave Net

**Componentes:**

1. **Weave Net DaemonSet**: roda em cada node
2. **Weave Net CNI Plugin**: binário em `/opt/cni/bin/`
3. **Weave Net Bridge**: bridge virtual por node

**Fluxo de comunicação:**

```
Pod A (Node 1) → Pod D (Node 2)
        ↓
1. Pod A envia pacote para 10.32.0.8
        ↓
2. Pacote vai para weave bridge no Node 1
        ↓
3. Weave router vê que destino está no Node 2
        ↓
4. Encapsula pacote em VXLAN/UDP
        ↓
5. Envia pela rede física para Node 2
        ↓
6. Weave router no Node 2 desencapsula
        ↓
7. Encaminha para Pod D via weave bridge
        ↓
8. Pod D recebe o pacote
```

### Range de IPs (IPAM)

Weave Net usa **IP Address Management (IPAM)** para alocar IPs.

**Range padrão:** `10.32.0.0/12`

```
10.32.0.0/12 = 10.32.0.0 - 10.47.255.255
             = 2^20 IPs = 1.048.576 endereços
```

**Como funciona:**
- Cada node recebe um **subnet** do range
- Weave distribui automaticamente ranges entre nodes
- Não requer etcd ou banco de dados externo

**Ver ranges alocados:**

```bash
# Executar no node
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status

# Output mostra:
# 10.32.0.0/12 (via peer abc123)
# 10.40.0.0/12 (via peer def456)
```

### Encryption

Weave suporta **encryption automática** do tráfego entre nodes.

**Algoritmos suportados:**
- **NaCl** (default): encryption + authentication
- **IPsec**: encryption via kernel

**Habilitar encryption:**

```bash
# Durante instalação, usar secret para encryption
kubectl create secret -n kube-system generic weave-passwd --from-literal=weave-passwd=MY_SECRET_PASSWORD

# Weave detecta o secret e habilita encryption automaticamente
```

**Verificar se encryption está ativa:**

```bash
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status connections

# Output:
# <- 192.168.1.11:6783   established encrypted   sleeve
```

### Network Policies com Weave Net

Weave Net suporta **Kubernetes Network Policies**.

**Exemplo de Network Policy:**

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: default
spec:
  podSelector: {}
  policyTypes:
  - Ingress
```

```bash
# Aplicar
kubectl apply -f deny-all-ingress.yaml

# Agora NENHUM tráfego de entrada é permitido para pods no namespace default
```

**Weave Net implementa as policies via iptables.**

---

## 📦 Instalação e Configuração

### Instalar Weave Net

**Método 1: kubectl apply (mais comum)**

```bash
# Instalar Weave Net
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Verificar instalação
kubectl get pods -n kube-system -l name=weave-net

# Output:
# NAME              READY   STATUS    RESTARTS   AGE
# weave-net-abc12   2/2     Running   0          1m
# weave-net-xyz34   2/2     Running   0          1m
```

**Método 2: Weave Launcher Script**

```bash
# Baixar e executar
kubectl apply -f "https://cloud.weave.works/k8s/net?k8s-version=$(kubectl version | base64 | tr -d '\n')"
```

### Verificar Status do Weave

```bash
# Ver pods do Weave
kubectl get pods -n kube-system -l name=weave-net

# Ver logs do Weave
kubectl logs -n kube-system weave-net-xxxxx -c weave

# Ver status da rede Weave
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status

# Output:
#         Version: 2.8.1 (up to date)
#       Service: router
#      Protocol: weave 1..2
#          Name: 7a:ab:cd:ef:12:34(node01)
#   Encryption: disabled
#   PeerDiscovery: enabled
```

### Verificar Configuração CNI

```bash
# Ver configuração CNI criada pelo Weave
cat /etc/cni/net.d/10-weave.conflist

# Output:
# {
#   "cniVersion": "0.3.0",
#   "name": "weave",
#   "plugins": [
#     {
#       "name": "weave",
#       "type": "weave-net",
#       "hairpinMode": true
#     },
#     {
#       "type": "portmap",
#       "capabilities": {"portMappings": true},
#       "snat": true
#     }
#   ]
# }
```

### Configurar Range de IPs Customizado

Por padrão, Weave usa `10.32.0.0/12`. Para customizar:

```bash
# Download do manifest
wget https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Editar o manifest
vi weave-daemonset-k8s.yaml

# Adicionar variável de ambiente IPALLOC_RANGE no container weave
# ...
# env:
#   - name: IPALLOC_RANGE
#     value: 10.50.0.0/16
# ...

# Aplicar
kubectl apply -f weave-daemonset-k8s.yaml
```

### Configurar Encryption

```bash
# Criar secret com senha
kubectl create secret -n kube-system generic weave-passwd \
  --from-literal=weave-passwd=$(openssl rand -base64 16)

# Weave detecta automaticamente e habilita encryption
# Verificar logs
kubectl logs -n kube-system weave-net-xxxxx -c weave | grep -i encr

# Output:
# INFO: 2023/10/01 10:00:00.123456 encryption enabled
```

---

## 🔍 Troubleshooting

### 1. Pods não conseguem se comunicar

**Verificar:**

```bash
# 1. Weave Net está rodando?
kubectl get pods -n kube-system -l name=weave-net

# 2. Ver logs do Weave
kubectl logs -n kube-system weave-net-xxxxx -c weave

# 3. Verificar status
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status

# 4. Verificar peers (connections)
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status connections

# Output deve mostrar conexões com outros nodes:
# <- 192.168.1.11:6783   established fastdp
# <- 192.168.1.12:6783   established fastdp
```

### 2. CNI plugin não está sendo chamado

**Verificar:**

```bash
# 1. Binário CNI existe?
ls -la /opt/cni/bin/weave-*

# 2. Configuração CNI existe?
ls -la /etc/cni/net.d/

# 3. Ver logs do kubelet (procurar por erros CNI)
journalctl -u kubelet -f | grep -i cni
```

### 3. Range de IPs esgotado

**Verificar:**

```bash
# Ver range configurado
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status ipam

# Output:
# 10.32.0.0/12:
#   - 10.32.0.0/12 (owned by node01)
#   - 10.40.0.0/12 (owned by node02)

# Se range está pequeno, aumentar IPALLOC_RANGE
# Exemplo: mudar de /12 para /10
# 10.32.0.0/10 = 4.194.304 IPs (4x mais)
```

### 4. Weave não conecta entre nodes

**Problemas comuns:**

**Firewall bloqueando portas:**

Weave usa:
- **TCP 6783**: controle e dados
- **UDP 6783**: sleeve (fallback) e encryption
- **UDP 6784**: DNS discovery

```bash
# Verificar se portas estão abertas
sudo netstat -tulpn | grep 6783

# Abrir portas no firewall
sudo ufw allow 6783/tcp
sudo ufw allow 6783/udp
sudo ufw allow 6784/udp
```

**Verificar conectividade:**

```bash
# De um node, tentar conectar em outro
nc -zv 192.168.1.11 6783

# Output:
# Connection to 192.168.1.11 6783 port [tcp/*] succeeded!
```

### 5. DNS não funciona

**Weave tem seu próprio DNS (WeaveDNS) - mas geralmente usa CoreDNS do Kubernetes.**

```bash
# Verificar CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Testar DNS de dentro de um pod
kubectl run tmp --rm -i --tty --image=busybox -- sh
nslookup kubernetes.default

# Se falhar, verificar logs do CoreDNS
kubectl logs -n kube-system coredns-xxxxx
```

### 6. Performance baixa

**Weave pode usar dois modos de datapath:**

1. **fastdp** (fast datapath): usa kernel VXLAN (mais rápido)
2. **sleeve**: userspace overlay (mais lento, fallback)

**Verificar modo:**

```bash
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status connections

# Output:
# <- 192.168.1.11:6783   established fastdp    (✅ bom)
# <- 192.168.1.12:6783   established sleeve    (⚠️ mais lento)
```

**Forçar fastdp:**

Certifique-se que:
- Kernel suporta VXLAN (Linux ≥3.12)
- MTU está configurado corretamente

```bash
# Verificar MTU
ip link show eth0

# MTU deve ser ≥1376 para VXLAN funcionar
# Se menor, aumentar:
sudo ip link set dev eth0 mtu 1500
```

---

## 📝 Resumo

### CNI (Container Network Interface)

**O que é:**
- Especificação para configurar redes de containers
- Padrão usado por Kubernetes, Podman, CRI-O, etc

**Responsabilidades:**
- ✅ Criar interface de rede
- ✅ Alocar IP
- ✅ Configurar rotas
- ❌ Não faz port mapping ou network policies (isso é extra)

**Localização:**
- **Binários**: `/opt/cni/bin/`
- **Configuração**: `/etc/cni/net.d/`

### Weave Net

**Características:**
- Overlay network (VXLAN/UDP)
- Range padrão: `10.32.0.0/12`
- Suporta encryption (NaCl/IPsec)
- Suporta Network Policies
- DaemonSet em cada node

**Instalação:**
```bash
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
```

**Verificar:**
```bash
kubectl get pods -n kube-system -l name=weave-net
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status
```

**Portas:**
- TCP/UDP 6783: controle + dados
- UDP 6784: DNS discovery

**Datapaths:**
- **fastdp**: kernel VXLAN (rápido) ✅
- **sleeve**: userspace (lento, fallback) ⚠️

---

## 🔧 Comandos Essenciais

### CNI

```bash
# Ver binários CNI
ls -la /opt/cni/bin/

# Ver configurações CNI
ls -la /etc/cni/net.d/
cat /etc/cni/net.d/10-weave.conflist
```

### Weave Net

```bash
# Instalar
kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml

# Ver pods
kubectl get pods -n kube-system -l name=weave-net

# Status
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status

# Connections
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status connections

# IPAM (IP ranges)
kubectl exec -n kube-system weave-net-xxxxx -c weave -- /home/weave/weave --local status ipam

# Logs
kubectl logs -n kube-system weave-net-xxxxx -c weave
```

---

## 💡 Dicas para o Exame CKA

1. **CNI é só uma especificação**
   - Plugins (Weave, Calico, Flannel) implementam a especificação
   - Binários em `/opt/cni/bin/`, config em `/etc/cni/net.d/`

2. **Weave Net é popular no CKA**
   - Fácil de instalar (um único kubectl apply)
   - Não precisa de configuração externa
   - Suporta encryption e Network Policies

3. **Troubleshooting CNI**
   - Ver logs do kubelet: `journalctl -u kubelet | grep -i cni`
   - Ver logs do plugin: `kubectl logs -n kube-system weave-net-xxxxx -c weave`
   - Verificar binários: `ls /opt/cni/bin/`
   - Verificar config: `cat /etc/cni/net.d/*.conflist`

4. **Range de IPs**
   - Weave default: `10.32.0.0/12` (1M IPs)
   - Customizar via `IPALLOC_RANGE` env var
   - Verificar: `weave --local status ipam`

5. **Portas do Weave**
   - TCP/UDP **6783**: comunicação entre nodes
   - UDP **6784**: DNS discovery
   - Firewall deve permitir essas portas

6. **Performance**
   - **fastdp** (VXLAN kernel) = rápido ✅
   - **sleeve** (userspace) = lento ⚠️
   - Verificar: `weave --local status connections`

---

⬅️ **Anterior**: [network-fundamentals.md](./network-fundamentals.md) | ➡️ **Próximo**: [network-policies.md](./network-policies.md)
