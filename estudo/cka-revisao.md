# Revisão CKA — Tópicos da Semana

Guia direto ao ponto, focado no que cai na prova (killer.sh style). Cada seção tem: conceito, comandos-chave e exercício prático com verificação.

---

## 1. Labels em Node e Pod

**Conceito:** Labels são pares chave=valor usados para selecionar objetos (via `selector`). Node Affinity e `nodeSelector` dependem de labels nos **nodes**. Labels em pods servem para Services, Deployments, NetworkPolicies etc.

**Comandos-chave:**
```bash
# Ver labels de um node
kubectl get nodes --show-labels

# Adicionar label a um node
kubectl label node <node-name> disktype=ssd

# Remover label (sinal de menos no final)
kubectl label node <node-name> disktype-

# Adicionar/alterar label em pod já existente (precisa --overwrite se já existir)
kubectl label pod <pod-name> env=prod --overwrite

# Filtrar por label
kubectl get nodes -l disktype=ssd
kubectl get pods -l env=prod
```

**Exercício 1:**
1. Liste os labels de todos os nodes do seu cluster.
2. Adicione o label `zone=eu-west` a um node worker.
3. Crie um pod simples (nginx) com `nodeSelector: zone: eu-west`.
4. Verifique que o pod foi escalonado nesse node.

Verificação: `kubectl get pod <pod> -o wide` — a coluna NODE deve mostrar o node com o label.

---

## 2. Node Affinity

**Conceito:** Evolução do `nodeSelector`, com regras mais expressivas (`In`, `NotIn`, `Exists`, `Gt`, `Lt`). Dois tipos:
- `requiredDuringSchedulingIgnoredDuringExecution` → regra **obrigatória** (hard rule)
- `preferredDuringSchedulingIgnoredDuringExecution` → regra **preferencial** (soft rule, com `weight`)

O "IgnoredDuringExecution" significa: se o label do node mudar depois que o pod já está rodando, o pod **não é removido**.

**Exemplo (required):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: affinity-pod
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: nginx
    image: nginx
```

**Exemplo (preferred, com peso):**
```yaml
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80
        preference:
          matchExpressions:
          - key: zone
            operator: In
            values:
            - eu-west
```

**Exercício 2:**
1. Crie um pod que só possa rodar em nodes com label `disktype=ssd` usando `nodeAffinity` obrigatória.
2. Tente criar em um cluster sem nenhum node com esse label — observe o status do pod (`Pending`).
3. Adicione o label ao node e veja o pod ser escalonado (dica: pode precisar deletar/recriar o pod, pois o scheduler só decide na criação).

Verificação: `kubectl describe pod <pod>` — evento de scheduling deve mostrar sucesso ou falha (`FailedScheduling`).

---

## 3. Static Pods

**Conceito:** Pods gerenciados diretamente pelo **kubelet** de um node específico, sem passar pelo API server/scheduler. Definidos por manifestos YAML em um diretório monitorado pelo kubelet (`staticPodPath`).

**Onde configurar:**
```bash
# Ver o config do kubelet (procure staticPodPath)
cat /var/lib/kubelet/config.yaml | grep staticPodPath

# Diretório padrão comum
ls /etc/kubernetes/manifests/
```

**Pontos-chave para a prova:**
- O nome do pod no API server aparece com sufixo do node: `<pod-name>-<node-name>`.
- Para criar: basta colocar/copiar o YAML no `staticPodPath` do node correto (via SSH ou `--rootfs`).
- Para remover: apagar o arquivo do diretório — o kubelet remove o pod automaticamente.
- O control plane (`kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager`) geralmente **são** static pods no diretório `/etc/kubernetes/manifests/`.
- Um "mirror pod" é a representação read-only no API server; você não consegue editar via `kubectl edit`.

**Exercício 3:**
1. Descubra o `staticPodPath` do kubelet em um node (pode ser via `ssh` para o node no exame).
2. Crie um manifesto de static pod (nginx) e coloque nesse diretório.
3. Confirme com `kubectl get pods -o wide` que o pod aparece com sufixo do node.
4. Remova o arquivo e confirme que o pod some.

Verificação: `kubectl get pods -A | grep <node-name>` e `crictl ps` diretamente no node.

---

## 4. Priority Classes

**Conceito:** Definem prioridade de agendamento. Pods com prioridade mais alta são agendados antes e podem causar **preemption** (expulsão) de pods de menor prioridade quando não há recursos.

**Exemplo:**
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000000
globalDefault: false
description: "Prioridade alta para workloads críticos"
```

Uso no pod:
```yaml
spec:
  priorityClassName: high-priority
  containers:
  - name: nginx
    image: nginx
```

**Pontos-chave:**
- `value`: quanto maior, mais prioridade (int32, até ~1 bilhão para classes de usuário; valores acima de 1 bilhão são reservados pelo sistema).
- `globalDefault: true` → aplicado a pods sem `priorityClassName` explícito (só pode existir **uma** classe com isso).
- `preemptionPolicy: Never` impede que o pod dessa classe faça preemption de outros (fica na fila esperando recurso).

**Exercício 4:**
1. Crie duas PriorityClasses: `low-priority` (value: 1000) e `high-priority` (value: 1000000).
2. Preencha um node pequeno (ou namespace com resource quota) com pods `low-priority` até não sobrar recurso.
3. Crie um pod `high-priority` pedindo recursos — observe um pod de baixa prioridade sendo removido (evento `Preempted`).

Verificação: `kubectl get priorityclasses` e `kubectl describe pod <pod-preemptado>` (evento de preemption).

---

## 5. Multiple Schedulers

**Conceito:** É possível rodar mais de um scheduler no cluster. Cada pod escolhe qual scheduler o agenda via `schedulerName`. Se não especificado, usa `default-scheduler`.

**Como funciona na prática (exame):**
1. O scheduler customizado normalmente já roda como Deployment/static pod no cluster de teste (ou você o cria a partir do binário `kube-scheduler`).
2. No manifesto do pod, você define:
```yaml
spec:
  schedulerName: my-scheduler
  containers:
  - name: nginx
    image: nginx
```

**Pontos-chave:**
- Se o scheduler nomeado não existir/não estiver rodando, o pod fica `Pending` para sempre (nenhum scheduler pega ele).
- Para verificar qual scheduler agendou um pod: veja os **Events** do pod (`Successfully assigned ... by <scheduler-name>`).

**Exercício 5:**
1. Verifique se existe um scheduler customizado no cluster (`kubectl get pods -n kube-system | grep scheduler`).
2. Crie um pod com `schedulerName: my-scheduler` apontando para esse scheduler customizado.
3. Confirme nos eventos do pod que ele foi agendado pelo scheduler correto.
4. Crie outro pod com `schedulerName` de um scheduler inexistente e observe que fica `Pending`.

Verificação: `kubectl get events --field-selector involvedObject.name=<pod>` ou `kubectl describe pod <pod>`.

---

## 6. Admission Controllers

**Conceito:** Plugins que interceptam requisições ao `kube-apiserver` **depois** da autenticação/autorização e **antes** de persistir no etcd. Podem **modificar** (mutating) ou **validar/rejeitar** (validating) o objeto.

**Onde configurar:**
```bash
# Flag no manifesto do kube-apiserver (static pod)
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep enable-admission-plugins
```
```yaml
- --enable-admission-plugins=NodeRestriction,NamespaceLifecycle,LimitRanger,ResourceQuota
- --disable-admission-plugins=...
```

**Admission Controllers comuns na prova:**
- `NamespaceLifecycle` — impede criar objetos em namespaces sendo deletados/inexistentes.
- `LimitRanger` — aplica `LimitRange` (defaults de request/limit).
- `ResourceQuota` — aplica `ResourceQuota` do namespace.
- `NodeRestriction` — limita o que um kubelet pode modificar (só seu próprio node/pods).
- `ServiceAccount` — automatiza criação/anexação de ServiceAccounts.
- `DefaultStorageClass` — atribui StorageClass default a PVCs sem uma.
- `MutatingAdmissionWebhook` / `ValidatingAdmissionWebhook` — permitem plugar webhooks customizados.

**Exercício 6:**
1. Veja quais admission controllers estão habilitados no seu cluster.
2. Habilite o `NamespaceLifecycle` (se não estiver) editando o static pod manifest do kube-apiserver.
3. Teste criar um objeto num namespace que está `Terminating` e observe a rejeição.

Verificação: `kubectl get events -A | grep -i admission` e o próprio erro retornado pelo `kubectl apply`.

---

## 7. Validating e Mutating Admission Webhooks

**Conceito:** Formas de estender Admission Controllers via webhooks HTTP externos (você registra um `MutatingWebhookConfiguration` ou `ValidatingWebhookConfiguration` que aponta para um Service/servidor externo).

**Ordem de execução importante (cai na prova):**
1. Autenticação → Autorização
2. **Mutating Admission Controllers** (built-in) → **Mutating Webhooks**
3. Validação de schema do objeto
4. **Validating Admission Controllers** (built-in) → **Validating Webhooks**
5. Persistência no etcd

Ou seja: **mutating sempre roda antes de validating** — porque um webhook mutante pode alterar o objeto, e você precisa validar a versão final, já modificada.

**Exemplo simplificado de ValidatingWebhookConfiguration:**
```yaml
apiVersion: admissionregistration.k8s.io/v1
kind: ValidatingWebhookConfiguration
metadata:
  name: pod-policy.example.com
webhooks:
- name: pod-policy.example.com
  clientConfig:
    service:
      name: webhook-service
      namespace: default
      path: "/validate"
    caBundle: <base64-ca-cert>
  rules:
  - apiGroups: [""]
    apiVersions: ["v1"]
    operations: ["CREATE"]
    resources: ["pods"]
  admissionReviewVersions: ["v1"]
  sideEffects: None
  failurePolicy: Fail
```

**Pontos-chave:**
- `failurePolicy: Fail` → se o webhook não responder, a requisição é **rejeitada** (mais seguro).
- `failurePolicy: Ignore` → se falhar, a requisição **passa** (menos seguro, mas evita travar o cluster).
- `sideEffects` deve estar declarado (`None`, `NoneOnDryRun`, etc.) — necessário para dry-run funcionar.
- Ambos precisam de um servidor HTTPS válido (certificado confiável via `caBundle`) rodando o endpoint do webhook.

**Exercício 7 (mais conceitual, difícil de montar do zero no exame):**
1. Explique com suas palavras a diferença entre um Mutating e um Validating webhook, com um exemplo de cada (ex: mutating injeta um sidecar automaticamente; validating rejeita pods sem label `owner`).
2. Liste os objetos: `kubectl get validatingwebhookconfigurations` e `kubectl get mutatingwebhookconfigurations`.
3. Se houver algum no cluster de prática, use `kubectl describe` para ver as `rules` e o `failurePolicy` configurados.

Verificação: entendimento conceitual + saber ler a config existente rapidamente (no exame raramente pedem para criar o servidor do webhook do zero, mas pedem para **interpretar/ajustar** configs existentes).

---

## 8. Pods com Múltiplos Containers

**Conceito:** Um Pod pode ter mais de um container compartilhando rede (mesmo IP/localhost) e, opcionalmente, volumes. Padrões mais cobrados:

- **Sidecar:** container auxiliar que roda ao lado do principal (ex: coletor de logs, proxy). Desde 1.28+, sidecars podem ser definidos como `initContainers` com `restartPolicy: Always`.
- **Init Container:** roda **antes** dos containers principais, sequencialmente, e precisa terminar com sucesso para o próximo (ou o pod) iniciar.
- **Ambassador:** proxy que representa a conexão com serviços externos.
- **Adapter:** padroniza/normaliza a saída (ex: logs) do container principal para um formato comum.

**Exemplo multi-container (sidecar clássico compartilhando volume):**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-container-pod
spec:
  volumes:
  - name: shared-logs
    emptyDir: {}
  containers:
  - name: app
    image: busybox
    command: ["sh", "-c", "while true; do echo $(date) >> /var/log/app.log; sleep 5; done"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
  - name: log-sidecar
    image: busybox
    command: ["sh", "-c", "tail -f /var/log/app.log"]
    volumeMounts:
    - name: shared-logs
      mountPath: /var/log
```

**Exemplo de init container (sidecar nativo desde 1.28+):**
```yaml
spec:
  initContainers:
  - name: sidecar
    image: busybox
    restartPolicy: Always   # isso o torna um "sidecar container" que roda durante toda a vida do pod
    command: ["sh", "-c", "while true; do sleep 30; done"]
  containers:
  - name: main-app
    image: nginx
```

**Exercício 8:**
1. Crie um pod com um container principal escrevendo logs em um arquivo, e um segundo container fazendo `tail -f` desse arquivo, compartilhando um `emptyDir`.
2. Verifique os logs de cada container separadamente: `kubectl logs <pod> -c app` e `kubectl logs <pod> -c log-sidecar`.
3. Crie um pod com um `initContainer` tradicional (sem `restartPolicy`) que falha propositalmente (`exit 1`) e observe que o container principal nunca inicia.

Verificação: `kubectl describe pod <pod>` (seção Init Containers mostrará `Error`/`CrashLoopBackOff` do init container).

---

## Checklist rápido de comandos para decorar

```bash
kubectl label node <node> key=value
kubectl label node <node> key-
kubectl get nodes -l key=value
kubectl get pods -o wide
kubectl describe pod <pod>
kubectl get priorityclasses
kubectl get validatingwebhookconfigurations
kubectl get mutatingwebhookconfigurations
kubectl logs <pod> -c <container>
cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep admission-plugins
```

Quer que eu monte um mini-simulado com 5-6 questões estilo prova (com cenário e tarefa, sem gabarito na tela) misturando esses tópicos, pra você testar antes de ver a explicação?
