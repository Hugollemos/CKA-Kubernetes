# DNS no Kubernetes e CoreDNS

## DNS no Kubernetes

O Kubernetes possui um servidor DNS interno que resolve nomes de Pods e Services automaticamente.

---

## Resolução de DNS para Services

### Formato

```
<nome-service>.<namespace>.svc.cluster.local
```

### Exemplos

```bash
# Service no namespace default
web-service.default.svc.cluster.local

# Service no namespace apps
nginx-service.apps.svc.cluster.local
```

Do mesmo namespace, você pode usar apenas o nome simples:
```bash
# Dentro do namespace default → funciona diretamente
curl web-service

# De outro namespace → precisa do nome completo
curl web-service.default.svc.cluster.local
```

---

## Resolução de DNS para Pods

Os Pods têm registros DNS baseados em seus IPs (hífens no lugar de pontos):

```
<IP-com-hifens>.<namespace>.pod.cluster.local
```

### Exemplo

```bash
# Pod com IP 10.244.1.10 no namespace default
10-244-1-10.default.pod.cluster.local
```

### Testando na prática

```bash
# Criar pod no namespace apps
kubectl run nginx --image=nginx --namespace apps

# Ver o IP do pod
kubectl get po -n apps -o wide
# IP: 10.244.1.3

# Resolver o DNS do pod (de outro pod)
kubectl run -it test --image=busybox:1.28 --rm --restart=Never -- \
  nslookup 10-244-1-3.apps.pod.cluster.local

# Acessar via curl
kubectl run -it nginx-test --image=nginx --rm --restart=Never -- \
  curl -Is http://10-244-1-3.apps.pod.cluster.local
```

---

## CoreDNS

O **CoreDNS** é o servidor DNS padrão do Kubernetes desde a versão 1.13. Ele roda como pods no namespace `kube-system`.

### Verificar os pods do CoreDNS

```bash
kubectl get pods -n kube-system
# NAME                         READY   STATUS
# coredns-66bff467f8-2vghh     1/1     Running
# coredns-66bff467f8-t5nzm     1/1     Running
```

### Verificar o Deployment

```bash
kubectl get deployment -n kube-system
# NAME      READY   UP-TO-DATE   AVAILABLE
# coredns   2/2     2            2
```

### Verificar o Service do CoreDNS

```bash
kubectl get service -n kube-system
# NAME       TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)
# kube-dns   ClusterIP   10.96.0.10   <none>        53/UDP,53/TCP,9153/TCP
```

---

## Arquivo de Configuração (Corefile)

```bash
kubectl describe cm coredns -n kube-system
```

```
.:53 {
    errors
    health {
       lameduck 5s
    }
    ready
    kubernetes cluster.local in-addr.arpa ip6.arpa {
       pods insecure
       fallthrough in-addr.arpa ip6.arpa
       ttl 30
    }
    prometheus :9153
    forward . /etc/resolv.conf    # encaminha DNS externo para o resolv.conf do nó
    cache 30
    loop
    reload
}
```

---

## Como o kubelet configura o DNS dos Pods

O kubelet configura cada Pod para usar o CoreDNS como servidor DNS:

```bash
cat /var/lib/kubelet/config.yaml | grep -A2 clusterDNS
# clusterDNS:
# - 10.96.0.10            ← IP do serviço kube-dns
# clusterDomain: cluster.local
```

O `/etc/resolv.conf` dentro de cada Pod é configurado automaticamente:

```bash
kubectl run -it --rm --restart=Never test-pod --image=busybox -- cat /etc/resolv.conf
# nameserver 10.96.0.10
# search default.svc.cluster.local svc.cluster.local cluster.local
# options ndots:5
```

---

## FQDN na Prática

Graças ao `search` no `resolv.conf`, você pode usar nomes parciais:

```bash
# Todos resolvem para o mesmo endereço dentro do mesmo namespace
host web-service
# web-service.default.svc.cluster.local has address 10.106.112.101

host web-service.default
# web-service.default.svc.cluster.local has address 10.106.112.101

host web-service.default.svc
# web-service.default.svc.cluster.local has address 10.106.112.101

host web-service.default.svc.cluster.local
# web-service.default.svc.cluster.local has address 10.106.112.101
```

---

## Resolver Pods e Services via exec

```bash
# Resolver um service
kubectl exec -it meu-pod -- nslookup web-service.default.svc.cluster.local

# Resolver um pod pelo IP
kubectl exec -it meu-pod -- nslookup 10-244-1-4.default.pod.cluster.local
```

---

## Troubleshooting de DNS

```bash
# Verificar se CoreDNS está rodando
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Ver logs do CoreDNS
kubectl logs -n kube-system -l k8s-app=kube-dns

# Testar resolução de dentro de um pod temporário
kubectl run dns-test --image=busybox:1.28 --rm -it --restart=Never -- nslookup kubernetes

# Verificar configmap
kubectl get cm coredns -n kube-system -o yaml
```

---

## Referências

- https://kubernetes.io/docs/concepts/services-networking/dns-pod-service/
- https://kubernetes.io/docs/tasks/administer-cluster/dns-debugging-resolution/
- https://coredns.io/
