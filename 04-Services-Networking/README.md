# Networking (Redes)

Esta pasta contém guias sobre networking no Kubernetes, incluindo Services e conectividade entre pods.

## 📚 Conteúdo

### [dns-coredns.md](./dns-coredns.md)
**DNS interno do cluster e CoreDNS**
- Resolução de DNS para Services (`<nome>.<namespace>.svc.cluster.local`)
- Resolução de DNS para Pods (IP com hífens)
- CoreDNS: pods, deployment e service no kube-system
- Arquivo de configuração Corefile
- Como o kubelet configura o DNS dos pods
- Troubleshooting de DNS

### [network-policies.md](./network-policies.md)
**Controle de tráfego entre Pods**
- O que são Network Policies e plugins compatíveis (Calico, Weave, Cilium)
- Tipos de tráfego: Ingress e Egress
- Selectors: podSelector, namespaceSelector, ipBlock
- Combinando selectors (AND vs OR)
- Política "Deny All" padrão
- Regras aditivas entre políticas

### [services.md](./services.md)
**Abstração de rede para expor aplicações**
- O que são Services e por que são necessários
- Tipos de Services:
  - **ClusterIP**: acesso interno no cluster (padrão)
  - **NodePort**: expõe em uma porta de cada nó
  - **LoadBalancer**: cria load balancer externo (cloud)
  - **ExternalName**: alias DNS para serviços externos
- Service Discovery (DNS e variáveis de ambiente)
- Endpoints: mapeamento para pods
- Estratégias de load balancing
- Session Affinity (sticky sessions)
- externalTrafficPolicy (Local vs Cluster)
- Troubleshooting de conectividade

## 🎯 Importância para o Exame CKA

Services & Networking representa **20% da prova**.

Você precisa dominar:
- Criar Services de diferentes tipos
- Entender como Service Discovery funciona
- Debugar problemas de conectividade
- Compreender Endpoints
- Usar Network Policies (se aplicável)
- Trabalhar com DNS (CoreDNS)

## 💡 Conceitos-Chave

### Services

**ClusterIP** (interno)
```bash
kubectl expose deployment nginx --port=80 --type=ClusterIP
# Acesso: http://service-name.namespace.svc.cluster.local
```

**NodePort** (externo em porta do nó)
```bash
kubectl expose deployment nginx --port=80 --type=NodePort
# Acesso: http://NODE-IP:30000-32767
```

**LoadBalancer** (cloud)
```bash
kubectl expose deployment nginx --port=80 --type=LoadBalancer
# Acesso: http://EXTERNAL-IP (provisionado pelo cloud provider)
```

### Service Discovery

**DNS** (recomendado)
```bash
# Dentro do cluster:
curl http://nginx-service              # mesmo namespace
curl http://nginx-service.default      # especifica namespace
curl http://nginx-service.default.svc.cluster.local  # FQDN completo
```

**Environment Variables** (legado)
```bash
# Kubernetes injeta automaticamente:
NGINX_SERVICE_HOST=10.96.0.5
NGINX_SERVICE_PORT=80
```

### Endpoints

```bash
# Ver endpoints de um service
kubectl get endpoints nginx-service

# Endpoints são atualizados automaticamente
# quando pods são criados/destruídos
```

## 🛠️ Comandos Úteis

```bash
# Criar Service
kubectl expose deployment nginx --port=80 --type=ClusterIP

# Listar Services
kubectl get svc

# Descrever Service (mostra endpoints!)
kubectl describe svc nginx-service

# Ver Endpoints
kubectl get endpoints nginx-service

# Testar conectividade
kubectl run tmp --rm -i --tty --image=busybox -- wget -O- http://nginx-service

# Port forward (acesso local)
kubectl port-forward svc/nginx-service 8080:80
# Acessa via http://localhost:8080
```

## 🚨 Troubleshooting Comum

### Service não está acessível

**1. Verificar se Service existe**
```bash
kubectl get svc nginx-service
```

**2. Verificar Endpoints**
```bash
kubectl get endpoints nginx-service
# Se vazio: selector não match com nenhum Pod
```

**3. Verificar labels dos Pods**
```bash
kubectl get pods --show-labels
# Compare com selector do Service
```

**4. Testar dentro do cluster**
```bash
kubectl run tmp --rm -i --tty --image=busybox -- sh
# Dentro do container:
wget -O- http://nginx-service
```

**5. Verificar iptables (kube-proxy)**
```bash
sudo iptables -t nat -L -n | grep nginx-service
```

**6. Verificar DNS**
```bash
kubectl run tmp --rm -i --tty --image=busybox -- nslookup nginx-service
```

### NodePort não funciona

**1. Verificar porta**
```bash
kubectl get svc nginx-service
# Procure por nodePort na saída (30000-32767)
```

**2. Testar de dentro primeiro**
```bash
kubectl run tmp --rm -i --tty --image=busybox -- wget -O- http://nginx-service
```

**3. Verificar firewall**
```bash
# Porta nodePort deve estar aberta no firewall
```

**4. Testar via IP do nó**
```bash
curl http://NODE-IP:30080
```

### LoadBalancer fica <pending>

**1. Aguardar provisionamento**
```bash
kubectl get svc nginx-service -w
# Pode levar alguns minutos
```

**2. Verificar eventos**
```bash
kubectl describe svc nginx-service
# Procure por mensagens de erro
```

**3. Verificar cloud provider**
```bash
# Cloud controller pode não estar configurado
# Verificar credenciais e permissões
```

## 📊 Fluxo de Tráfego

```
Cliente
  ↓
Service (ClusterIP virtual)
  ↓
Kube-Proxy (regras iptables/IPVS)
  ↓
Endpoints (IPs dos Pods)
  ↓
Pod (container rodando)
```

## 🔗 Relação com Outros Componentes

- **Kube-Proxy**: implementa as regras de rede para Services
- **CoreDNS**: resolve nomes de Services para IPs
- **Endpoints Controller**: mantém lista de IPs dos pods
- **API Server**: armazena definições de Services

---

⬅️ **Anterior**: [03-Workloads-Scheduling](../03-Workloads-Scheduling/) | ➡️ **Próximo**: [05-Storage](../05-Storage/)
