# Admission Controllers

## 📋 O que são Admission Controllers?

**Admission Controllers** são plugins que interceptam requisições ao **kube-apiserver** após a autenticação/autorização, mas **antes** de persistir o objeto no etcd. Eles podem **validar**, **modificar** ou **rejeitar** requisições.

### Características principais:
- ✅ Executam **após** autenticação e autorização
- ✅ Executam **antes** de persistir no etcd
- ✅ Podem **modificar** objetos (mutating)
- ✅ Podem **validar** objetos (validating)
- ✅ Podem **rejeitar** requisições
- ✅ São compilados no kube-apiserver (não são plugins externos)

## 🔄 Fluxo de uma Requisição no API Server

```
┌─────────────────────────────────────────────────────────────┐
│  kubectl create pod nginx --image=nginx                     │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  1. Authentication (Autenticação)                           │
│     ├─ Certificados                                         │
│     ├─ Tokens                                               │
│     └─ Service Accounts                                     │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  2. Authorization (Autorização)                             │
│     ├─ RBAC (Role-Based Access Control)                     │
│     ├─ ABAC (Attribute-Based Access Control)                │
│     └─ Webhook                                              │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  3. MUTATING Admission Controllers                          │
│     ├─ NamespaceLifecycle                                   │
│     ├─ LimitRanger (adiciona limits padrão)                 │
│     ├─ ServiceAccount (injeta serviceAccountName)           │
│     ├─ DefaultStorageClass (adiciona storageClass padrão)   │
│     └─ MutatingAdmissionWebhook (webhooks customizados)     │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  4. Schema Validation (Validação do Schema)                 │
│     └─ Valida se o YAML está correto                        │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  5. VALIDATING Admission Controllers                        │
│     ├─ NamespaceExists (verifica se namespace existe)       │
│     ├─ ResourceQuota (verifica quotas)                      │
│     ├─ PodSecurityPolicy (valida políticas de segurança)    │
│     └─ ValidatingAdmissionWebhook (webhooks customizados)   │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  6. Persist to etcd                                         │
│     └─ Objeto é salvo no etcd                               │
└─────────────────────────────────────────────────────────────┘
```

## 🎯 Tipos de Admission Controllers

### 1. Mutating Admission Controllers
- **Modificam** a requisição antes de persistir
- Executam **primeiro** (antes dos validating)
- Exemplos: adicionar labels, injetar sidecars, definir valores padrão

### 2. Validating Admission Controllers
- **Validam** a requisição sem modificar
- Executam **depois** dos mutating
- Exemplos: verificar quotas, validar nomes, checar políticas

## 🔄 Validating e Mutating Admission Controllers (Detalhado)

### Diferenças Fundamentais

| Característica | Mutating | Validating |
|----------------|----------|------------|
| **Ordem de execução** | Primeiro | Segundo (depois dos mutating) |
| **Modifica objetos?** | ✅ Sim | ❌ Não |
| **Pode rejeitar?** | ✅ Sim | ✅ Sim |
| **Uso principal** | Adicionar defaults, injetar sidecars | Validar políticas, checar regras |
| **Exemplo** | Adicionar labels, injetar volumes | Verificar quotas, bloquear nomes |

### Por que a ordem importa?

```
┌─────────────────────────────────────────────────────────────┐
│  1. MUTATING Controllers executam primeiro                  │
│     ├─ Podem MODIFICAR o objeto                             │
│     ├─ Adicionam campos faltantes                           │
│     └─ Injetam recursos (sidecars, volumes, etc)            │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  2. Schema Validation                                       │
│     └─ Valida estrutura YAML após mutações                  │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  3. VALIDATING Controllers executam por último              │
│     ├─ Recebem objeto JÁ MODIFICADO pelos mutating         │
│     ├─ NÃO podem mais modificar                            │
│     └─ Apenas ACEITAM ou REJEITAM                          │
└─────────────────────────────────────────────────────────────┘
```

**Por que essa ordem?**
- **Mutating primeiro**: Completa o objeto com defaults antes de validar
- **Validating depois**: Valida o objeto final (já com todas as modificações)

### Exemplos de Mutating Admission Controllers

#### 1. ServiceAccount (Built-in Mutating)

**O que faz:** Injeta automaticamente `serviceAccountName` e monta o token

```yaml
# Você envia:
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx
```

```yaml
# ServiceAccount Controller adiciona:
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  serviceAccountName: default    # ← ADICIONADO
  automountServiceAccountToken: true
  containers:
  - name: nginx
    image: nginx
    volumeMounts:                 # ← ADICIONADO
    - name: kube-api-access-xxxxx
      mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      readOnly: true
  volumes:                        # ← ADICIONADO
  - name: kube-api-access-xxxxx
    projected:
      sources:
      - serviceAccountToken:
          path: token
```

#### 2. DefaultStorageClass (Built-in Mutating)

**O que faz:** Adiciona `storageClassName` padrão em PVCs

```yaml
# Você envia:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

```yaml
# DefaultStorageClass adiciona:
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  storageClassName: standard    # ← ADICIONADO
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

#### 3. MutatingAdmissionWebhook (Customizado)

**Exemplo: Istio Service Mesh**

Istio usa um webhook mutating para injetar automaticamente o sidecar proxy (envoy) em todos os pods.

```yaml
# Você cria:
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: default
  labels:
    app: myapp
spec:
  containers:
  - name: app
    image: myapp:1.0
```

```yaml
# Istio webhook injeta sidecar:
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: default
  labels:
    app: myapp
  annotations:
    sidecar.istio.io/status: injected    # ← ADICIONADO
spec:
  initContainers:                         # ← ADICIONADO
  - name: istio-init
    image: istio/proxyv2:1.20.0
    # ... configura iptables
  containers:
  - name: app
    image: myapp:1.0
  - name: istio-proxy                     # ← ADICIONADO (sidecar)
    image: istio/proxyv2:1.20.0
    # ... configuração do Envoy proxy
  volumes:                                # ← ADICIONADO
  - name: istio-envoy
    emptyDir: {}
  - name: istio-certs
    secret:
      secretName: istio.default
```

**Como configurar webhook mutating customizado:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: istio-sidecar-injector
webhooks:
- name: sidecar-injector.istio.io
  clientConfig:
    service:
      name: istio-sidecar-injector
      namespace: istio-system
      path: "/inject"
    caBundle: <base64-encoded-ca-cert>
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  namespaceSelector:
    matchLabels:
      istio-injection: enabled    # Apenas namespaces com este label
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail              # Se webhook falhar, rejeita pod
```

### Exemplos de Validating Admission Controllers

#### 1. ResourceQuota (Built-in Validating)

**O que faz:** Valida se há quota disponível no namespace

```yaml
# ResourceQuota no namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-quota
  namespace: dev
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "20"
```

```bash
# Tentativa de criar pod que excede quota
kubectl run nginx --image=nginx -n dev

# ResourceQuota Controller REJEITA:
Error from server (Forbidden): pods "nginx" is forbidden:
exceeded quota: compute-quota, requested: pods=1,
used: pods=20, limited: pods=20
```

**Não modifica nada - apenas VALIDA e REJEITA se exceder!**

#### 2. PodSecurity (Built-in Validating)

**O que faz:** Valida pods contra padrões de segurança

```yaml
# Namespace com Pod Security Admission (restricted)
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
```

```yaml
# Tentativa de criar pod privilegiado
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
  namespace: production
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true    # ❌ Não permitido em "restricted"
```

```bash
kubectl apply -f privileged-pod.yaml

# PodSecurity Controller REJEITA:
Error from server (Forbidden): error when creating "pod.yaml":
pods "privileged-pod" is forbidden: violates PodSecurity
"restricted:latest": privileged (container "nginx" must not set
securityContext.privileged=true)
```

#### 3. ValidatingAdmissionWebhook (Customizado)

**Exemplo: Open Policy Agent (OPA)**

OPA valida políticas customizadas via webhook.

**Política:** Bloquear imagens sem tag específica em produção

```rego
# Política OPA (policy.rego)
package kubernetes.admission

deny[msg] {
  input.request.kind.kind == "Pod"
  input.request.namespace == "production"
  image := input.request.object.spec.containers[_].image
  not contains(image, ":")
  msg := sprintf("Image '%v' must have explicit tag in production", [image])
}

deny[msg] {
  input.request.kind.kind == "Pod"
  input.request.namespace == "production"
  image := input.request.object.spec.containers[_].image
  endswith(image, ":latest")
  msg := sprintf("Image '%v' cannot use 'latest' tag in production", [image])
}
```

**Configuração do webhook:**

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: opa-validating-webhook
webhooks:
- name: validating-webhook.openpolicyagent.org
  clientConfig:
    service:
      name: opa
      namespace: opa
      path: "/v1/admit"
    caBundle: <base64-ca-cert>
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
```

**Teste:**

```bash
# Pod com tag 'latest' será REJEITADO
kubectl run nginx --image=nginx:latest -n production
# Error: admission webhook "validating-webhook.openpolicyagent.org" denied:
# Image 'nginx:latest' cannot use 'latest' tag in production

# Pod sem tag será REJEITADO
kubectl run nginx --image=nginx -n production
# Error: Image 'nginx' must have explicit tag in production

# Pod com tag específica será ACEITO
kubectl run nginx --image=nginx:1.21.0 -n production
# pod/nginx created ✅
```

### Cenários Práticos: Mutating + Validating Juntos

#### Cenário 1: Deploy com Defaults e Validação

```yaml
# 1. Você envia este pod (mínimo)
apiVersion: v1
kind: Pod
metadata:
  name: app
  namespace: production
spec:
  containers:
  - name: app
    image: myapp:1.0
```

**Pipeline de Admission:**

1. **Mutating Controllers:**
   - `ServiceAccount`: Adiciona `serviceAccountName: default` e monta token
   - `LimitRanger`: Adiciona `resources.requests` e `resources.limits` (se LimitRange existir)
   - `MutatingWebhook` (Istio): Injeta sidecar proxy

2. **Schema Validation:** Valida estrutura YAML após mutações

3. **Validating Controllers:**
   - `ResourceQuota`: Verifica se namespace tem quota disponível
   - `PodSecurity`: Valida se pod atende padrão "restricted"
   - `ValidatingWebhook` (OPA): Valida políticas customizadas (imagem não é `latest`, etc.)

4. **Se TODOS passarem:** Pod é criado no etcd ✅

5. **Se ALGUM rejeitar:** Pod não é criado, erro retornado ❌

#### Cenário 2: Istio + OPA + ResourceQuota

**Setup:**
- Namespace com `istio-injection: enabled`
- OPA valida tags de imagem
- ResourceQuota limita recursos

```bash
# 1. Criar namespace
kubectl create namespace prod
kubectl label namespace prod istio-injection=enabled

# 2. Criar ResourceQuota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: prod-quota
  namespace: prod
spec:
  hard:
    requests.cpu: "10"
    requests.memory: 20Gi
    pods: "50"
EOF

# 3. Tentar criar pod
kubectl run app --image=myapp:1.0 -n prod
```

**Pipeline:**
1. **Mutating:**
   - Istio injeta sidecar (2 containers agora)
   - ServiceAccount adiciona token
2. **Validating:**
   - ResourceQuota valida: `pods=1` OK, `cpu/memory` do app+sidecar OK
   - OPA valida: tag `1.0` OK (não é `latest`)
3. **Resultado:** Pod criado com sidecar ✅

```bash
# Ver que sidecar foi injetado
kubectl get pod app -n prod -o jsonpath='{.spec.containers[*].name}'
# Output: app istio-proxy
```

### Implementando um Webhook Customizado

#### Exemplo: Webhook que adiciona label automático (Mutating)

```python
from flask import Flask, request, jsonify
import base64
import json

app = Flask(__name__)

@app.route('/mutate', methods=['POST'])
def mutate():
    admission_review = request.json

    # Extrair pod do request
    pod = admission_review['request']['object']

    # Adicionar label customizado
    if 'labels' not in pod['metadata']:
        pod['metadata']['labels'] = {}

    pod['metadata']['labels']['mutated-by'] = 'custom-webhook'
    pod['metadata']['labels']['mutated-at'] = '2024-03-05'

    # Criar patch JSON (RFC 6902 JSON Patch)
    patch = [
        {
            "op": "add",
            "path": "/metadata/labels/mutated-by",
            "value": "custom-webhook"
        },
        {
            "op": "add",
            "path": "/metadata/labels/mutated-at",
            "value": "2024-03-05"
        }
    ]

    # Codificar patch em base64
    patch_base64 = base64.b64encode(json.dumps(patch).encode()).decode()

    # Retornar AdmissionReview com patch
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": admission_review['request']['uid'],
            "allowed": True,
            "patchType": "JSONPatch",
            "patch": patch_base64
        }
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8443, ssl_context=('cert.pem', 'key.pem'))
```

#### Exemplo: Webhook que valida labels obrigatórios (Validating)

```python
@app.route('/validate', methods=['POST'])
def validate():
    admission_review = request.json
    pod = admission_review['request']['object']

    # Labels obrigatórios
    required_labels = ['app', 'env', 'team']

    # Verificar se todos os labels existem
    labels = pod['metadata'].get('labels', {})
    missing = [label for label in required_labels if label not in labels]

    if missing:
        return jsonify({
            "apiVersion": "admission.k8s.io/v1",
            "kind": "AdmissionReview",
            "response": {
                "uid": admission_review['request']['uid'],
                "allowed": False,
                "status": {
                    "message": f"Missing required labels: {', '.join(missing)}. Required labels are: {', '.join(required_labels)}"
                }
            }
        })

    # Todos os labels presentes, aceitar
    return jsonify({
        "apiVersion": "admission.k8s.io/v1",
        "kind": "AdmissionReview",
        "response": {
            "uid": admission_review['request']['uid'],
            "allowed": True
        }
    })
```

### Troubleshooting de Admission Controllers

#### Pod rejeitado por webhook

```bash
# 1. Ver erro detalhado
kubectl describe pod <pod-name>

# 2. Ver eventos de admission
kubectl get events -A | grep -i "admission\|webhook\|denied"

# 3. Ver logs do webhook
kubectl logs -n <webhook-namespace> <webhook-pod>

# 4. Testar conectividade do API server para webhook
kubectl run test -n <webhook-namespace> --image=busybox --rm -it -- \
  wget -O- http://<webhook-service>:8443/health
```

#### Webhook não está sendo chamado

```bash
# 1. Verificar configuração
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# 2. Ver detalhes
kubectl describe mutatingwebhookconfiguration <name>
kubectl describe validatingwebhookconfiguration <name>

# 3. Verificar se webhook está rodando
kubectl get pods -n <webhook-namespace>

# 4. Ver logs do kube-apiserver
kubectl logs -n kube-system kube-apiserver-<node> | grep webhook
```

#### Webhook muito lento (timeout)

```yaml
# Aumentar timeout no webhook config
webhooks:
- name: my-webhook
  timeoutSeconds: 30    # Padrão: 10s, Max: 30s
  failurePolicy: Ignore  # Ou Fail
```

### Boas Práticas

#### Para Mutating Webhooks:

1. **Seja idempotente**
   - Aplicar mutação múltiplas vezes deve ter mesmo resultado

2. **Use JSON Patch**
   - Mais eficiente que retornar objeto completo

3. **Não faça mudanças grandes**
   - Adicionar labels: OK
   - Adicionar sidecar: OK
   - Reescrever todo o pod: ❌ Evite

#### Para Validating Webhooks:

1. **Mensagens de erro claras**
   ```json
   {
     "allowed": false,
     "status": {
       "message": "Image 'nginx:latest' uses forbidden 'latest' tag. Use semantic versioning like 'nginx:1.21.0'"
     }
   }
   ```

2. **Seja rápido**
   - Timeout padrão: 10s
   - Seu webhook deve responder em < 1s

3. **failurePolicy: Fail para produção**
   ```yaml
   failurePolicy: Fail    # Mais seguro
   # vs
   failurePolicy: Ignore  # Apenas para dev/test
   ```

#### Para ambos:

1. **Monitore performance**
   - Latência do webhook
   - Taxa de rejeição
   - Disponibilidade

2. **Use namespaceSelector**
   ```yaml
   namespaceSelector:
     matchLabels:
       webhook-enabled: "true"
   ```

3. **Implemente health checks**
   ```python
   @app.route('/health', methods=['GET'])
   def health():
       return jsonify({"status": "healthy"}), 200
   ```

4. **TLS é obrigatório**
   - Webhooks DEVEM usar HTTPS
   - API Server valida certificado

## 📚 Principais Admission Controllers

### NamespaceLifecycle
```yaml
# O que faz:
# - Impede criar objetos em namespaces que estão sendo deletados
# - Impede deletar namespaces do sistema (kube-system, kube-public)
```

**Exemplo:**
```bash
# Tentar criar pod em namespace que não existe
kubectl run nginx --image=nginx -n nonexistent
# Erro: namespaces "nonexistent" not found
```

### LimitRanger
```yaml
# O que faz:
# - Aplica LimitRange padrão aos pods
# - Adiciona limits/requests se não especificados
```

**Exemplo:**
```yaml
# LimitRange no namespace
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    type: Container
```

```yaml
# Pod SEM resources definidos
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx

# LimitRanger adiciona automaticamente:
# resources:
#   limits:
#     memory: 512Mi
#     cpu: 500m
#   requests:
#     memory: 256Mi
#     cpu: 250m
```

### ResourceQuota
```yaml
# O que faz:
# - Valida se namespace tem quota disponível
# - Rejeita criação se exceder quota
```

**Exemplo:**
```yaml
# ResourceQuota no namespace
apiVersion: v1
kind: ResourceQuota
metadata:
  name: dev-quota
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    pods: "10"
```

```bash
# Tentar criar 11º pod
kubectl run nginx-11 --image=nginx
# Erro: exceeded quota: dev-quota, requested: pods=1, used: pods=10, limited: pods=10
```

### ServiceAccount
```yaml
# O que faz:
# - Injeta serviceAccountName se não especificado
# - Monta automaticamente o token do ServiceAccount
```

**Exemplo:**
```yaml
# Pod SEM serviceAccountName
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
  - name: nginx
    image: nginx

# ServiceAccount adiciona automaticamente:
# spec:
#   serviceAccountName: default
#   volumes:
#   - name: kube-api-access-xxxxx
#     projected:
#       sources:
#       - serviceAccountToken:
#           path: token
```

### DefaultStorageClass
```yaml
# O que faz:
# - Adiciona storageClassName padrão em PVCs
```

**Exemplo:**
```yaml
# PVC SEM storageClassName
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: my-pvc
spec:
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi

# DefaultStorageClass adiciona automaticamente:
# storageClassName: standard  # ou outro padrão do cluster
```

### NodeRestriction
```yaml
# O que faz:
# - Limita o que kubelets podem modificar
# - Previne kubelet de modificar labels de outros nós
# - Previne kubelet de modificar taints de outros nós
```

**Segurança importante**: Previne que um nó comprometido afete outros nós.

### PodSecurityPolicy (DEPRECATED → substituído por PodSecurity)
```yaml
# O que faz:
# - Valida políticas de segurança de pods
# - Controla: privileged, hostNetwork, volumes, etc.
```

**Nota**: PSP foi deprecado em v1.21 e removido em v1.25. Substituído por **Pod Security Admission**.

### PodSecurity (Novo - substitui PSP)
```yaml
# O que faz:
# - Valida pods contra padrões de segurança
# - Níveis: privileged, baseline, restricted
```

**Exemplo:**
```yaml
# Aplicar no namespace
apiVersion: v1
kind: Namespace
metadata:
  name: my-namespace
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### MutatingAdmissionWebhook
```yaml
# O que faz:
# - Chama webhook HTTP externo para MODIFICAR objetos
# - Permite lógica customizada de mutação
```

**Exemplo de uso**: Istio injeta sidecars automaticamente

### ValidatingAdmissionWebhook
```yaml
# O que faz:
# - Chama webhook HTTP externo para VALIDAR objetos
# - Permite lógica customizada de validação
```

**Exemplo de uso**: Open Policy Agent (OPA) valida políticas customizadas

### ImagePolicyWebhook
```yaml
# O que faz:
# - Valida imagens de containers antes de criar pods
# - Chama um webhook externo para aprovar/rejeitar imagens
# - Permite integração com scanners de vulnerabilidade
```

**ImagePolicyWebhook** é um admission controller que permite validar imagens de containers através de um webhook externo antes de permitir a criação de pods. É muito útil para implementar políticas de segurança de imagens.

### Casos de Uso do ImagePolicyWebhook

1. **Scanning de Vulnerabilidades**
   - Integrar com Trivy, Clair, Anchore para bloquear imagens com vulnerabilidades críticas

2. **Restrição de Registries**
   - Permitir apenas imagens de registries aprovados (docker.io, gcr.io específico, registry privado)
   - Bloquear imagens de fontes desconhecidas

3. **Validação de Assinaturas**
   - Verificar se imagens estão assinadas (Sigstore, Notary)
   - Garantir integridade e proveniência das imagens

4. **Políticas de Tags**
   - Bloquear uso de tag `latest` em produção
   - Exigir tags semânticas (v1.2.3)

### Como Funciona

```
┌─────────────────────────────────────────────────────────────┐
│  kubectl create pod nginx --image=nginx:1.21                │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  API Server: Authentication + Authorization                 │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  ImagePolicyWebhook Admission Controller                    │
│  ├─ Extrai imagens do pod spec                              │
│  ├─ Envia para webhook externo                              │
│  └─ Aguarda resposta (allow/deny)                           │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  Webhook Externo (Image Scanner)                            │
│  ├─ Valida imagem (registry, tag, vulnerabilidades)         │
│  ├─ Escaneia com Trivy/Clair/Anchore                        │
│  └─ Retorna: "allowed": true/false                          │
└─────────────────────┬───────────────────────────────────────┘
                      │
                      ↓
┌─────────────────────────────────────────────────────────────┐
│  Se "allowed": true  → Pod criado                           │
│  Se "allowed": false → Erro: image rejected by policy       │
└─────────────────────────────────────────────────────────────┘
```

### Configuração do ImagePolicyWebhook

#### Passo 1: Habilitar o Admission Controller

Editar `/etc/kubernetes/manifests/kube-apiserver.yaml`:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NodeRestriction,ImagePolicyWebhook
    - --admission-control-config-file=/etc/kubernetes/admission-config.yaml
    # ... outros parâmetros
    volumeMounts:
    - name: admission-config
      mountPath: /etc/kubernetes/admission-config.yaml
      readOnly: true
    - name: webhook-config
      mountPath: /etc/kubernetes/webhook-config.yaml
      readOnly: true
  volumes:
  - name: admission-config
    hostPath:
      path: /etc/kubernetes/admission-config.yaml
      type: File
  - name: webhook-config
    hostPath:
      path: /etc/kubernetes/webhook-config.yaml
      type: File
```

#### Passo 2: Criar Arquivo de Configuração do Admission Controller

Arquivo `/etc/kubernetes/admission-config.yaml`:

```yaml
apiVersion: apiserver.config.k8s.io/v1
kind: AdmissionConfiguration
plugins:
- name: ImagePolicyWebhook
  configuration:
    imagePolicy:
      kubeConfigFile: /etc/kubernetes/webhook-config.yaml
      allowTTL: 50           # Cache de imagens aprovadas (50s)
      denyTTL: 50            # Cache de imagens rejeitadas (50s)
      retryBackoff: 500      # Retry delay em ms
      defaultAllow: false    # Se webhook falhar, bloquear (false) ou permitir (true)
```

**Parâmetros importantes:**
- `defaultAllow: false` (SEGURO): Se webhook não responder, BLOQUEIA pods
- `defaultAllow: true` (INSEGURO): Se webhook não responder, PERMITE pods
- `allowTTL/denyTTL`: Cache para evitar consultas repetidas

#### Passo 3: Criar Configuração do Webhook

Arquivo `/etc/kubernetes/webhook-config.yaml` (formato kubeconfig):

```yaml
apiVersion: v1
kind: Config
clusters:
- name: image-scanner
  cluster:
    certificate-authority: /etc/kubernetes/pki/ca.crt
    server: https://image-scanner.default.svc:8080/validate
contexts:
- name: image-scanner
  context:
    cluster: image-scanner
    user: api-server
current-context: image-scanner
users:
- name: api-server
  user:
    client-certificate: /etc/kubernetes/pki/apiserver.crt
    client-key: /etc/kubernetes/pki/apiserver.key
```

### Formato da Requisição ao Webhook

O kube-apiserver envia uma requisição POST ao webhook:

```json
{
  "apiVersion": "imagepolicy.k8s.io/v1alpha1",
  "kind": "ImageReview",
  "spec": {
    "containers": [
      {
        "image": "nginx:1.21"
      },
      {
        "image": "redis:6.2"
      }
    ],
    "annotations": {
      "namespace": "default"
    },
    "namespace": "default"
  }
}
```

### Formato da Resposta do Webhook

O webhook deve responder com:

```json
{
  "apiVersion": "imagepolicy.k8s.io/v1alpha1",
  "kind": "ImageReview",
  "status": {
    "allowed": true,
    "reason": "Image nginx:1.21 passed vulnerability scan"
  }
}
```

Se a imagem for rejeitada:

```json
{
  "apiVersion": "imagepolicy.k8s.io/v1alpha1",
  "kind": "ImageReview",
  "status": {
    "allowed": false,
    "reason": "Image nginx:1.21 has critical vulnerabilities (CVE-2023-1234)"
  }
}
```

### Exemplo de Implementação de Webhook

Exemplo simples em Python (Flask):

```python
from flask import Flask, request, jsonify

app = Flask(__name__)

ALLOWED_REGISTRIES = [
    "docker.io/library",
    "gcr.io/my-project",
    "registry.company.com"
]

BLOCKED_TAGS = ["latest", "dev", "test"]

@app.route('/validate', methods=['POST'])
def validate_image():
    review = request.json
    containers = review.get('spec', {}).get('containers', [])

    for container in containers:
        image = container.get('image', '')

        # 1. Verificar registry
        if not any(image.startswith(registry) for registry in ALLOWED_REGISTRIES):
            return jsonify({
                "apiVersion": "imagepolicy.k8s.io/v1alpha1",
                "kind": "ImageReview",
                "status": {
                    "allowed": False,
                    "reason": f"Image {image} from unauthorized registry"
                }
            })

        # 2. Verificar tag
        if ':' in image:
            tag = image.split(':')[-1]
            if tag in BLOCKED_TAGS:
                return jsonify({
                    "apiVersion": "imagepolicy.k8s.io/v1alpha1",
                    "kind": "ImageReview",
                    "status": {
                        "allowed": False,
                        "reason": f"Tag '{tag}' is not allowed in production"
                    }
                })

        # 3. Escanear vulnerabilidades (integração com Trivy)
        # vulnerabilities = scan_with_trivy(image)
        # if vulnerabilities['critical'] > 0:
        #     return deny(f"Image has {vulnerabilities['critical']} critical CVEs")

    # Todas as imagens aprovadas
    return jsonify({
        "apiVersion": "imagepolicy.k8s.io/v1alpha1",
        "kind": "ImageReview",
        "status": {
            "allowed": True,
            "reason": "All images passed policy checks"
        }
    })

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080, ssl_context=('cert.pem', 'key.pem'))
```

### Integração com Trivy Scanner

Exemplo de webhook que usa Trivy para escanear imagens:

```python
import subprocess
import json

def scan_with_trivy(image):
    """Escaneia imagem com Trivy e retorna vulnerabilidades"""
    cmd = [
        'trivy', 'image',
        '--format', 'json',
        '--severity', 'CRITICAL,HIGH',
        image
    ]

    result = subprocess.run(cmd, capture_output=True, text=True)
    scan_result = json.loads(result.stdout)

    # Contar vulnerabilidades
    critical = 0
    high = 0

    for result in scan_result.get('Results', []):
        for vuln in result.get('Vulnerabilities', []):
            if vuln['Severity'] == 'CRITICAL':
                critical += 1
            elif vuln['Severity'] == 'HIGH':
                high += 1

    return {
        'critical': critical,
        'high': high
    }

@app.route('/validate', methods=['POST'])
def validate_image():
    review = request.json
    containers = review.get('spec', {}).get('containers', [])

    for container in containers:
        image = container.get('image', '')

        # Escanear com Trivy
        vulns = scan_with_trivy(image)

        if vulns['critical'] > 0:
            return jsonify({
                "apiVersion": "imagepolicy.k8s.io/v1alpha1",
                "kind": "ImageReview",
                "status": {
                    "allowed": False,
                    "reason": f"Image {image} has {vulns['critical']} CRITICAL vulnerabilities"
                }
            })

        if vulns['high'] > 5:
            return jsonify({
                "apiVersion": "imagepolicy.k8s.io/v1alpha1",
                "kind": "ImageReview",
                "status": {
                    "allowed": False,
                    "reason": f"Image {image} has too many HIGH vulnerabilities ({vulns['high']})"
                }
            })

    return jsonify({
        "apiVersion": "imagepolicy.k8s.io/v1alpha1",
        "kind": "ImageReview",
        "status": {
            "allowed": True,
            "reason": "Images passed vulnerability scan"
        }
    })
```

### Exemplo Prático: Bloquear Tag "latest"

**Cenário**: Queremos bloquear pods que usam imagem com tag `latest` em produção.

**Webhook simples:**

```python
@app.route('/validate', methods=['POST'])
def validate_image():
    review = request.json
    containers = review.get('spec', {}).get('containers', [])
    namespace = review.get('spec', {}).get('namespace', '')

    # Aplicar apenas em namespace production
    if namespace == 'production':
        for container in containers:
            image = container.get('image', '')

            # Se não tem tag ou tem tag 'latest'
            if ':' not in image or image.endswith(':latest'):
                return jsonify({
                    "apiVersion": "imagepolicy.k8s.io/v1alpha1",
                    "kind": "ImageReview",
                    "status": {
                        "allowed": False,
                        "reason": f"Image '{image}' uses 'latest' tag, which is not allowed in production. Use specific version tags (e.g., nginx:1.21.0)"
                    }
                })

    return jsonify({
        "apiVersion": "imagepolicy.k8s.io/v1alpha1",
        "kind": "ImageReview",
        "status": {
            "allowed": True,
            "reason": "Images approved"
        }
    })
```

**Teste:**

```bash
# Este pod será REJEITADO
kubectl run nginx --image=nginx:latest -n production
# Error: admission webhook denied the request: Image 'nginx:latest' uses 'latest' tag

# Este pod será ACEITO
kubectl run nginx --image=nginx:1.21.0 -n production
# pod/nginx created
```

### Soluções Comerciais com ImagePolicyWebhook

1. **Anchore Engine**
   - Scanner de vulnerabilidades open-source
   - Integra com ImagePolicyWebhook
   - Políticas customizáveis

2. **Aqua Security**
   - Scanner enterprise com políticas avançadas
   - Compliance checks (CIS, PCI-DSS)

3. **Sysdig Secure**
   - Scanning inline durante deploy
   - Runtime protection

4. **Falco + OPA**
   - Falco detecta comportamento anômalo
   - OPA valida políticas de imagens

### Troubleshooting do ImagePolicyWebhook

#### Pod é rejeitado sem motivo claro

```bash
# Ver eventos
kubectl get events -A | grep -i "image.*denied\|admission.*denied"

# Ver logs do kube-apiserver
kubectl logs -n kube-system kube-apiserver-<node> | grep ImagePolicyWebhook

# Ver logs do webhook
kubectl logs -n default image-scanner-pod
```

#### Webhook não responde

```bash
# Verificar se webhook está rodando
kubectl get pods -l app=image-scanner

# Testar conectividade do control plane para o webhook
kubectl run test --image=busybox --rm -it -- wget -O- http://image-scanner.default.svc:8080/health

# Ver configuração do webhook
cat /etc/kubernetes/webhook-config.yaml

# Ver se certificados estão corretos
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout
```

#### defaultAllow está permitindo imagens não-seguras

```bash
# Editar admission-config.yaml
sudo vi /etc/kubernetes/admission-config.yaml

# Alterar:
# defaultAllow: true   # INSEGURO
# para:
# defaultAllow: false  # SEGURO

# Reiniciar kube-apiserver (modificar manifest)
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
# (Adicionar espaço e salvar para forçar restart)
```

### ImagePolicyWebhook vs ValidatingAdmissionWebhook

| Característica | ImagePolicyWebhook | ValidatingAdmissionWebhook |
|----------------|-------------------|---------------------------|
| **Escopo** | Apenas imagens de containers | Qualquer objeto Kubernetes |
| **Configuração** | Via arquivos no control plane | Via objetos Kubernetes (ValidatingWebhookConfiguration) |
| **Formato** | ImageReview (imagepolicy.k8s.io/v1alpha1) | AdmissionReview (admission.k8s.io/v1) |
| **Foco** | Validação de imagens | Validação genérica |
| **Cache** | allowTTL/denyTTL embutidos | Sem cache nativo |
| **Uso comum** | Scanners de vulnerabilidade | Políticas OPA, validações customizadas |

**Quando usar ImagePolicyWebhook:**
- Você quer especificamente validar imagens
- Precisa de cache embutido (allowTTL/denyTTL)
- Quer usar formato ImageReview dedicado

**Quando usar ValidatingAdmissionWebhook:**
- Validação mais genérica de objetos
- Múltiplos tipos de validações
- Mais flexibilidade na configuração

### Boas Práticas

1. **defaultAllow: false em produção**
   - Se webhook falhar, é mais seguro bloquear do que permitir

2. **Use cache (allowTTL/denyTTL)**
   - Evita sobrecarga no webhook para mesmas imagens
   - 50-300 segundos é um bom valor

3. **Implemente health checks no webhook**
   - Endpoint `/health` ou `/readiness`
   - Monitore disponibilidade

4. **Use timeouts curtos**
   - `retryBackoff: 500` (500ms)
   - Não trave deployments por webhook lento

5. **Teste antes de aplicar em produção**
   - Comece com `defaultAllow: true` + logging
   - Analise quais imagens seriam bloqueadas
   - Depois mude para `defaultAllow: false`

6. **Políticas por namespace**
   - Mais restritivo em production
   - Mais permissivo em dev/test

7. **Monitore e alerte**
   - Log todas as rejeições
   - Alerte quando muitas imagens são bloqueadas
   - Dashboard com métricas

### Limitações do ImagePolicyWebhook

1. **Não previne pull de imagens já aprovadas**
   - Se imagem foi aprovada uma vez, pode ser usada depois mesmo se ficar vulnerável

2. **Não escaneia runtime**
   - Valida apenas no momento de criação do pod
   - Não detecta vulnerabilidades descobertas depois

3. **Dependência do webhook externo**
   - Se webhook cair, pode bloquear todos os deployments (se `defaultAllow: false`)

4. **Overhead em performance**
   - Cada criação de pod consulta webhook
   - Cache ajuda, mas ainda adiciona latência

**Solução complementar**: Use runtime security (Falco, Aqua) para proteção contínua.

## 🔧 Configurando Admission Controllers

### Ver Admission Controllers habilitados

```bash
# Ver configuração do kube-apiserver
kubectl -n kube-system get pod kube-apiserver-<node> -o yaml | grep enable-admission-plugins

# OU via processo do apiserver
ps aux | grep kube-apiserver | grep admission-plugins
```

### Habilitar/Desabilitar Admission Controllers

Editar o manifest do kube-apiserver (static pod):

```bash
# Editar manifest
sudo vi /etc/kubernetes/manifests/kube-apiserver.yaml
```

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: kube-apiserver
  namespace: kube-system
spec:
  containers:
  - command:
    - kube-apiserver
    - --enable-admission-plugins=NodeRestriction,NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota
    - --disable-admission-plugins=PodSecurityPolicy
    # ... outros parâmetros
```

**Salvar e aguardar**: O kubelet recria o pod automaticamente.

### Admission Controllers Recomendados

```bash
--enable-admission-plugins=\
  NodeRestriction,\
  NamespaceLifecycle,\
  LimitRanger,\
  ServiceAccount,\
  DefaultStorageClass,\
  ResourceQuota,\
  PodSecurity
```

## 🧪 Exemplos Práticos

### Exemplo 1: LimitRanger em ação

```bash
# 1. Criar namespace
kubectl create namespace test-limitrange

# 2. Criar LimitRange
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: test-limitrange
spec:
  limits:
  - default:
      memory: 512Mi
      cpu: 500m
    defaultRequest:
      memory: 256Mi
      cpu: 250m
    type: Container
EOF

# 3. Criar pod SEM resources
kubectl run nginx --image=nginx -n test-limitrange

# 4. Verificar que LimitRanger adicionou os values
kubectl get pod nginx -n test-limitrange -o yaml | grep -A 10 resources:

# Output:
#   resources:
#     limits:
#       cpu: 500m
#       memory: 512Mi
#     requests:
#       cpu: 250m
#       memory: 256Mi
```

### Exemplo 2: ResourceQuota bloqueando criação

```bash
# 1. Criar namespace
kubectl create namespace test-quota

# 2. Criar ResourceQuota
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: quota
  namespace: test-quota
spec:
  hard:
    pods: "2"
    requests.cpu: "1"
    requests.memory: 1Gi
EOF

# 3. Criar primeiro pod (OK)
kubectl run pod1 --image=nginx -n test-quota

# 4. Criar segundo pod (OK)
kubectl run pod2 --image=nginx -n test-quota

# 5. Tentar criar terceiro pod (FALHA)
kubectl run pod3 --image=nginx -n test-quota
# Error: exceeded quota: quota, requested: pods=1, used: pods=2, limited: pods=2

# 6. Verificar quota
kubectl describe resourcequota quota -n test-quota
```

### Exemplo 3: NamespaceLifecycle protegendo namespaces do sistema

```bash
# Tentar deletar namespace kube-system (BLOQUEADO)
kubectl delete namespace kube-system
# Error: namespaces "kube-system" is forbidden: this namespace may not be deleted

# Tentar criar pod em namespace inexistente (BLOQUEADO)
kubectl run nginx --image=nginx -n nonexistent
# Error: namespaces "nonexistent" not found
```

### Exemplo 4: ServiceAccount injetando token

```bash
# Criar pod sem especificar serviceAccountName
kubectl run nginx --image=nginx

# Verificar que ServiceAccount foi injetado
kubectl get pod nginx -o yaml | grep serviceAccountName
# Output: serviceAccountName: default

# Ver volume montado automaticamente
kubectl get pod nginx -o yaml | grep -A 5 "volumes:"
# Output mostra volume com token do ServiceAccount
```

## 🔍 Admission Webhooks Customizados

### MutatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: MutatingWebhookConfiguration
metadata:
  name: pod-label-injector
webhooks:
- name: inject-labels.example.com
  clientConfig:
    service:
      name: webhook-service
      namespace: webhook-system
      path: "/mutate"
    caBundle: LS0tLS1CRUdJTi...  # CA cert em base64
  rules:
  - operations: ["CREATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Fail  # ou Ignore
```

**O que faz:**
- Quando um pod é criado
- Chama o webhook em `webhook-service.webhook-system/mutate`
- Webhook pode adicionar/modificar labels, annotations, etc.

### ValidatingWebhookConfiguration

```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-policy-validator
webhooks:
- name: validate-pods.example.com
  clientConfig:
    service:
      name: webhook-service
      namespace: webhook-system
      path: "/validate"
    caBundle: LS0tLS1CRUdJTi...
  rules:
  - operations: ["CREATE", "UPDATE"]
    apiGroups: [""]
    apiVersions: ["v1"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  timeoutSeconds: 5
  failurePolicy: Fail
```

**O que faz:**
- Quando um pod é criado/atualizado
- Chama o webhook para validação
- Webhook pode aceitar ou rejeitar

## 🎯 Pod Security Admission (Substitui PSP)

### Níveis de Segurança

| Nível | Descrição | Restrições |
|-------|-----------|------------|
| **privileged** | Sem restrições | Permite tudo |
| **baseline** | Minimamente restritivo | Previne escalação de privilégios conhecidas |
| **restricted** | Altamente restritivo | Hardening de segurança |

### Modos de Aplicação

| Modo | Comportamento |
|------|---------------|
| **enforce** | Rejeita pods que violam | Bloqueia criação |
| **audit** | Registra violações | Permite criação, mas loga |
| **warn** | Mostra warning | Permite criação, mostra aviso |

### Aplicar no Namespace

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/enforce-version: v1.28
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
```

### Exemplo de Pod bloqueado pelo nível "restricted"

```yaml
# Este pod será REJEITADO no namespace com enforce: restricted
apiVersion: v1
kind: Pod
metadata:
  name: privileged-pod
spec:
  containers:
  - name: nginx
    image: nginx
    securityContext:
      privileged: true    # ❌ Não permitido em "restricted"
```

```bash
kubectl apply -f privileged-pod.yaml -n production
# Error: pods "privileged-pod" is forbidden: violates PodSecurity "restricted:latest"
```

### Pod que passa no nível "restricted"

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secure-pod
spec:
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    seccompProfile:
      type: RuntimeDefault
  containers:
  - name: nginx
    image: nginx
    securityContext:
      allowPrivilegeEscalation: false
      capabilities:
        drop:
        - ALL
    resources:
      limits:
        memory: 128Mi
        cpu: 100m
```

## 🛠️ Comandos Úteis

### Ver admission controllers habilitados

```bash
# Via manifest do kube-apiserver
kubectl -n kube-system get pod kube-apiserver-<node> -o yaml | \
  grep -A 1 enable-admission-plugins

# Via processo
ps aux | grep kube-apiserver | grep -o 'enable-admission-plugins=[^[:space:]]*'
```

### Listar webhooks configurados

```bash
# Mutating webhooks
kubectl get mutatingwebhookconfigurations

# Validating webhooks
kubectl get validatingwebhookconfigurations

# Ver detalhes
kubectl describe mutatingwebhookconfigurations <name>
kubectl describe validatingwebhookconfigurations <name>
```

### Ver eventos de admission rejeitadas

```bash
# Ver eventos de erro
kubectl get events -A --field-selector type=Warning | grep -i "forbidden\|quota\|admission"

# Ver eventos de um pod específico
kubectl describe pod <pod-name> | grep -A 10 Events:
```

### Testar Pod Security Admission

```bash
# Verificar labels do namespace
kubectl get namespace <namespace> --show-labels

# Tentar criar pod que viola política
kubectl run test --image=nginx --privileged=true -n <namespace>
```

## 📚 Admission Controllers por Categoria

### Segurança
- **NodeRestriction**: Limita o que kubelets podem fazer
- **PodSecurity**: Valida políticas de segurança de pods
- **ImagePolicyWebhook**: Valida imagens de containers via webhook externo
- **PodSecurityPolicy** (deprecated): Valida PSPs
- **SecurityContextDeny** (deprecated): Nega SecurityContext específicos

### Gerenciamento de Recursos
- **LimitRanger**: Aplica limites padrão
- **ResourceQuota**: Valida quotas de namespace

### Namespaces
- **NamespaceLifecycle**: Protege namespaces do sistema
- **NamespaceExists** (deprecated): Verifica se namespace existe
- **NamespaceAutoProvision** (deprecated): Cria namespace automaticamente

### Storage
- **DefaultStorageClass**: Adiciona storageClass padrão
- **StorageObjectInUseProtection**: Previne deletar PV/PVC em uso
- **PersistentVolumeClaimResize**: Valida resize de PVCs

### Networking
- **DenyEscalatingExec**: Previne exec em pods privileged
- **EventRateLimit**: Limita rate de eventos

### Service Accounts
- **ServiceAccount**: Injeta ServiceAccount em pods
- **DefaultTolerationSeconds**: Adiciona tolerations padrão

### Extensibilidade
- **MutatingAdmissionWebhook**: Webhooks customizados de mutação
- **ValidatingAdmissionWebhook**: Webhooks customizados de validação

## 📖 Recursos para Estudo

### Documentação Oficial
- [Admission Controllers Reference](https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/)
- [Dynamic Admission Control](https://kubernetes.io/docs/reference/access-authn-authz/extensible-admission-controllers/)
- [Pod Security Admission](https://kubernetes.io/docs/concepts/security/pod-security-admission/)
- [Pod Security Standards](https://kubernetes.io/docs/concepts/security/pod-security-standards/)

### Comandos Rápidos de Revisão

```bash
# Ver admission controllers habilitados
ps aux | grep kube-apiserver | grep admission-plugins

# Listar webhooks
kubectl get mutatingwebhookconfigurations
kubectl get validatingwebhookconfigurations

# Ver ResourceQuotas
kubectl get resourcequota -A

# Ver LimitRanges
kubectl get limitrange -A

# Testar Pod Security
kubectl label namespace <ns> pod-security.kubernetes.io/enforce=restricted
```

## 🎯 Pontos Importantes para a Prova CKA

### ✅ Você precisa saber:

1. **O que são Admission Controllers**
   - Plugins que interceptam requisições ao API Server
   - Executam após autenticação/autorização, antes de persistir no etcd

2. **Tipos de Admission Controllers**
   - **Mutating**: modificam objetos (executam PRIMEIRO)
   - **Validating**: validam objetos (executam DEPOIS dos mutating)
   - Ordem: Mutating → Schema Validation → Validating

3. **Por que a ordem importa**
   - Mutating adiciona defaults/modificações primeiro
   - Validating valida o objeto JÁ modificado
   - Exemplo: LimitRanger (mutating) adiciona resources, ResourceQuota (validating) valida totais

3. **Principais Admission Controllers**
   - **LimitRanger**: aplica limits padrão
   - **ResourceQuota**: valida quotas
   - **ServiceAccount**: injeta serviceAccountName
   - **NamespaceLifecycle**: protege namespaces do sistema
   - **NodeRestriction**: limita kubelets
   - **ImagePolicyWebhook**: valida imagens via webhook externo

4. **Como habilitar/desabilitar**
   - Editar `/etc/kubernetes/manifests/kube-apiserver.yaml`
   - Flag: `--enable-admission-plugins=`
   - Flag: `--disable-admission-plugins=`

5. **Pod Security Admission**
   - Níveis: privileged, baseline, restricted
   - Modos: enforce, audit, warn
   - Aplicar via labels no namespace

6. **Troubleshooting**
   - Ver eventos quando pods são rejeitados
   - Entender mensagens de erro de admission

### 🧪 Cenários típicos na prova:

#### Cenário 1: Pod Security Admission

> **"Configure o namespace 'production' para usar Pod Security Admission no nível 'restricted' em modo 'enforce'."**

**Solução:**
```bash
# Método 1: Label no namespace
kubectl label namespace production \
  pod-security.kubernetes.io/enforce=restricted \
  pod-security.kubernetes.io/audit=restricted \
  pod-security.kubernetes.io/warn=restricted

# Método 2: Criar namespace com labels
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: production
  labels:
    pod-security.kubernetes.io/enforce: restricted
    pod-security.kubernetes.io/audit: restricted
    pod-security.kubernetes.io/warn: restricted
EOF

# Verificar
kubectl get namespace production --show-labels
```

#### Cenário 2: Identificar por que pod foi rejeitado

> **"Um desenvolvedor relata que não consegue criar pods no namespace 'dev'. Investigue e resolva o problema."**

**Solução:**
```bash
# 1. Tentar criar pod para ver erro
kubectl run test --image=nginx -n dev

# 2. Ver eventos de admission
kubectl get events -n dev --sort-by='.lastTimestamp' | grep -i "denied\|forbidden"

# 3. Verificar ResourceQuota
kubectl get resourcequota -n dev
kubectl describe resourcequota -n dev

# 4. Verificar LimitRange
kubectl get limitrange -n dev
kubectl describe limitrange -n dev

# 5. Verificar Pod Security labels
kubectl get namespace dev --show-labels

# 6. Se for quota excedida, limpar recursos
kubectl delete pod <old-pods> -n dev

# 7. Se for falta de resources no pod, adicionar:
kubectl run test --image=nginx -n dev \
  --requests='cpu=100m,memory=128Mi' \
  --limits='cpu=200m,memory=256Mi'
```

#### Cenário 3: Entender ordem de execução

> **"Explique por que um pod tem `serviceAccountName: default` mesmo não tendo especificado isso no YAML."**

**Resposta:**
- O **ServiceAccount admission controller** (mutating) injeta automaticamente
- Executa ANTES de validar e persistir no etcd
- Isso acontece para TODOS os pods, a menos que você especifique explicitamente outro ServiceAccount ou `automountServiceAccountToken: false`

```bash
# Verificar admission controllers habilitados
kubectl -n kube-system get pod kube-apiserver-<node> -o yaml | \
  grep enable-admission-plugins

# Procure por "ServiceAccount" na lista
```

> **"Crie um LimitRange no namespace 'dev' que define CPU padrão de 500m e memória padrão de 256Mi para containers que não especificarem resources."**

**Solução:**
```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: LimitRange
metadata:
  name: default-limits
  namespace: dev
spec:
  limits:
  - default:
      cpu: 500m
      memory: 256Mi
    defaultRequest:
      cpu: 250m
      memory: 128Mi
    type: Container
EOF

# Verificar
kubectl describe limitrange default-limits -n dev
```

## 💡 Dicas para a Prova

1. **Admission Controllers são habilitados no kube-apiserver**
   - Editar o manifest estático
   - Reiniciar automático após salvar

2. **LimitRange vs ResourceQuota**
   - LimitRange: limites por pod/container
   - ResourceQuota: limites totais por namespace

3. **Pod Security Admission é o novo padrão**
   - PSP foi removido em v1.25
   - Use labels no namespace

4. **Ver por que pod foi rejeitado**
   ```bash
   kubectl describe pod <name>
   kubectl get events
   ```

5. **Testar antes de aplicar em produção**
   - Use modo `warn` primeiro
   - Depois `audit`
   - Por último `enforce`

---

⬅️ **Anterior**: [kube-scheduler.md](./kube-scheduler.md) | ➡️ **Próximo**: [../Componentes-Worker-Nodes/](../Componentes-Worker-Nodes/)
