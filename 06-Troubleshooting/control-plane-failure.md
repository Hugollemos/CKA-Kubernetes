# Troubleshooting — Falha do Control Plane

## Verificar Estado Geral dos Nós

```bash
kubectl get nodes
```

---

## Verificar Pods do Control Plane (kubeadm)

```bash
# Ver todos os pods do control plane no namespace kube-system
kubectl get pods -n kube-system

# Todos devem estar Running
# coredns-xxx            1/1   Running
# etcd-master            1/1   Running
# kube-apiserver-master  1/1   Running
# kube-controller-master 1/1   Running
# kube-scheduler-master  1/1   Running
```

---

## Verificar Serviços (instalação sem kubeadm)

Se o control plane foi instalado como serviços systemd:

```bash
# Status de cada componente
service kube-apiserver status
service kube-controller-manager status
service kube-scheduler status

# Em nós worker
service kubelet status
service kube-proxy status
```

---

## Ver Logs dos Componentes

### Componentes como Pods (kubeadm)

```bash
kubectl logs kube-apiserver-master -n kube-system
kubectl logs kube-controller-manager-master -n kube-system
kubectl logs kube-scheduler-master -n kube-system
kubectl logs etcd-master -n kube-system
```

### Componentes como serviços systemd

```bash
sudo journalctl -u kube-apiserver -f
sudo journalctl -u kube-controller-manager -f
sudo journalctl -u kube-scheduler -f
sudo journalctl -u etcd -f
```

### Se kubectl não funcionar (apiserver down)

```bash
# Acessar o container diretamente
crictl ps -a
crictl logs <container-id>

# Ou com docker
docker ps -a
docker logs <container-id>
```

---

## Verificações Comuns por Componente

### kube-apiserver

```bash
# Verificar o manifesto (kubeadm)
cat /etc/kubernetes/manifests/kube-apiserver.yaml

# Verificar certificados
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout | grep -A2 Validity
```

### etcd

```bash
# Verificar saúde do etcd
ETCDCTL_API=3 etcdctl endpoint health \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/healthcheck-client.crt \
  --key=/etc/kubernetes/pki/etcd/healthcheck-client.key

# Manifesto do etcd
cat /etc/kubernetes/manifests/etcd.yaml
```

### kube-controller-manager e kube-scheduler

```bash
# Manifestos
cat /etc/kubernetes/manifests/kube-controller-manager.yaml
cat /etc/kubernetes/manifests/kube-scheduler.yaml
```

---

## Problemas Comuns

| Sintoma | Causa provável | Como verificar |
|---------|----------------|----------------|
| kubectl não responde | apiserver down | `crictl logs` do apiserver |
| Pods não são agendados | scheduler down | `kubectl logs kube-scheduler` |
| Deployments não criam Pods | controller-manager down | `kubectl logs kube-controller-manager` |
| Estado do cluster perdido | etcd corrompido | `etcdctl endpoint health` |
| Certificado expirado | TLS inválido | `openssl x509 ... -text -noout` |

---

## Referências

- https://kubernetes.io/docs/tasks/debug/debug-cluster/
