# Componentes dos Worker Nodes

Esta pasta contém guias completos sobre os componentes que rodam em cada nó de trabalho (worker node) do cluster Kubernetes.

## 📚 Conteúdo

### [kubelet.md](./kubelet.md)
**Agente que roda em cada nó do cluster**
- Responsabilidades do Kubelet
- Ciclo de vida de um pod no Kubelet
- Health checks: liveness, readiness, startup probes
- Gerenciamento de recursos (requests, limits, QoS classes)
- Gerenciamento de volumes
- Static Pods
- Container Runtime Interface (CRI)
- Troubleshooting de problemas de pods

### [kube-proxy.md](./kube-proxy.md)
**Gerenciador de regras de rede**
- Responsabilidades do Kube Proxy
- Modos de funcionamento: userspace, iptables, IPVS
- Como funciona o roteamento de tráfego para Services
- Service Discovery (DNS e variáveis de ambiente)
- Tipos de Services: ClusterIP, NodePort, LoadBalancer, ExternalName
- Network Policies
- Troubleshooting de conectividade

## 🎯 Importância para o Exame CKA

Estes componentes são fundamentais para:
- **Troubleshooting (30%)**: Debugar pods que não iniciam, problemas de conectividade
- **Workloads & Scheduling (15%)**: Entender health checks e recursos
- **Services & Networking (20%)**: Compreender como Services funcionam

## 🔗 Ordem de Estudo Sugerida

1. **[kubelet.md](./kubelet.md)** - Comece pelo agente que gerencia os pods
2. **[kube-proxy.md](./kube-proxy.md)** - Depois entenda networking e services

## 💡 Conceitos-Chave

### Kubelet
- Registra o nó no cluster
- Garante que containers estejam rodando
- Executa health checks
- Reporta status ao API Server
- Gerencia volumes e secrets

### Kube Proxy
- Mantém regras de rede (iptables/IPVS)
- Implementa o conceito de Service
- Faz load balancing entre pods
- Permite comunicação entre pods de diferentes nós

## 🛠️ Comandos Úteis

```bash
# Ver status do kubelet
systemctl status kubelet
journalctl -u kubelet -f

# Ver logs do kube-proxy
kubectl logs -n kube-system kube-proxy-xxxxx

# Ver regras de iptables criadas pelo kube-proxy
sudo iptables -t nat -L -n -v | grep <service-name>

# Debugar pod
kubectl describe pod <pod-name>
kubectl logs <pod-name>
kubectl exec -it <pod-name> -- /bin/bash
```

## 🚨 Troubleshooting Comum

### Problemas do Kubelet
- Node NotReady → verificar kubelet status
- Pod Pending → verificar recursos do nó
- ImagePullBackOff → problema para baixar imagem
- CrashLoopBackOff → container está falhando

### Problemas do Kube Proxy
- Não consegue acessar Service → verificar endpoints
- DNS não resolve → verificar CoreDNS
- NodePort não funciona → verificar regras de firewall

---

⬅️ **Anterior**: [Componentes-Control-Plane](../Componentes-Control-Plane/) | ➡️ **Próximo**: [Workloads](../Workloads/)
