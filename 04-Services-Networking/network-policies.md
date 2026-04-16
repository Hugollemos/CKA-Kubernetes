# Network Policies

## O que são Network Policies?

Por padrão, o Kubernetes permite **todo tráfego** entre Pods — qualquer Pod pode falar com qualquer outro. **Network Policies** permitem restringir esse tráfego.

Tipos de tráfego:
- **Ingress** — tráfego que **entra** no Pod
- **Egress** — tráfego que **sai** do Pod

---

## Pré-requisito: Plugin de Rede compatível

Nem todos os plugins CNI suportam Network Policies:

| Plugin | Suporta NetworkPolicy |
|--------|----------------------|
| Calico | Sim |
| Weave Net | Sim |
| Cilium | Sim |
| Flannel | **Não** |

---

## Exemplo de cenário

```
Usuário → [Web Pod :80] → [API Pod :5000] → [DB Pod :3306]
```

Objetivo: Apenas o API Pod deve conseguir falar com o DB Pod na porta 3306.

---

## Criar uma Network Policy

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: db-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      role: db             # aplica-se aos Pods com esta label
  policyTypes:
  - Ingress                # restringir tráfego de entrada
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: api-pod    # permitir apenas Pods com esta label
    ports:
    - protocol: TCP
      port: 3306
```

```bash
kubectl create -f policy-definition.yaml

# Listar Network Policies
kubectl get networkpolicies
kubectl get netpol

# Descrever
kubectl describe networkpolicy db-policy
```

---

## Selectors disponíveis em `from` / `to`

### `podSelector` — Pods específicos

```yaml
from:
- podSelector:
    matchLabels:
      role: api-pod
```

### `namespaceSelector` — Pods de um namespace específico

```yaml
from:
- namespaceSelector:
    matchLabels:
      name: production
```

### `ipBlock` — Blocos de IP externos

```yaml
from:
- ipBlock:
    cidr: 192.168.1.0/24
    except:
    - 192.168.1.100/32
```

### Combinando selectors

```yaml
# AND — Pod deve ter ambas as condições (mesmo item da lista)
from:
- podSelector:
    matchLabels:
      role: api-pod
  namespaceSelector:
    matchLabels:
      name: production

# OR — qualquer uma das condições (itens separados na lista)
from:
- podSelector:
    matchLabels:
      role: api-pod
- namespaceSelector:
    matchLabels:
      name: production
```

---

## Política de Egress (saída)

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: api-policy
spec:
  podSelector:
    matchLabels:
      role: api-pod
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          role: web-pod
    ports:
    - protocol: TCP
      port: 5000
  egress:
  - to:
    - podSelector:
        matchLabels:
          role: db
    ports:
    - protocol: TCP
      port: 3306
```

---

## Política "Deny All" por padrão

```yaml
# Negar todo ingress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
spec:
  podSelector: {}           # aplica a todos os Pods
  policyTypes:
  - Ingress
  # sem regras ingress = negar tudo
```

```yaml
# Negar todo egress
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-egress
spec:
  podSelector: {}
  policyTypes:
  - Egress
```

```yaml
# Permitir todo ingress (restaurar padrão)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-all-ingress
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  ingress:
  - {}                      # regra vazia = permitir tudo
```

---

## Importante

- Network Policies são **aditivas** — múltiplas policies para o mesmo Pod se somam (OR lógico entre elas)
- Uma Policy com `policyTypes: [Ingress]` mas sem regras `ingress` **bloqueia todo ingress**
- Pods **sem nenhuma NetworkPolicy** aplicada permitem **todo tráfego** (ingress e egress)

---

## Referências

- https://kubernetes.io/docs/concepts/services-networking/network-policies/
- https://kubernetes.io/docs/tasks/administer-cluster/declare-network-policy/
