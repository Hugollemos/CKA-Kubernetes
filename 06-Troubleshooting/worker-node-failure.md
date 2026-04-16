# Troubleshooting — Falha de Worker Node

## 1. Verificar Status dos Nós

```bash
# Ver todos os nós e seu status
kubectl get nodes

# Nó com problema estará NotReady
# NAME       STATUS     ROLES    AGE   VERSION
# node01     NotReady   <none>   10d   v1.28.0
```

---

## 2. Descrever o Nó com Problema

```bash
kubectl describe node worker-1
```

Observar:
- **Conditions** — mostra o estado do nó (Ready, DiskPressure, MemoryPressure, PIDPressure)
- **LastHeartbeatTime** — quando o nó enviou o último sinal de vida
- **Events** — eventos recentes com possíveis erros

```
Conditions:
  Type             Status    LastHeartbeatTime
  ----             ------    -----------------
  MemoryPressure   False     ...
  DiskPressure     False     ...
  PIDPressure      False     ...
  Ready            False     ← problema aqui!
```

---

## 3. Verificar Recursos do Nó

Acesse o nó via SSH e verifique:

```bash
# CPU e memória
top
free -h

# Espaço em disco
df -h

# Uso de disco por pasta
du -sh /var/lib/kubelet/
du -sh /var/log/
```

---

## 4. Verificar o kubelet

O kubelet é o agente que roda em cada nó. Se ele falhar, o nó fica NotReady.

```bash
# Ver status do kubelet
service kubelet status
systemctl status kubelet

# Ver logs do kubelet
sudo journalctl -u kubelet -f
sudo journalctl -u kubelet --since "10 minutes ago"
```

### Reiniciar o kubelet

```bash
sudo systemctl restart kubelet
sudo systemctl enable kubelet
```

---

## 5. Verificar Certificados do kubelet

```bash
# Verificar certificado do kubelet
openssl x509 -in /var/lib/kubelet/worker-1.crt -text -noout

# Verificar:
# - Not After (data de expiração)
# - Issuer (deve ser assinado pela CA correta)
# - Subject: deve ter o nome do nó
```

---

## 6. Verificar Configuração do kubelet

```bash
# Arquivo de configuração
cat /var/lib/kubelet/config.yaml

# Arquivo de serviço systemd
cat /etc/systemd/system/kubelet.service.d/10-kubeadm.conf
# ou
cat /lib/systemd/system/kubelet.service

# Verificar se o apiserver está acessível do nó
curl -k https://<apiserver-ip>:6443/healthz
```

---

## Problemas Comuns por Condition

| Condition | Status True | Causa | Solução |
|-----------|-------------|-------|---------|
| `MemoryPressure` | True | Memória insuficiente | Liberar memória ou aumentar nó |
| `DiskPressure` | True | Disco cheio | Limpar `/var/log`, imagens antigas |
| `PIDPressure` | True | Muitos processos | Matar processos desnecessários |
| `Ready` | False | kubelet parado | Reiniciar kubelet |
| `Ready` | Unknown | Nó isolado da rede | Verificar conectividade |

---

## Fluxo de Troubleshooting Resumido

```bash
# 1. Ver o nó com problema
kubectl get nodes
kubectl describe node <nome-do-no>

# 2. Acessar o nó (SSH)
ssh usuario@<ip-do-no>

# 3. Checar recursos
top && df -h && free -h

# 4. Checar kubelet
systemctl status kubelet
journalctl -u kubelet -n 50

# 5. Reiniciar se necessário
systemctl restart kubelet

# 6. Verificar certificados se kubelet falha ao conectar
openssl x509 -in /var/lib/kubelet/pki/kubelet-client-current.pem -text -noout
```

---

## Referências

- https://kubernetes.io/docs/tasks/debug/debug-cluster/
- https://kubernetes.io/docs/setup/production-environment/tools/kubeadm/troubleshooting-kubeadm/
