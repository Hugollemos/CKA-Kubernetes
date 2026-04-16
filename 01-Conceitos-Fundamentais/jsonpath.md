# JSONPath no Kubernetes

Este documento cobre **JSONPath** no contexto do Kubernetes, uma ferramenta essencial para extrair dados específicos de recursos usando `kubectl`.

## 📚 Conteúdo

1. [O que é JSONPath?](#o-que-é-jsonpath)
2. [Sintaxe Básica](#sintaxe-básica)
3. [Usando JSONPath com kubectl](#usando-jsonpath-com-kubectl)
4. [Seletores e Filtros](#seletores-e-filtros)
5. [Exemplos Práticos](#exemplos-práticos)
6. [Custom Columns](#custom-columns)
7. [Troubleshooting](#troubleshooting)

---

## 🔍 O que é JSONPath?

**JSONPath** é uma linguagem de consulta para JSON, similar ao XPath para XML. No Kubernetes, usamos JSONPath para **extrair dados específicos** de recursos ao invés de ver todo o YAML/JSON.

### Por que usar JSONPath?

**Sem JSONPath:**
```bash
kubectl get pod nginx -o yaml
# Output: 100+ linhas de YAML
# Difícil encontrar o que você precisa
```

**Com JSONPath:**
```bash
kubectl get pod nginx -o jsonpath='{.status.podIP}'
# Output: 10.244.1.5
# Apenas o que você precisa!
```

### Casos de Uso

- ✅ Extrair informações específicas (IPs, imagens, status)
- ✅ Listar recursos em formato customizado
- ✅ Automatizar scripts (obter valores para usar em comandos)
- ✅ Debugging rápido (verificar campos específicos)
- ✅ No exame CKA (economizar tempo!)

---

## 📖 Sintaxe Básica

### Estrutura JSONPath

JSONPath usa **dot notation** para navegar pelo JSON:

```
$ - Raiz do documento
. - Operador de acesso
[] - Subscript (acesso a arrays ou filtros)
* - Wildcard (todos os elementos)
```

### Exemplo de JSON

```json
{
  "apiVersion": "v1",
  "kind": "Pod",
  "metadata": {
    "name": "nginx",
    "namespace": "default",
    "labels": {
      "app": "web"
    }
  },
  "spec": {
    "containers": [
      {
        "name": "nginx",
        "image": "nginx:1.21"
      }
    ]
  },
  "status": {
    "phase": "Running",
    "podIP": "10.244.1.5"
  }
}
```

### Acessando Campos

| JSONPath | Resultado | Descrição |
|----------|-----------|-----------|
| `{.kind}` | `Pod` | Campo na raiz |
| `{.metadata.name}` | `nginx` | Campo aninhado |
| `{.metadata.labels.app}` | `web` | Campo em objeto aninhado |
| `{.spec.containers[0].name}` | `nginx` | Primeiro elemento do array |
| `{.spec.containers[0].image}` | `nginx:1.21` | Campo de elemento do array |
| `{.status.podIP}` | `10.244.1.5` | Campo em status |

### Sintaxe Importante

**1. Root `$` é implícito**
```bash
# No Kubernetes, $ é implícito, use apenas .
{.metadata.name}   # ✅ Correto
{$.metadata.name}  # ❌ Erro ($ é implícito)
```

**2. Sempre use aspas simples externas e chaves `{}`**
```bash
kubectl get pod nginx -o jsonpath='{.metadata.name}'
#                                  ^              ^
#                                  Aspas simples + chaves
```

**3. Arrays usam `[índice]` ou `[*]`**
```bash
{.spec.containers[0]}      # Primeiro container
{.spec.containers[1]}      # Segundo container
{.spec.containers[*]}      # Todos os containers
```

---

## ⚙️ Usando JSONPath com kubectl

### Opção `-o jsonpath`

```bash
kubectl get <resource> <name> -o jsonpath='<jsonpath-expression>'
```

### Ver JSON completo (para entender estrutura)

```bash
# Ver como JSON
kubectl get pod nginx -o json

# Ver como YAML (mais legível)
kubectl get pod nginx -o yaml

# Ver apenas spec
kubectl get pod nginx -o json | jq .spec

# Ver apenas status
kubectl get pod nginx -o json | jq .status
```

### Exemplos Básicos

```bash
# Nome do Pod
kubectl get pod nginx -o jsonpath='{.metadata.name}'
# Output: nginx

# Namespace do Pod
kubectl get pod nginx -o jsonpath='{.metadata.namespace}'
# Output: default

# IP do Pod
kubectl get pod nginx -o jsonpath='{.status.podIP}'
# Output: 10.244.1.5

# Imagem do container
kubectl get pod nginx -o jsonpath='{.spec.containers[0].image}'
# Output: nginx:1.21

# Status do Pod
kubectl get pod nginx -o jsonpath='{.status.phase}'
# Output: Running
```

### Adicionar newline ao final

Por padrão, JSONPath não adiciona `\n` ao final:

```bash
# Sem newline (ruim para scripts)
kubectl get pod nginx -o jsonpath='{.status.podIP}'
# Output: 10.244.1.5[user@host ~]$  (sem quebra de linha)

# Com newline (melhor)
kubectl get pod nginx -o jsonpath='{.status.podIP}{"\n"}'
# Output: 10.244.1.5
# [user@host ~]$
```

---

## 🎯 Seletores e Filtros

### Arrays - Todos os elementos `[*]`

```bash
# Nomes de todos os containers em um Pod
kubectl get pod nginx -o jsonpath='{.spec.containers[*].name}'
# Output: nginx sidecar

# Imagens de todos os containers
kubectl get pod nginx -o jsonpath='{.spec.containers[*].image}'
# Output: nginx:1.21 busybox:1.35

# Listar com quebra de linha entre itens
kubectl get pod nginx -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
# Output:
# nginx
# sidecar
```

### Lista de recursos - `items[*]`

Quando você faz `kubectl get pods` (plural), o resultado é um **array** em `items`:

```json
{
  "apiVersion": "v1",
  "kind": "List",
  "items": [
    { "metadata": { "name": "pod1" } },
    { "metadata": { "name": "pod2" } },
    { "metadata": { "name": "pod3" } }
  ]
}
```

```bash
# Nomes de todos os Pods
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
# Output: pod1 pod2 pod3

# IPs de todos os Pods
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
# Output: 10.244.1.5 10.244.1.6 10.244.1.7

# Com newline entre cada
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
# Output:
# pod1
# pod2
# pod3
```

### Range Loop `{range ...}{end}`

Para formatar melhor listas:

```bash
# Básico: Nome e IP de cada Pod
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
# Output:
# pod1    10.244.1.5
# pod2    10.244.1.6
# pod3    10.244.1.7

# Com labels
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.labels.app}{"\n"}{end}'
# Output:
# pod1    web
# pod2    api
# pod3    db
```

### Filtros `[?(...)]`

Filtrar elementos baseado em condições:

```bash
# Containers com porta 80
kubectl get pod nginx -o jsonpath='{.spec.containers[?(@.ports[0].containerPort==80)].name}'

# Pods com status Running
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'

# Pods com label app=web
kubectl get pods -o jsonpath='{.items[?(@.metadata.labels.app=="web")].metadata.name}'
```

**Sintaxe de filtro:**
- `@` - Elemento atual
- `==` - Igual
- `!=` - Diferente
- `<`, `>`, `<=`, `>=` - Comparações

---

## 💡 Exemplos Práticos

### 1. Informações de Pods

```bash
# Nome do Pod
kubectl get pod nginx -o jsonpath='{.metadata.name}{"\n"}'

# IP do Pod
kubectl get pod nginx -o jsonpath='{.status.podIP}{"\n"}'

# Node onde Pod está rodando
kubectl get pod nginx -o jsonpath='{.spec.nodeName}{"\n"}'

# Namespace do Pod
kubectl get pod nginx -o jsonpath='{.metadata.namespace}{"\n"}'

# Service Account
kubectl get pod nginx -o jsonpath='{.spec.serviceAccountName}{"\n"}'

# Imagens de todos os containers
kubectl get pod nginx -o jsonpath='{.spec.containers[*].image}{"\n"}'

# Nome de todos os containers
kubectl get pod nginx -o jsonpath='{.spec.containers[*].name}{"\n"}'
```

### 2. Listar múltiplos Pods

```bash
# Nomes de todos os Pods
kubectl get pods -o jsonpath='{.items[*].metadata.name}{"\n"}'

# IPs de todos os Pods
kubectl get pods -o jsonpath='{.items[*].status.podIP}{"\n"}'

# Nodes de todos os Pods
kubectl get pods -o jsonpath='{.items[*].spec.nodeName}{"\n"}'

# Nome + IP de cada Pod (formatado)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
# Output:
# nginx       10.244.1.5
# apache      10.244.1.6
# redis       10.244.1.7

# Nome + Node + IP (3 colunas)
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\t"}{.status.podIP}{"\n"}{end}'
```

### 3. Nodes

```bash
# Nomes de todos os Nodes
kubectl get nodes -o jsonpath='{.items[*].metadata.name}{"\n"}'

# IPs internos dos Nodes
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}{"\n"}'

# Versão do kubelet em cada Node
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.kubeletVersion}{"\n"}{end}'

# CPU capacity de cada Node
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.cpu}{"\n"}{end}'

# Memory capacity
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.capacity.memory}{"\n"}{end}'

# OS de cada Node
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.nodeInfo.osImage}{"\n"}{end}'
```

### 4. Services

```bash
# ClusterIP de um Service
kubectl get svc nginx -o jsonpath='{.spec.clusterIP}{"\n"}'

# Portas de um Service
kubectl get svc nginx -o jsonpath='{.spec.ports[*].port}{"\n"}'

# Tipo de Service (ClusterIP, NodePort, LoadBalancer)
kubectl get svc nginx -o jsonpath='{.spec.type}{"\n"}'

# Selector do Service
kubectl get svc nginx -o jsonpath='{.spec.selector}{"\n"}'

# Para LoadBalancer: IP externo
kubectl get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'

# Listar todos Services e seus ClusterIPs
kubectl get svc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.clusterIP}{"\n"}{end}'
```

### 5. Deployments

```bash
# Número de réplicas desejadas
kubectl get deploy nginx -o jsonpath='{.spec.replicas}{"\n"}'

# Número de réplicas disponíveis
kubectl get deploy nginx -o jsonpath='{.status.availableReplicas}{"\n"}'

# Imagem usada no Deployment
kubectl get deploy nginx -o jsonpath='{.spec.template.spec.containers[0].image}{"\n"}'

# Strategy de update
kubectl get deploy nginx -o jsonpath='{.spec.strategy.type}{"\n"}'

# Listar Deployments e suas imagens
kubectl get deploy -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.template.spec.containers[0].image}{"\n"}{end}'
```

### 6. ConfigMaps e Secrets

```bash
# Keys em um ConfigMap
kubectl get cm my-config -o jsonpath='{.data}' | jq 'keys'

# Valor de uma key específica
kubectl get cm my-config -o jsonpath='{.data.app\.properties}{"\n"}'

# Decodificar Secret (base64)
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 --decode
echo  # Adicionar newline

# Listar todas as keys de um Secret
kubectl get secret my-secret -o jsonpath='{.data}' | jq 'keys'
```

### 7. PersistentVolumes e PVCs

```bash
# Capacity de um PV
kubectl get pv my-pv -o jsonpath='{.spec.capacity.storage}{"\n"}'

# StorageClass de um PV
kubectl get pv my-pv -o jsonpath='{.spec.storageClassName}{"\n"}'

# Status de um PV (Available, Bound, Released)
kubectl get pv my-pv -o jsonpath='{.status.phase}{"\n"}'

# PV bound a um PVC
kubectl get pvc my-pvc -o jsonpath='{.spec.volumeName}{"\n"}'

# Storage request de um PVC
kubectl get pvc my-pvc -o jsonpath='{.spec.resources.requests.storage}{"\n"}'

# Listar todos PVs e seus status
kubectl get pv -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.phase}{"\t"}{.spec.capacity.storage}{"\n"}{end}'
```

### 8. Namespaces

```bash
# Listar todos namespaces
kubectl get ns -o jsonpath='{.items[*].metadata.name}{"\n"}'

# Status de um namespace
kubectl get ns default -o jsonpath='{.status.phase}{"\n"}'

# Loop por todos namespaces
for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
  echo "Namespace: $ns"
  kubectl get pods -n $ns
done
```

### 9. Ingress

```bash
# Host de um Ingress
kubectl get ingress my-ingress -o jsonpath='{.spec.rules[0].host}{"\n"}'

# IP/Address do Ingress
kubectl get ingress my-ingress -o jsonpath='{.status.loadBalancer.ingress[0].ip}{"\n"}'

# Paths de um Ingress
kubectl get ingress my-ingress -o jsonpath='{.spec.rules[0].http.paths[*].path}{"\n"}'

# Backends de um Ingress
kubectl get ingress my-ingress -o jsonpath='{.spec.rules[0].http.paths[*].backend.service.name}{"\n"}'

# TLS hosts
kubectl get ingress my-ingress -o jsonpath='{.spec.tls[*].hosts[*]}{"\n"}'
```

### 10. Events

```bash
# Mensagens de eventos de um Pod
kubectl get events --field-selector involvedObject.name=nginx -o jsonpath='{.items[*].message}{"\n"}'

# Últimos 5 eventos
kubectl get events --sort-by='.lastTimestamp' -o jsonpath='{range .items[-5:]}{.lastTimestamp}{"\t"}{.message}{"\n"}{end}'
```

---

## 📊 Custom Columns

**Custom Columns** combinam JSONPath com output formatado em tabela.

### Sintaxe

```bash
kubectl get <resource> -o custom-columns=<COLUMN_NAME>:<jsonpath>,<COLUMN_NAME>:<jsonpath>,...
```

### Exemplos

```bash
# Pods: Nome + IP
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP

# Output:
# NAME      IP
# nginx     10.244.1.5
# apache    10.244.1.6
# redis     10.244.1.7

# Pods: Nome + Node + IP
kubectl get pods -o custom-columns=NAME:.metadata.name,NODE:.spec.nodeName,IP:.status.podIP

# Nodes: Nome + CPU + Memory
kubectl get nodes -o custom-columns=NAME:.metadata.name,CPU:.status.capacity.cpu,MEMORY:.status.capacity.memory

# Deployments: Nome + Replicas + Image
kubectl get deploy -o custom-columns=NAME:.metadata.name,REPLICAS:.spec.replicas,IMAGE:.spec.template.spec.containers[0].image

# Services: Nome + Type + ClusterIP
kubectl get svc -o custom-columns=NAME:.metadata.name,TYPE:.spec.type,CLUSTER-IP:.spec.clusterIP

# PVCs: Nome + Status + Capacity + StorageClass
kubectl get pvc -o custom-columns=NAME:.metadata.name,STATUS:.status.phase,CAPACITY:.status.capacity.storage,STORAGECLASS:.spec.storageClassName
```

### Custom Columns com Todos os Namespaces

```bash
# Pods de todos namespaces com namespace na coluna
kubectl get pods -A -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName
```

### Salvar template de Custom Columns

```bash
# Criar arquivo com template
cat > pod-columns.txt <<EOF
NAME:.metadata.name
NAMESPACE:.metadata.namespace
IP:.status.podIP
NODE:.spec.nodeName
STATUS:.status.phase
EOF

# Usar template
kubectl get pods -o custom-columns-file=pod-columns.txt
```

---

## 🔧 Troubleshooting

### 1. JSONPath retorna vazio

**Problema:**
```bash
kubectl get pod nginx -o jsonpath='{.status.podIp}'
# Output: (vazio)
```

**Solução:** Verificar se campo existe e está correto (case-sensitive!)

```bash
# Ver JSON completo
kubectl get pod nginx -o json | grep -i podip
# Output: "podIP": "10.244.1.5"  (IP maiúsculo!)

# Corrigir
kubectl get pod nginx -o jsonpath='{.status.podIP}'
# Output: 10.244.1.5
```

### 2. Array está vazio ou null

**Problema:**
```bash
kubectl get pod nginx -o jsonpath='{.spec.containers[0].ports[0].containerPort}'
# Output: (vazio - container não tem porta definida)
```

**Solução:** Verificar se array tem elementos

```bash
# Ver spec completo
kubectl get pod nginx -o yaml | grep -A 10 containers

# Se não tem ports definidos, campo será vazio
```

### 3. Erro de sintaxe

**Problema:**
```bash
kubectl get pod nginx -o jsonpath="{.metadata.name}"
# Error: error executing jsonpath...
```

**Solução:** Usar **aspas simples** externas (não aspas duplas)

```bash
# ❌ Errado (aspas duplas)
kubectl get pod nginx -o jsonpath="{.metadata.name}"

# ✅ Correto (aspas simples)
kubectl get pod nginx -o jsonpath='{.metadata.name}'
```

### 4. Campo não existe

**Problema:**
```bash
kubectl get pod nginx -o jsonpath='{.status.hostIP}'
# Output: (vazio se campo não existe)
```

**Solução:** Ver estrutura JSON para confirmar caminho

```bash
# Ver JSON e procurar campo
kubectl get pod nginx -o json | jq .status | grep -i hostip
# Output: "hostIP": "192.168.1.10"

# Usar caminho correto
kubectl get pod nginx -o jsonpath='{.status.hostIP}'
```

### 5. Acessar campo com ponto no nome

**Problema:**
```bash
# ConfigMap com key "app.properties"
kubectl get cm my-config -o jsonpath='{.data.app.properties}'
# Output: (errado - interpreta como app -> properties)
```

**Solução:** Escapar ponto com `\` ou usar colchetes

```bash
# Método 1: Escapar ponto
kubectl get cm my-config -o jsonpath='{.data.app\.properties}'

# Método 2: Usar colchetes
kubectl get cm my-config -o jsonpath='{.data["app.properties"]}'
```

### 6. Debugar JSONPath complexo

```bash
# Passo 1: Ver JSON completo
kubectl get pod nginx -o json > pod.json

# Passo 2: Usar jq para testar query
cat pod.json | jq '.status.containerStatuses[0].ready'
# Output: true

# Passo 3: Converter para JSONPath kubectl
kubectl get pod nginx -o jsonpath='{.status.containerStatuses[0].ready}'
```

---

## 🔬 Troubleshooting Avançado

### 7. Campos opcionais e valores null

**Problema:**
```bash
# Tentar acessar loadBalancer.ingress em Service do tipo ClusterIP
kubectl get svc nginx -o jsonpath='{.status.loadBalancer.ingress[0].ip}'
# Output: (vazio - campo não existe em ClusterIP)
```

**Solução:** Verificar tipo do Service primeiro ou usar filtro condicional

```bash
# Ver tipo do Service
kubectl get svc nginx -o jsonpath='{.spec.type}{"\n"}'
# Output: ClusterIP

# Apenas Services do tipo LoadBalancer têm .status.loadBalancer.ingress

# Filtrar apenas LoadBalancers
kubectl get svc -o jsonpath='{range .items[?(@.spec.type=="LoadBalancer")]}{.metadata.name}{"\t"}{.status.loadBalancer.ingress[0].ip}{"\n"}{end}'
```

### 8. Lidar com arrays vazios

**Problema:**
```bash
# Pod sem volumes montados
kubectl get pod nginx -o jsonpath='{.spec.volumes[0].name}'
# Output: (vazio - array volumes não existe ou está vazio)
```

**Solução:** Verificar se array tem elementos antes

```bash
# Ver se tem volumes
kubectl get pod nginx -o json | jq '.spec.volumes'
# Output: null (ou [])

# Listar apenas Pods COM volumes
kubectl get pods -o json | jq -r '.items[] | select(.spec.volumes != null) | .metadata.name'

# Ou contar volumes
kubectl get pod nginx -o json | jq '.spec.volumes | length'
# Output: 0
```

### 9. Múltiplos containers - acessar específico

**Problema:**
```bash
# Pod com múltiplos containers, quer apenas um específico
kubectl get pod app -o jsonpath='{.spec.containers[0].image}'
# Output: nginx:1.21
# Mas você quer o container "sidecar", não o primeiro!
```

**Solução:** Usar filtro para encontrar container por nome

```bash
# Ver todos containers e seus nomes
kubectl get pod app -o jsonpath='{range .spec.containers[*]}{.name}{"\t"}{.image}{"\n"}{end}'
# Output:
# nginx       nginx:1.21
# sidecar     busybox:1.35
# logger      fluent-bit:2.0

# Filtrar container por nome
kubectl get pod app -o jsonpath='{.spec.containers[?(@.name=="sidecar")].image}'
# Output: busybox:1.35

# Usar jq (mais fácil para filtros complexos)
kubectl get pod app -o json | jq -r '.spec.containers[] | select(.name=="sidecar") | .image'
# Output: busybox:1.35
```

### 10. Caracteres especiais em nomes de campos

**Problema:**
```bash
# Annotation com caracteres especiais
kubectl get pod nginx -o jsonpath='{.metadata.annotations.kubectl.kubernetes.io/last-applied-configuration}'
# Error: erro de sintaxe
```

**Solução:** Usar colchetes e aspas duplas

```bash
# Método 1: Colchetes com aspas duplas
kubectl get pod nginx -o jsonpath='{.metadata.annotations["kubectl\.kubernetes\.io/last-applied-configuration"]}'

# Método 2: Usar jq (mais simples)
kubectl get pod nginx -o json | jq -r '.metadata.annotations."kubectl.kubernetes.io/last-applied-configuration"'

# Listar todas annotations
kubectl get pod nginx -o json | jq '.metadata.annotations'
```

### 11. Ordenar resultados

**Problema:**
```bash
# JSONPath não suporta ordenação nativa
kubectl get pods -o jsonpath='{.items[*].metadata.name}'
# Output: ordem aleatória
```

**Solução:** Usar `--sort-by` com JSONPath

```bash
# Ordenar Pods por nome
kubectl get pods --sort-by=.metadata.name

# Ordenar por creationTimestamp (mais recente primeiro)
kubectl get pods --sort-by=.metadata.creationTimestamp

# Ordenar Nodes por CPU capacity
kubectl get nodes --sort-by=.status.capacity.cpu

# Combinar com JSONPath
kubectl get pods --sort-by=.metadata.creationTimestamp -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.metadata.creationTimestamp}{"\n"}{end}'

# Ordenar reverso (usar tac ou sort -r)
kubectl get pods --sort-by=.metadata.creationTimestamp -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}' | tac
```

### 12. Contar elementos

**Problema:**
```bash
# Contar quantos Pods estão Running
```

**Solução:** Usar filtro + wc ou jq

```bash
# Método 1: JSONPath + filtro + wc
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}' | wc -w
# Output: 5

# Método 2: jq com length
kubectl get pods -o json | jq '[.items[] | select(.status.phase=="Running")] | length'
# Output: 5

# Contar total de containers em um Pod
kubectl get pod nginx -o json | jq '.spec.containers | length'
# Output: 2

# Contar Pods por Node
kubectl get pods -o json | jq -r '.items[] | .spec.nodeName' | sort | uniq -c
# Output:
#   3 node01
#   2 node02
#   4 node03
```

### 13. Extrair valores aninhados complexos

**Problema:**
```bash
# Extrair todos os environment variables de todos containers
```

**Solução:** Combinar range com filtros

```bash
# Método 1: JSONPath com nested range
kubectl get pod nginx -o jsonpath='{range .spec.containers[*]}{"\nContainer: "}{.name}{"\n"}{range .env[*]}{.name}{"="}{.value}{"\n"}{end}{end}'

# Método 2: jq (mais legível)
kubectl get pod nginx -o json | jq -r '.spec.containers[] | "Container: \(.name)", (.env[]? | "\(.name)=\(.value)")'

# Extrair todas as portas de todos containers
kubectl get pods -o json | jq -r '.items[] | .metadata.name as $pod | .spec.containers[] | "\($pod)\t\(.name)\t\(.ports[]?.containerPort // "no-port")"'
```

### 14. Trabalhar com timestamps

**Problema:**
```bash
# Converter timestamps para formato legível
kubectl get pods -o jsonpath='{.items[*].metadata.creationTimestamp}'
# Output: 2024-01-15T10:30:45Z (formato ISO)
```

**Solução:** Usar date para converter

```bash
# Extrair timestamp
timestamp=$(kubectl get pod nginx -o jsonpath='{.metadata.creationTimestamp}')

# Converter para formato legível
date -d "$timestamp" "+%Y-%m-%d %H:%M:%S"
# Output: 2024-01-15 10:30:45

# Calcular idade do Pod (em segundos)
created=$(kubectl get pod nginx -o jsonpath='{.metadata.creationTimestamp}')
now=$(date -u +%s)
created_epoch=$(date -d "$created" +%s)
age=$((now - created_epoch))
echo "Pod age: $age seconds ($((age / 60)) minutes)"

# Listar Pods com idade
kubectl get pods -o json | jq -r '.items[] | "\(.metadata.name)\t\(.metadata.creationTimestamp)"' | while read name timestamp; do
  age=$(( $(date +%s) - $(date -d "$timestamp" +%s) ))
  echo "$name\t$(($age / 60)) minutes old"
done
```

### 15. Combinar múltiplos JSONPath queries

**Problema:**
```bash
# Extrair múltiplas informações de diferentes níveis
```

**Solução:** Usar múltiplas expressões em um range

```bash
# Pod: Nome, Namespace, Node, IP, Status
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}{"\t"}{.metadata.name}{"\t"}{.spec.nodeName}{"\t"}{.status.podIP}{"\t"}{.status.phase}{"\n"}{end}'

# Node: Nome, IP interno, IP externo, versão kubelet, OS
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.addresses[?(@.type=="InternalIP")].address}{"\t"}{.status.addresses[?(@.type=="ExternalIP")].address}{"\t"}{.status.nodeInfo.kubeletVersion}{"\t"}{.status.nodeInfo.osImage}{"\n"}{end}'

# Service: Nome, Tipo, ClusterIP, Portas, Selector
kubectl get svc -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.type}{"\t"}{.spec.clusterIP}{"\t"}{.spec.ports[0].port}{"\t"}{.spec.selector}{"\n"}{end}'
```

### 16. Debugging: Ver exatamente o que JSONPath retorna

**Problema:**
```bash
# JSONPath retorna algo inesperado
kubectl get pods -o jsonpath='{.items[*].status.podIP}'
# Output: 10.244.1.5 10.244.1.6 (sem separação clara)
```

**Solução:** Usar diferentes separadores e formatos

```bash
# Com newline entre cada
kubectl get pods -o jsonpath='{range .items[*]}{.status.podIP}{"\n"}{end}'
# Output:
# 10.244.1.5
# 10.244.1.6

# Com tab e newline
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Com separador customizado (vírgula)
kubectl get pods -o jsonpath='{range .items[*]}{.status.podIP}{","}{end}' | sed 's/,$/\n/'
# Output: 10.244.1.5,10.244.1.6

# Debug: Ver tipo do valor retornado com jq
kubectl get pod nginx -o json | jq -r '.status.podIP | type'
# Output: string

kubectl get pod nginx -o json | jq -r '.spec.containers | type'
# Output: array
```

### 17. JSONPath com recursos que não existem

**Problema:**
```bash
# Tentar acessar Pod que não existe
kubectl get pod nao-existe -o jsonpath='{.metadata.name}'
# Error: pods "nao-existe" not found
```

**Solução:** Verificar existência primeiro ou usar grep para filtrar

```bash
# Verificar se existe antes
if kubectl get pod nginx &>/dev/null; then
  kubectl get pod nginx -o jsonpath='{.metadata.name}{"\n"}'
else
  echo "Pod não existe"
fi

# Listar apenas Pods que existem (óbvio, mas útil em scripts)
kubectl get pods --no-headers 2>/dev/null | awk '{print $1}' | while read pod; do
  ip=$(kubectl get pod $pod -o jsonpath='{.status.podIP}')
  echo "$pod: $ip"
done
```

### 18. Performance: JSONPath vs jq

**Problema:**
```bash
# JSONPath pode ser lento para queries complexas
```

**Comparação:**

```bash
# JSONPath (nativo kubectl, mais rápido para queries simples)
time kubectl get pods -o jsonpath='{.items[*].metadata.name}' > /dev/null
# real    0m0.150s

# jq (mais flexível, melhor para queries complexas)
time kubectl get pods -o json | jq -r '.items[].metadata.name' > /dev/null
# real    0m0.180s

# Para queries simples: use JSONPath (mais rápido)
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Para queries complexas com filtros: use jq (mais fácil)
kubectl get pods -o json | jq -r '.items[] | select(.status.phase=="Running" and .spec.nodeName=="node01") | .metadata.name'
```

**Quando usar cada um:**

| Cenário | Use | Exemplo |
|---------|-----|---------|
| Campo simples | JSONPath | `{.metadata.name}` |
| Lista simples | JSONPath | `{.items[*].metadata.name}` |
| Filtro simples | JSONPath | `{.items[?(@.status.phase=="Running")].metadata.name}` |
| Filtros complexos | jq | `select(.a and .b or .c)` |
| Manipulação de strings | jq | `split("/") | last` |
| Cálculos | jq | `(.a + .b) / 2` |
| Formatação JSON | jq | `{name, ip, node}` |

### 19. Escapar caracteres em valores

**Problema:**
```bash
# Label com caracteres especiais
kubectl get pods -l 'app.kubernetes.io/name=nginx'
# Como extrair com JSONPath?
```

**Solução:**

```bash
# Labels com barra precisam de escape ou colchetes
kubectl get pod nginx -o jsonpath='{.metadata.labels.app\.kubernetes\.io/name}'

# Ou usar colchetes
kubectl get pod nginx -o jsonpath='{.metadata.labels["app.kubernetes.io/name"]}'

# Com jq (mais simples)
kubectl get pod nginx -o json | jq -r '.metadata.labels."app.kubernetes.io/name"'

# Listar todas as labels
kubectl get pod nginx -o json | jq '.metadata.labels'
```

### 20. Criar scripts reutilizáveis com JSONPath

**Solução:** Criar funções bash

```bash
# Função para pegar IP de um Pod
get_pod_ip() {
  local pod=$1
  kubectl get pod "$pod" -o jsonpath='{.status.podIP}' 2>/dev/null
}

# Usar função
ip=$(get_pod_ip nginx)
echo "Nginx IP: $ip"

# Função para listar Pods em um Node
pods_on_node() {
  local node=$1
  kubectl get pods -A -o jsonpath="{range .items[?(@.spec.nodeName=='$node')]}{.metadata.namespace}{'/'}{.metadata.name}{'\n'}{end}"
}

# Usar
pods_on_node node01

# Função para contar recursos por namespace
count_by_namespace() {
  local resource=$1
  kubectl get "$resource" -A -o json | jq -r '.items[] | .metadata.namespace' | sort | uniq -c | sort -rn
}

# Usar
count_by_namespace pods
# Output:
#   15 kube-system
#   8 default
#   3 monitoring

# Salvar funções em arquivo
cat > ~/.kubectl_helpers.sh <<'EOF'
get_pod_ip() {
  kubectl get pod "$1" -o jsonpath='{.status.podIP}' 2>/dev/null
}

get_pod_node() {
  kubectl get pod "$1" -o jsonpath='{.spec.nodeName}' 2>/dev/null
}

list_pod_containers() {
  kubectl get pod "$1" -o jsonpath='{range .spec.containers[*]}{.name}{"\n"}{end}'
}

get_svc_clusterip() {
  kubectl get svc "$1" -o jsonpath='{.spec.clusterIP}' 2>/dev/null
}
EOF

# Carregar funções
source ~/.kubectl_helpers.sh

# Adicionar ao .bashrc para sempre carregar
echo "source ~/.kubectl_helpers.sh" >> ~/.bashrc
```

---

## 📝 Resumo

### JSONPath Básico

**Sintaxe:**
```bash
kubectl get <resource> <name> -o jsonpath='<expression>'
```

**Elementos comuns:**
- `.field` - Campo na raiz
- `.nested.field` - Campo aninhado
- `[0]` - Primeiro elemento do array
- `[*]` - Todos elementos do array
- `{"\n"}` - Newline
- `{"\t"}` - Tab

### Múltiplos recursos (items)

```bash
# Padrão: {.items[*]...}
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Com range para formatar
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\n"}{end}'
```

### Custom Columns

```bash
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP
```

### Exemplos Mais Usados

```bash
# IPs de Pods
kubectl get pods -o jsonpath='{.items[*].status.podIP}'

# Imagens de containers
kubectl get pods -o jsonpath='{.items[*].spec.containers[*].image}'

# Nodes de Pods
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.nodeName}{"\n"}{end}'

# ClusterIP de Services
kubectl get svc -o jsonpath='{.items[*].spec.clusterIP}'

# IPs de Nodes
kubectl get nodes -o jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}'

# Decodificar Secret
kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 --decode
```

---

## 🔧 Comandos Essenciais

```bash
# Ver JSON completo (entender estrutura)
kubectl get pod nginx -o json
kubectl get pod nginx -o yaml  # Mais legível

# Campo simples
kubectl get pod nginx -o jsonpath='{.metadata.name}'

# Com newline
kubectl get pod nginx -o jsonpath='{.metadata.name}{"\n"}'

# Array - primeiro elemento
kubectl get pod nginx -o jsonpath='{.spec.containers[0].name}'

# Array - todos elementos
kubectl get pod nginx -o jsonpath='{.spec.containers[*].name}'

# Múltiplos recursos
kubectl get pods -o jsonpath='{.items[*].metadata.name}'

# Range para formatar
kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'

# Custom columns
kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP

# Filtro
kubectl get pods -o jsonpath='{.items[?(@.status.phase=="Running")].metadata.name}'
```

---

## 💡 Dicas para o Exame CKA

1. **Ver estrutura JSON primeiro**
   ```bash
   # Sempre que não souber o caminho
   kubectl get pod nginx -o json | less
   # ou
   kubectl get pod nginx -o yaml | less
   ```

2. **Usar aspas simples**
   ```bash
   # ✅ Correto
   -o jsonpath='{.metadata.name}'

   # ❌ Errado
   -o jsonpath="{.metadata.name}"
   ```

3. **Não esquecer `.items[*]` para múltiplos recursos**
   ```bash
   # Um Pod
   kubectl get pod nginx -o jsonpath='{.metadata.name}'

   # Múltiplos Pods (precisa de .items)
   kubectl get pods -o jsonpath='{.items[*].metadata.name}'
   ```

4. **Adicionar `{"\n"}` para melhor leitura**
   ```bash
   kubectl get pod nginx -o jsonpath='{.status.podIP}{"\n"}'
   ```

5. **Range para formatar listas**
   ```bash
   kubectl get pods -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.status.podIP}{"\n"}{end}'
   ```

6. **Custom columns para tabelas**
   ```bash
   # Mais legível que JSONPath puro
   kubectl get pods -o custom-columns=NAME:.metadata.name,IP:.status.podIP,NODE:.spec.nodeName
   ```

7. **Campo case-sensitive**
   ```bash
   # ❌ Errado
   {.status.podip}

   # ✅ Correto
   {.status.podIP}
   ```

8. **Decodificar Secrets**
   ```bash
   kubectl get secret my-secret -o jsonpath='{.data.password}' | base64 --decode
   echo  # Adicionar newline
   ```

9. **Loop por namespaces**
   ```bash
   for ns in $(kubectl get ns -o jsonpath='{.items[*].metadata.name}'); do
     kubectl get pods -n $ns
   done
   ```

10. **Usar `jq` para queries complexas**
    ```bash
    # Se JSONPath for muito complexo, use jq
    kubectl get pods -o json | jq '.items[] | select(.status.phase=="Running") | .metadata.name'
    ```

---

## 🔗 Recursos Adicionais

- [Kubernetes JSONPath Support](https://kubernetes.io/docs/reference/kubectl/jsonpath/)
- [JSONPath Online Evaluator](https://jsonpath.com/)
- [kubectl Cheat Sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

---

⬅️ **Anterior**: [dicas-e-links.md](./dicas-e-links.md) | ➡️ **Próximo**: [../Workloads/pods.md](../Workloads/pods.md)
