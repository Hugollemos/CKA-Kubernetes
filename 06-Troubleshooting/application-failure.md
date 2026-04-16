# Troubleshooting — Falha de Aplicação

## Abordagem

Siga o fluxo de cima para baixo: do usuário → service → pod → logs.

---

## 1. Verificar o acesso externo

```bash
# Testar acesso ao serviço via NodePort
curl http://<node-ip>:<node-port>

# Testar acesso interno ao ClusterIP
kubectl run -it test --image=busybox --rm --restart=Never -- \
  wget -qO- http://<service-name>:<port>
```

---

## 2. Verificar o Service

```bash
# Ver os endpoints do service — devem apontar para os Pods corretos
kubectl describe service web-service

# Verificar se os seletores do Service batem com as labels dos Pods
kubectl get service web-service -o yaml | grep selector
kubectl get pods --show-labels
```

Erros comuns:
- Labels dos Pods não batem com o `selector` do Service
- Porta errada (`port`, `targetPort`, `nodePort`)

---

## 3. Verificar o Pod

```bash
# Ver status do pod
kubectl get pods

# Ver eventos e detalhes
kubectl describe pod web

# Logs do pod
kubectl logs web

# Logs em tempo real
kubectl logs web -f

# Logs do pod anterior (se o pod reiniciou)
kubectl logs web -f --previous
```

Erros comuns:
- Pod em `CrashLoopBackOff` → ver logs do pod
- Pod em `Pending` → ver eventos (describe)
- Pod em `Error` → ver logs

---

## 4. Verificar o Pod do banco de dados (ou dependências)

```bash
# Logs do banco
kubectl logs db-pod

# Verificar se o service do banco resolve corretamente
kubectl exec -it web-pod -- nslookup db-service
kubectl exec -it web-pod -- wget -qO- http://db-service:5432
```

---

## 5. Verificar ConfigMaps e Secrets

```bash
# Verificar variáveis de ambiente do Pod
kubectl exec -it web-pod -- env | grep DB

# Ver configmap
kubectl get configmap app-config -o yaml

# Ver secret (em base64)
kubectl get secret app-secret -o yaml
```

---

## Referência Rápida

```bash
# Status dos pods
kubectl get pods -A

# Descrever pod (ver eventos de erro)
kubectl describe pod <nome>

# Logs do pod (container atual)
kubectl logs <pod> [-c <container>]

# Logs do container anterior (após crash)
kubectl logs <pod> --previous

# Acessar o container
kubectl exec -it <pod> -- /bin/sh

# Ver endpoints do service
kubectl get endpoints <service>
kubectl describe service <service>
```

---

## Referências

- https://kubernetes.io/docs/tasks/debug/debug-application/
