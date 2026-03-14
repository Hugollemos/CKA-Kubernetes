# Fundamentos de Redes para Kubernetes

Este documento cobre conceitos básicos de redes essenciais para entender networking no Kubernetes.

## 📚 Conteúdo

1. [Switching e Routing](#switching-e-routing)
2. [Default Gateway](#default-gateway)
3. [DNS (Domain Name System)](#dns-domain-name-system)
4. [Network Namespaces](#network-namespaces)
5. [Docker Networking](#docker-networking)
6. [Relação com Kubernetes](#relação-com-kubernetes)

---

## 🔌 Switching e Routing

### O que é um Switch?

**Switch** é um dispositivo de rede que conecta hosts na **mesma rede local (LAN)**.

```
┌──────────────┐      ┌──────────────┐      ┌──────────────┐
│   Host A     │      │   Host B     │      │   Host C     │
│ 192.168.1.10 │      │ 192.168.1.11 │      │ 192.168.1.12 │
└──────┬───────┘      └──────┬───────┘      └──────┬───────┘
       │                     │                     │
       └─────────────────────┼─────────────────────┘
                             │
                      ┌──────▼───────┐
                      │    Switch    │
                      └──────────────┘
```

**Características:**
- Opera na **Camada 2** (Data Link) do modelo OSI
- Usa **endereços MAC** para encaminhar pacotes
- Conecta hosts **na mesma rede** (mesmo segmento)
- Não pode conectar redes diferentes

### Criar Interface de Rede no Linux

```bash
# Ver interfaces de rede
ip link

# Output:
# 1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536
# 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500
# 3: eth1: <BROADCAST,MULTICAST> mtu 1500

# Ver endereços IP das interfaces
ip addr

# Adicionar endereço IP a uma interface
sudo ip addr add 192.168.1.10/24 dev eth0

# Ativar interface
sudo ip link set dev eth0 up

# Desativar interface
sudo ip link set dev eth0 down
```

### Conectar Dois Hosts Diretamente

```bash
# === Host A (192.168.1.10) ===
sudo ip addr add 192.168.1.10/24 dev eth0
sudo ip link set dev eth0 up

# === Host B (192.168.1.11) ===
sudo ip addr add 192.168.1.11/24 dev eth0
sudo ip link set dev eth0 up

# Testar conectividade de A para B
ping 192.168.1.11

# Testar de B para A
ping 192.168.1.10
```

**Nota:** `/24` significa **subnet mask 255.255.255.0** (primeiros 24 bits são rede, últimos 8 bits são hosts).

### O que é um Router?

**Router** é um dispositivo que conecta **diferentes redes**.

```
┌────────────────────────────────┐         ┌────────────────────────────────┐
│     Rede 192.168.1.0/24        │         │     Rede 192.168.2.0/24        │
│                                │         │                                │
│  ┌──────────┐   ┌──────────┐  │         │  ┌──────────┐   ┌──────────┐  │
│  │  Host A  │   │  Host B  │  │         │  │  Host C  │   │  Host D  │  │
│  │   .10    │   │   .11    │  │         │  │   .10    │   │   .11    │  │
│  └────┬─────┘   └────┬─────┘  │         │  └────┬─────┘   └────┬─────┘  │
│       │              │         │         │       │              │         │
│       └──────┬───────┘         │         │       └──────┬───────┘         │
│              │                 │         │              │                 │
│        ┌─────▼─────┐           │         │        ┌─────▼─────┐           │
│        │  Switch   │           │         │        │  Switch   │           │
│        └─────┬─────┘           │         │        └─────┬─────┘           │
│              │                 │         │              │                 │
│         192.168.1.1            │         │         192.168.2.1            │
└──────────────┼─────────────────┘         └──────────────┼─────────────────┘
               │                                          │
               │         ┌────────────────┐               │
               └─────────┤     Router     ├───────────────┘
                         │  192.168.1.1   │
                         │  192.168.2.1   │
                         └────────────────┘
```

**Características:**
- Opera na **Camada 3** (Network) do modelo OSI
- Usa **endereços IP** para encaminhar pacotes
- Conecta **redes diferentes**
- Toma decisões de roteamento baseadas em tabelas de rotas

### Routing Table (Tabela de Rotas)

A **routing table** armazena informações sobre como alcançar diferentes redes.

```bash
# Ver tabela de rotas
ip route
# ou
route -n

# Output:
# Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
# 0.0.0.0         192.168.1.1     0.0.0.0         UG    100    0        0 eth0
# 192.168.1.0     0.0.0.0         255.255.255.0   U     100    0        0 eth0
```

**Colunas importantes:**
- **Destination**: rede de destino
- **Gateway**: próximo hop (router) para alcançar a rede
- **Genmask**: subnet mask da rede
- **Iface**: interface de saída

### Adicionar Rota Estática

```bash
# Adicionar rota para rede 192.168.2.0/24 via gateway 192.168.1.1
sudo ip route add 192.168.2.0/24 via 192.168.1.1

# Ver rotas
ip route

# Deletar rota
sudo ip route del 192.168.2.0/24
```

### Exemplo Prático: Conectar Duas Redes

**Cenário:** Host A (192.168.1.10) quer acessar Host C (192.168.2.10)

```bash
# === Router (192.168.1.1 e 192.168.2.1) ===
# Habilitar IP forwarding (permitir que router encaminhe pacotes)
sudo sysctl -w net.ipv4.ip_forward=1

# Tornar permanente
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf

# === Host A (192.168.1.10) ===
# Adicionar rota padrão para acessar outras redes via router
sudo ip route add default via 192.168.1.1

# Testar conectividade para Host C
ping 192.168.2.10

# === Host C (192.168.2.10) ===
# Adicionar rota padrão para acessar outras redes via router
sudo ip route add default via 192.168.2.1

# Testar conectividade para Host A
ping 192.168.1.10
```

---

## 🚪 Default Gateway

**Default Gateway** é o router usado quando o destino **não está na rede local**.

### Como funciona?

1. **Host verifica**: destino está na mesma rede?
   - Se **SIM**: envia diretamente via switch
   - Se **NÃO**: envia para o default gateway

2. **Default gateway (router)** recebe o pacote e:
   - Consulta sua routing table
   - Encaminha para o próximo hop

### Configurar Default Gateway

```bash
# Ver gateway padrão atual
ip route | grep default

# Output:
# default via 192.168.1.1 dev eth0

# Adicionar default gateway
sudo ip route add default via 192.168.1.1

# Deletar default gateway
sudo ip route del default

# Adicionar com métrica (prioridade)
sudo ip route add default via 192.168.1.1 metric 100
```

### Exemplo: Host sem Default Gateway

```bash
# Host A (192.168.1.10) sem default gateway
ping 192.168.1.11   # ✅ Funciona (mesma rede)
ping 8.8.8.8        # ❌ Falha (rede diferente, sem gateway)

# Network is unreachable

# Adicionar default gateway
sudo ip route add default via 192.168.1.1

# Agora funciona
ping 8.8.8.8        # ✅ Funciona (via gateway)
```

### Múltiplos Default Gateways

```bash
# Adicionar múltiplos gateways com métricas (menor = maior prioridade)
sudo ip route add default via 192.168.1.1 metric 100
sudo ip route add default via 192.168.1.2 metric 200

# Gateway 192.168.1.1 será usado primeiro (métrica menor)
# Se falhar, usa 192.168.1.2
```

---

## 🌐 DNS (Domain Name System)

### O que é DNS?

**DNS** traduz **nomes de domínio** (humanos) para **endereços IP** (máquinas).

```
www.google.com  →  DNS  →  142.250.185.46
```

### Por que usar DNS?

```bash
# Sem DNS (difícil de lembrar)
ping 142.250.185.46

# Com DNS (fácil de lembrar)
ping www.google.com
```

### Arquivo /etc/hosts (DNS Local)

O arquivo `/etc/hosts` mapeia nomes para IPs **localmente** (sem servidor DNS).

```bash
# Ver /etc/hosts
cat /etc/hosts

# Conteúdo:
# 127.0.0.1       localhost
# 192.168.1.10    db
# 192.168.1.11    web
# 192.168.1.12    api
```

**Usar nomes:**

```bash
# Ao invés de:
ping 192.168.1.10

# Use:
ping db
```

### Adicionar Entrada no /etc/hosts

```bash
# Editar arquivo
sudo vi /etc/hosts

# Adicionar:
192.168.1.100    myserver
192.168.1.101    database.local

# Testar
ping myserver
# Funciona! Resolve para 192.168.1.100
```

**Limitações do /etc/hosts:**
- ❌ Precisa replicar em todos os hosts
- ❌ Difícil de gerenciar em larga escala
- ✅ **Solução**: usar servidor DNS

### DNS Server (Servidor DNS)

**DNS Server** centraliza a resolução de nomes.

```
┌─────────────┐      ┌─────────────┐      ┌─────────────┐
│   Host A    │      │   Host B    │      │   Host C    │
│             │      │             │      │             │
└──────┬──────┘      └──────┬──────┘      └──────┬──────┘
       │                    │                    │
       │    Consulta DNS    │                    │
       └────────────────────┼────────────────────┘
                            │
                      ┌─────▼──────┐
                      │ DNS Server │
                      │ 192.168.1.5│
                      └────────────┘
```

### Configurar DNS no Linux

Arquivo `/etc/resolv.conf` configura servidores DNS.

```bash
# Ver configuração DNS
cat /etc/resolv.conf

# Conteúdo:
# nameserver 192.168.1.5
# nameserver 8.8.8.8
# nameserver 8.8.4.4
```

**Ordem de resolução:**
1. Tenta `192.168.1.5` primeiro
2. Se falhar, tenta `8.8.8.8`
3. Se falhar, tenta `8.8.4.4`

### Adicionar DNS Server

```bash
# Editar /etc/resolv.conf
sudo vi /etc/resolv.conf

# Adicionar:
nameserver 1.1.1.1         # Cloudflare DNS
nameserver 8.8.8.8         # Google DNS
nameserver 8.8.4.4         # Google DNS secundário

# Testar
nslookup www.google.com
```

### Ordem de Resolução: /etc/hosts vs DNS

Por padrão, o Linux consulta **primeiro** `/etc/hosts`, **depois** DNS.

Configurado em `/etc/nsswitch.conf`:

```bash
cat /etc/nsswitch.conf

# Linha importante:
# hosts:    files dns
#           ^     ^
#           |     +--- Depois consulta DNS (/etc/resolv.conf)
#           +--------- Primeiro consulta /etc/hosts (files)
```

**Exemplo:**

```bash
# /etc/hosts tem:
192.168.1.100    www.google.com

# /etc/resolv.conf tem:
nameserver 8.8.8.8

# Testar:
ping www.google.com
# Resolve para 192.168.1.100 (de /etc/hosts, NÃO de DNS!)
```

### DNS Record Types

| Tipo | Nome | Descrição | Exemplo |
|------|------|-----------|---------|
| **A** | Address | Mapeia nome → IPv4 | `www.example.com → 93.184.216.34` |
| **AAAA** | IPv6 Address | Mapeia nome → IPv6 | `www.example.com → 2606:2800:220:1:248:1893:25c8:1946` |
| **CNAME** | Canonical Name | Alias para outro nome | `blog.example.com → www.example.com` |
| **MX** | Mail Exchange | Servidor de email | `example.com → mail.example.com` |
| **NS** | Name Server | Servidor DNS autoritativo | `example.com → ns1.example.com` |
| **TXT** | Text | Texto arbitrário | `example.com → "v=spf1 include:_spf.google.com ~all"` |

### Testar DNS

```bash
# nslookup (simples)
nslookup www.google.com

# dig (detalhado)
dig www.google.com

# host (resumido)
host www.google.com

# getent (usa /etc/hosts + DNS)
getent hosts www.google.com
```

### Exemplo: nslookup

```bash
nslookup www.google.com

# Output:
# Server:         8.8.8.8
# Address:        8.8.8.8#53
#
# Non-authoritative answer:
# Name:   www.google.com
# Address: 142.250.185.46
```

### Exemplo: dig

```bash
dig www.google.com

# Output:
# ; <<>> DiG 9.16.1-Ubuntu <<>> www.google.com
# ;; QUESTION SECTION:
# ;www.google.com.                        IN      A
#
# ;; ANSWER SECTION:
# www.google.com.         300     IN      A       142.250.185.46
```

### DNS no Kubernetes: CoreDNS

**CoreDNS** é o servidor DNS padrão do Kubernetes.

```bash
# Ver pods do CoreDNS
kubectl get pods -n kube-system | grep coredns

# Output:
# coredns-5d78c9869d-abc12   1/1     Running   0          5d
# coredns-5d78c9869d-xyz34   1/1     Running   0          5d

# Ver configuração do CoreDNS
kubectl get configmap coredns -n kube-system -o yaml

# Ver logs do CoreDNS
kubectl logs -n kube-system coredns-5d78c9869d-abc12
```

**CoreDNS resolve:**
- **Services**: `my-service.default.svc.cluster.local`
- **Pods**: `10-244-1-5.default.pod.cluster.local` (IP do pod com `-` ao invés de `.`)

**Exemplo:**

```bash
# Criar Service
kubectl create service clusterip my-service --tcp=80:80

# Dentro de um pod, testar DNS
kubectl run tmp --rm -i --tty --image=busybox -- sh

# Dentro do container:
nslookup my-service
# Output:
# Server:    10.96.0.10  (ClusterIP do CoreDNS)
# Address 1: 10.96.0.10 kube-dns.kube-system.svc.cluster.local
#
# Name:      my-service
# Address 1: 10.96.100.50 my-service.default.svc.cluster.local

# Testar acesso
wget -O- http://my-service
```

**Formato FQDN no Kubernetes:**

```
<service-name>.<namespace>.svc.<cluster-domain>
     |             |         |         |
     |             |         |         +--- cluster.local (padrão)
     |             |         +------------- "svc" (para services)
     |             +----------------------- namespace
     +------------------------------------- nome do service
```

**Exemplos:**

```bash
# Mesmo namespace
curl http://nginx

# Namespace específico
curl http://nginx.default

# FQDN completo
curl http://nginx.default.svc.cluster.local
```

---

## 🔒 Network Namespaces

### O que são Network Namespaces?

**Network Namespaces** isolam recursos de rede em um host Linux.

**Cada namespace tem:**
- Suas próprias **interfaces de rede**
- Sua própria **routing table**
- Suas próprias **regras de iptables**
- Seus próprios **sockets de rede**

```
┌────────────────────────────────────────────────────────────┐
│                      Host Linux                            │
│                                                            │
│  ┌──────────────────┐         ┌──────────────────┐        │
│  │  Namespace RED   │         │  Namespace BLUE  │        │
│  │                  │         │                  │        │
│  │  ┌────────────┐  │         │  ┌────────────┐  │        │
│  │  │   eth0     │  │         │  │   eth0     │  │        │
│  │  │ 10.0.1.10  │  │         │  │ 10.0.2.10  │  │        │
│  │  └────────────┘  │         │  └────────────┘  │        │
│  │                  │         │                  │        │
│  │  Route Table     │         │  Route Table     │        │
│  │  iptables        │         │  iptables        │        │
│  └──────────────────┘         └──────────────────┘        │
│                                                            │
└────────────────────────────────────────────────────────────┘
```

### Por que usar Network Namespaces?

**Casos de uso:**
1. **Containers**: cada container tem seu próprio namespace de rede
2. **Isolamento**: processos em namespaces diferentes não podem se comunicar (a menos que explicitamente conectados)
3. **Redes sobrepostas**: usar mesmos IPs em namespaces diferentes

### Criar e Gerenciar Network Namespaces

```bash
# Criar namespace
sudo ip netns add red
sudo ip netns add blue

# Listar namespaces
ip netns list
# Output:
# red
# blue

# Executar comando em um namespace
sudo ip netns exec red ip link
# Lista interfaces de rede APENAS do namespace red

# Deletar namespace
sudo ip netns del red
```

### Verificar Interfaces em um Namespace

```bash
# Ver interfaces no namespace red
sudo ip netns exec red ip link

# Output (apenas loopback):
# 1: lo: <LOOPBACK> mtu 65536 qdisc noop state DOWN
```

**Nota:** Por padrão, um novo namespace só tem a interface `lo` (loopback) desabilitada.

### Habilitar Loopback em um Namespace

```bash
# Habilitar loopback
sudo ip netns exec red ip link set dev lo up

# Testar
sudo ip netns exec red ping 127.0.0.1
# ✅ Funciona
```

### Conectar Dois Network Namespaces

Para conectar dois namespaces, usamos um **veth pair** (virtual ethernet pair).

**veth pair** é como um "cabo virtual" com duas pontas.

```
┌──────────────────┐                    ┌──────────────────┐
│  Namespace RED   │                    │  Namespace BLUE  │
│                  │                    │                  │
│  ┌────────────┐  │    veth pair       │  ┌────────────┐  │
│  │   veth-r   │◄─┼────────────────────┼─►│   veth-b   │  │
│  │ 10.0.1.1   │  │                    │  │ 10.0.1.2   │  │
│  └────────────┘  │                    │  └────────────┘  │
└──────────────────┘                    └──────────────────┘
```

**Passo a passo:**

```bash
# 1. Criar veth pair
sudo ip link add veth-r type veth peer name veth-b

# 2. Verificar (ainda no namespace padrão)
ip link
# veth-r e veth-b estão visíveis

# 3. Mover veth-r para namespace red
sudo ip link set veth-r netns red

# 4. Mover veth-b para namespace blue
sudo ip link set veth-b netns blue

# 5. Verificar que sumiram do namespace padrão
ip link
# veth-r e veth-b NÃO aparecem mais

# 6. Verificar que estão nos namespaces
sudo ip netns exec red ip link
# veth-r aparece

sudo ip netns exec blue ip link
# veth-b aparece

# 7. Atribuir endereços IP
sudo ip netns exec red ip addr add 10.0.1.1/24 dev veth-r
sudo ip netns exec blue ip addr add 10.0.1.2/24 dev veth-b

# 8. Ativar interfaces
sudo ip netns exec red ip link set dev veth-r up
sudo ip netns exec blue ip link set dev veth-b up

# 9. Testar conectividade
sudo ip netns exec red ping 10.0.1.2
# ✅ Funciona! Red pode pingar Blue

sudo ip netns exec blue ping 10.0.1.1
# ✅ Funciona! Blue pode pingar Red
```

### Conectar Namespaces via Linux Bridge

Para conectar **múltiplos** namespaces, usamos um **Linux Bridge** (switch virtual).

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│ Namespace 1  │     │ Namespace 2  │     │ Namespace 3  │
│              │     │              │     │              │
│ ┌──────────┐ │     │ ┌──────────┐ │     │ ┌──────────┐ │
│ │  veth-1  │ │     │ │  veth-2  │ │     │ │  veth-3  │ │
│ │10.0.1.10 │ │     │ │10.0.1.11 │ │     │ │10.0.1.12 │ │
│ └────┬─────┘ │     │ └────┬─────┘ │     │ └────┬─────┘ │
└──────┼───────┘     └──────┼───────┘     └──────┼───────┘
       │                    │                    │
       │                    │                    │
       └────────────────────┼────────────────────┘
                            │
                     ┌──────▼──────┐
                     │ Linux Bridge│
                     │   (v-net-0) │
                     │  10.0.1.1   │
                     └─────────────┘
```

**Passo a passo:**

```bash
# 1. Criar bridge
sudo ip link add v-net-0 type bridge

# 2. Ativar bridge
sudo ip link set dev v-net-0 up

# 3. Criar namespaces
sudo ip netns add red
sudo ip netns add blue

# 4. Criar veth pairs
sudo ip link add veth-r type veth peer name veth-r-br
sudo ip link add veth-b type veth peer name veth-b-br

# 5. Mover uma ponta para namespace
sudo ip link set veth-r netns red
sudo ip link set veth-b netns blue

# 6. Conectar outra ponta ao bridge
sudo ip link set veth-r-br master v-net-0
sudo ip link set veth-b-br master v-net-0

# 7. Atribuir IPs
sudo ip netns exec red ip addr add 10.0.1.10/24 dev veth-r
sudo ip netns exec blue ip addr add 10.0.1.11/24 dev veth-b

# 8. Ativar interfaces
sudo ip netns exec red ip link set dev veth-r up
sudo ip netns exec blue ip link set dev veth-b up
sudo ip link set dev veth-r-br up
sudo ip link set dev veth-b-br up

# 9. Adicionar IP ao bridge (para host acessar namespaces)
sudo ip addr add 10.0.1.1/24 dev v-net-0

# 10. Testar conectividade
sudo ip netns exec red ping 10.0.1.11   # Red → Blue ✅
sudo ip netns exec blue ping 10.0.1.10  # Blue → Red ✅
sudo ip netns exec red ping 10.0.1.1    # Red → Host ✅
ping 10.0.1.10                          # Host → Red ✅
```

### Conectar Namespace à Internet

Para namespaces acessarem a internet, precisamos de **NAT (Network Address Translation)**.

```bash
# 1. Adicionar default gateway no namespace
sudo ip netns exec red ip route add default via 10.0.1.1

# 2. Habilitar IP forwarding no host
sudo sysctl -w net.ipv4.ip_forward=1

# 3. Adicionar regra de NAT (iptables)
sudo iptables -t nat -A POSTROUTING -s 10.0.1.0/24 -j MASQUERADE

# 4. Testar acesso à internet
sudo ip netns exec red ping 8.8.8.8
# ✅ Funciona!
```

**Explicação da regra iptables:**
- `-t nat`: tabela NAT
- `-A POSTROUTING`: adicionar regra na chain POSTROUTING
- `-s 10.0.1.0/24`: para pacotes originados da rede 10.0.1.0/24
- `-j MASQUERADE`: fazer NAT (substituir IP source pelo IP do host)

---

## 🐳 Docker Networking

### Como Docker Usa Network Namespaces

**Cada container Docker roda em seu próprio network namespace.**

```bash
# Criar container
docker run -d --name nginx nginx

# Ver namespaces (indiretamente via PID do container)
docker inspect nginx --format '{{.State.Pid}}'
# Output: 12345

# Acessar namespace do container
sudo nsenter -t 12345 -n ip addr
# Mostra interfaces de rede DO CONTAINER
```

### Docker Network Drivers

Docker suporta diferentes **network drivers**:

| Driver | Descrição | Isolamento | Use Case |
|--------|-----------|------------|----------|
| **bridge** | Rede privada isolada (padrão) | ✅ Sim | Containers no mesmo host |
| **host** | Usa rede do host diretamente | ❌ Não | Performance máxima |
| **none** | Sem rede | ✅ Completo | Casos especiais |
| **overlay** | Rede entre múltiplos hosts (Swarm) | ✅ Sim | Clusters Docker Swarm |
| **macvlan** | Atribui MAC address ao container | ⚠️ Parcial | Containers aparecem como hosts físicos |

### Docker Bridge Network

**Bridge** é o modo padrão. Docker cria um bridge `docker0`.

```bash
# Ver bridge do Docker
ip link show docker0

# Ver rede bridge do Docker
docker network ls
# Output:
# NETWORK ID     NAME      DRIVER    SCOPE
# abc123def456   bridge    bridge    local

# Inspecionar rede bridge
docker network inspect bridge
```

**Arquitetura:**

```
┌────────────────────────────────────────────────────────────┐
│                      Host Linux                            │
│                                                            │
│  ┌──────────────────┐         ┌──────────────────┐        │
│  │  Container 1     │         │  Container 2     │        │
│  │                  │         │                  │        │
│  │  ┌────────────┐  │         │  ┌────────────┐  │        │
│  │  │   eth0     │  │         │  │   eth0     │  │        │
│  │  │172.17.0.2  │  │         │  │172.17.0.3  │  │        │
│  │  └─────┬──────┘  │         │  └─────┬──────┘  │        │
│  └────────┼─────────┘         └────────┼─────────┘        │
│           │        veth pair           │                  │
│           │                            │                  │
│        ┌──▼────────────────────────────▼──┐               │
│        │      docker0 bridge              │               │
│        │       172.17.0.1                 │               │
│        └──────────────┬───────────────────┘               │
│                       │                                   │
│                  ┌────▼────┐                              │
│                  │  eth0   │ (interface física)           │
│                  └─────────┘                              │
└────────────────────────────────────────────────────────────┘
```

**Fluxo de rede:**
1. Container cria pacote para internet
2. Pacote sai pela interface `eth0` do container
3. Passa pelo veth pair até `docker0` bridge
4. `docker0` encaminha para interface física do host (com NAT)
5. Sai para internet

### Port Mapping

```bash
# Container sem port mapping (acessível apenas internamente)
docker run -d --name nginx nginx

# Container COM port mapping (acessível de fora)
docker run -d --name nginx -p 8080:80 nginx
#                              ^    ^
#                              |    +--- Porta do container
#                              +-------- Porta do host

# Acessar de fora
curl http://localhost:8080
```

**Como funciona port mapping:**

```bash
# Docker cria regra iptables NAT
sudo iptables -t nat -L -n | grep 8080

# Output:
# DNAT  tcp  --  0.0.0.0/0  0.0.0.0/0  tcp dpt:8080 to:172.17.0.2:80
```

### Docker Host Network

Container usa **diretamente** a rede do host (sem namespace separado).

```bash
# Container com rede do host
docker run -d --name nginx --network host nginx

# Nginx escuta na porta 80 DIRETAMENTE no host
curl http://localhost:80
# ✅ Funciona sem port mapping!
```

**Vantagens:**
- ✅ Melhor performance (sem overhead de NAT)
- ✅ Sem necessidade de port mapping

**Desvantagens:**
- ❌ Sem isolamento de rede
- ❌ Conflito de portas entre containers

### Docker None Network

Container **sem rede** (apenas loopback).

```bash
# Container sem rede
docker run -d --name isolated --network none nginx

# Tentar acessar internet de dentro do container
docker exec isolated ping 8.8.8.8
# ❌ Falha: network unreachable
```

**Use case:** containers que não precisam de rede (processamento local, batch jobs).

---

## ☸️ Relação com Kubernetes

### Kubernetes CNI (Container Network Interface)

**CNI** é um padrão para configurar redes de containers. Kubernetes usa CNI plugins.

**Plugins CNI populares:**
- **Flannel**: overlay network simples
- **Calico**: networking + network policies
- **Weave Net**: overlay network com encryption
- **Cilium**: eBPF-based networking
- **Kindnet**: simples, usado pelo Kind

### Como Kubernetes Usa Network Namespaces

**Cada Pod tem seu próprio network namespace.**

```bash
# Ver pod
kubectl get pods

# Ver onde pod está rodando
kubectl get pod nginx -o wide
# Output:
# NAME    READY   STATUS    NODE
# nginx   1/1     Running   node01

# SSH no node
ssh node01

# Encontrar PID do container do pod
docker ps | grep nginx
# ou
crictl ps | grep nginx

# Acessar namespace de rede do pod
sudo nsenter -t <PID> -n ip addr
```

### Pod Networking no Kubernetes

**Requisitos de rede do Kubernetes:**
1. **Todos os Pods** podem se comunicar entre si **sem NAT**
2. **Todos os Nodes** podem se comunicar com todos os Pods **sem NAT**
3. **IP que o Pod vê** é o **mesmo IP que outros veem**

```
┌────────────────────────────────────────────────────────────┐
│                        Node 1                              │
│                                                            │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │   Pod A      │    │   Pod B      │    │   Pod C      │ │
│  │  10.244.1.5  │    │  10.244.1.6  │    │  10.244.1.7  │ │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘ │
│         │                   │                   │         │
│         └───────────────────┼───────────────────┘         │
│                             │                             │
│                       ┌─────▼──────┐                      │
│                       │   CNI      │                      │
│                       │  Bridge    │                      │
│                       └─────┬──────┘                      │
│                             │                             │
└─────────────────────────────┼─────────────────────────────┘
                              │
        ┌─────────────────────┼─────────────────────┐
        │                     │                     │
┌───────▼────────┐   ┌────────▼────────┐   ┌───────▼────────┐
│     Node 2     │   │     Node 3      │   │     Node 4     │
│  10.244.2.0/24 │   │  10.244.3.0/24  │   │  10.244.4.0/24 │
└────────────────┘   └─────────────────┘   └────────────────┘
```

### DNS no Kubernetes (CoreDNS)

**CoreDNS** roda como Deployment no namespace `kube-system`.

```bash
# Ver pods do CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Ver ConfigMap do CoreDNS
kubectl get cm coredns -n kube-system -o yaml
```

**Pods automaticamente usam CoreDNS:**

```bash
# Dentro de um pod, ver /etc/resolv.conf
kubectl exec -it nginx -- cat /etc/resolv.conf

# Output:
# nameserver 10.96.0.10   (ClusterIP do CoreDNS)
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

**DNS Records criados automaticamente:**

| Recurso | DNS Name | Exemplo |
|---------|----------|---------|
| **Service** | `<service>.<namespace>.svc.cluster.local` | `nginx.default.svc.cluster.local` |
| **Pod** | `<pod-ip-with-dashes>.<namespace>.pod.cluster.local` | `10-244-1-5.default.pod.cluster.local` |
| **Headless Service** | `<pod-name>.<service>.<namespace>.svc.cluster.local` | `web-0.nginx.default.svc.cluster.local` |

---

## 📝 Resumo

### Switching
- Conecta hosts na **mesma rede**
- Usa **endereços MAC**
- Camada 2 (Data Link)

### Routing
- Conecta **redes diferentes**
- Usa **endereços IP**
- Camada 3 (Network)
- **Default gateway**: router padrão para redes externas

### DNS
- Traduz **nomes** → **IPs**
- `/etc/hosts`: DNS local
- `/etc/resolv.conf`: configuração de DNS servers
- **CoreDNS**: servidor DNS do Kubernetes

### Network Namespaces
- Isolam recursos de rede
- Cada namespace tem suas próprias interfaces, rotas, iptables
- **veth pair**: conecta dois namespaces
- **Linux bridge**: conecta múltiplos namespaces (switch virtual)

### Docker Networking
- Cada container = um network namespace
- **Bridge**: rede isolada privada (padrão)
- **Host**: usa rede do host diretamente
- **None**: sem rede
- Port mapping: `-p host:container`

### Kubernetes Networking
- Cada Pod = um network namespace
- **CNI**: padrão para plugins de rede
- Todos os Pods podem se comunicar diretamente (sem NAT)
- **CoreDNS**: DNS do cluster
- **Services**: abstração de rede com DNS automático

---

## 🔧 Comandos Essenciais

### Networking Básico

```bash
# Ver interfaces
ip link
ip addr

# Adicionar IP
sudo ip addr add 192.168.1.10/24 dev eth0

# Ver rotas
ip route

# Adicionar rota
sudo ip route add 192.168.2.0/24 via 192.168.1.1

# Adicionar default gateway
sudo ip route add default via 192.168.1.1

# Habilitar IP forwarding
sudo sysctl -w net.ipv4.ip_forward=1
```

### DNS

```bash
# Testar DNS
nslookup www.google.com
dig www.google.com
host www.google.com

# Ver configuração DNS
cat /etc/resolv.conf

# Ver hosts locais
cat /etc/hosts
```

### Network Namespaces

```bash
# Criar namespace
sudo ip netns add red

# Listar namespaces
ip netns list

# Executar comando em namespace
sudo ip netns exec red ip link

# Criar veth pair
sudo ip link add veth-r type veth peer name veth-b

# Mover interface para namespace
sudo ip link set veth-r netns red

# Deletar namespace
sudo ip netns del red
```

### Docker

```bash
# Ver redes
docker network ls

# Inspecionar rede
docker network inspect bridge

# Criar container com port mapping
docker run -d -p 8080:80 nginx

# Container com rede do host
docker run -d --network host nginx
```

### Kubernetes

```bash
# Ver pods do CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Ver configuração DNS de um pod
kubectl exec -it <pod> -- cat /etc/resolv.conf

# Testar DNS de dentro do pod
kubectl exec -it <pod> -- nslookup kubernetes.default
```

---

## 💡 Dicas para o Exame CKA

1. **Entenda switching vs routing**
   - Switch: mesma rede (Camada 2, MAC)
   - Router: redes diferentes (Camada 3, IP)

2. **Default gateway é crucial**
   - Necessário para acessar redes externas
   - Verificar com: `ip route | grep default`

3. **DNS no Kubernetes**
   - CoreDNS resolve Services automaticamente
   - Formato: `<service>.<namespace>.svc.cluster.local`
   - Troubleshooting: `kubectl logs -n kube-system <coredns-pod>`

4. **Network Namespaces são a base de containers**
   - Cada Pod = 1 network namespace
   - veth pairs conectam namespaces
   - Linux bridge = switch virtual

5. **Comandos essenciais**
   - `ip link`: ver interfaces
   - `ip addr`: ver IPs
   - `ip route`: ver rotas
   - `nslookup`/`dig`: testar DNS

6. **Troubleshooting de rede**
   - `ping`: testar conectividade básica
   - `nslookup`: testar resolução DNS
   - `ip route`: verificar rotas
   - `kubectl exec -it <pod> -- sh`: entrar no pod para debug

---

⬅️ **Anterior**: [services.md](./services.md) | ➡️ **Próximo**: [network-policies.md](./network-policies.md)
