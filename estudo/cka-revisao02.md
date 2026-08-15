# Revisão CKA — Parte 2

Obs: no seu "vpc" eu assumi que é **PV/PVC** (PersistentVolume/PersistentVolumeClaim), já que "VPC" (rede de nuvem) não é conteúdo do CKA e é um erro de digitação comum de "PVC". Se você quis dizer outra coisa, me fala que eu ajusto.

---

## 1. Environment Variables no Kubernetes

**Conceito:** Formas de injetar variáveis de ambiente em um container: valor direto, a partir de ConfigMap, a partir de Secret, ou a partir de metadados do próprio Pod (Downward API).

**Valor direto:**
```yaml
containers:
- name: app
  image: nginx
  env:
  - name: APP_MODE
    value: "production"
```

**A partir de um ConfigMap (uma chave específica):**
```yaml
  env:
  - name: APP_MODE
    valueFrom:
      configMapKeyRef:
        name: app-config
        key: mode
```

**Todas as chaves de um ConfigMap de uma vez (`envFrom`):**
```yaml
  envFrom:
  - configMapRef:
      name: app-config
```

**A partir de um Secret:**
```yaml
  env:
  - name: DB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: db-secret
        key: password
```

**Downward API (metadados do próprio pod):**
```yaml
  env:
  - name: POD_NAME
    valueFrom:
      fieldRef:
        fieldPath: metadata.name
  - name: POD_IP
    valueFrom:
      fieldRef:
        fieldPath: status.podIP
```

**Exercício 1:**
1. Crie um ConfigMap com duas chaves (`mode` e `region`).
2. Crie um pod que injete `mode` como env var individual e o `region` via `envFrom`.
3. Confirme dentro do container: `kubectl exec <pod> -- env | grep -E "mode|region"`.

Verificação: variáveis devem aparecer com `kubectl exec <pod> -- printenv`.

---

## 2. Configurando ConfigMaps

**Conceito:** Objeto para armazenar dados de configuração não-sensíveis (pares chave-valor ou arquivos inteiros), desacoplando configuração da imagem do container.

**Criar via linha de comando:**
```bash
# A partir de literais
kubectl create configmap app-config --from-literal=mode=prod --from-literal=region=us-east

# A partir de um arquivo (chave = nome do arquivo)
kubectl create configmap app-config --from-file=app.properties

# A partir de um diretório inteiro
kubectl create configmap app-config --from-file=./config-dir/
```

**Via YAML:**
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  mode: "prod"
  region: "us-east"
  app.properties: |
    key1=value1
    key2=value2
```

**Consumindo como volume (arquivos montados):**
```yaml
volumes:
- name: config-volume
  configMap:
    name: app-config
containers:
- name: app
  volumeMounts:
  - name: config-volume
    mountPath: /etc/config
```
Cada chave do ConfigMap vira um **arquivo** dentro de `/etc/config`.

**Ponto-chave para a prova:** ConfigMaps montados como volume **atualizam automaticamente** quando o ConfigMap muda (com delay, via kubelet sync); já como env var, **não atualizam** sem recriar o pod.

**Exercício 2:**
1. Crie um ConfigMap a partir de um arquivo `app.properties` com 2-3 linhas de configuração.
2. Monte-o como volume em `/etc/config` num pod.
3. Edite o ConfigMap (`kubectl edit configmap app-config`) e depois de ~1 minuto, verifique dentro do pod se o arquivo mudou (`kubectl exec <pod> -- cat /etc/config/app.properties`).

Verificação: conteúdo do arquivo dentro do pod deve refletir a edição, mesmo sem recriar o pod.

---

## 3. Secrets

**Conceito:** Similar ao ConfigMap, mas para dados sensíveis (senhas, tokens, certificados). Valores são armazenados em **base64** (não é criptografia — é apenas encoding, então trate como sensível de qualquer forma).

**Criar via linha de comando:**
```bash
kubectl create secret generic db-secret --from-literal=password=S3cr3t123

# A partir de arquivo
kubectl create secret generic tls-secret --from-file=cert.crt --from-file=cert.key

# Secret do tipo docker-registry
kubectl create secret docker-registry regcred \
  --docker-server=<server> --docker-username=<user> \
  --docker-password=<pass> --docker-email=<email>
```

**Via YAML (valores já em base64):**
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
data:
  password: UzNjcjN0MTIz   # echo -n 'S3cr3t123' | base64
```

**Consumindo:**
```yaml
# Como env var
env:
- name: DB_PASSWORD
  valueFrom:
    secretKeyRef:
      name: db-secret
      key: password

# Como volume (cada chave vira arquivo)
volumes:
- name: secret-volume
  secret:
    secretName: db-secret
```

**Pontos-chave:**
- `kubectl get secret <name> -o jsonpath='{.data.password}' | base64 -d` para decodificar rapidamente.
- Secrets do tipo `kubernetes.io/dockerconfigjson` são usados em `imagePullSecrets` no pod/ServiceAccount.
- Por padrão, Secrets **não são criptografados no etcd** a menos que `EncryptionConfiguration` esteja habilitado no `kube-apiserver`.

**Exercício 3:**
1. Crie um Secret com usuário e senha via `--from-literal`.
2. Monte-o como env vars em um pod.
3. Decodifique o valor de dentro do Secret usando `kubectl get secret ... -o jsonpath` + `base64 -d`, sem entrar no pod.

Verificação: valor decodificado deve bater com o original.

---

## 4. Backup e Restore do etcd

**Conceito:** etcd guarda todo o estado do cluster. Backup = snapshot; restore = recriar o data-dir do etcd a partir do snapshot (e reapontar o etcd estático para o novo diretório).

**Descobrir os certificados/endpoint (olhe o manifest do etcd):**
```bash
cat /etc/kubernetes/manifests/etcd.yaml
# procure: --cert-file, --key-file, --trusted-ca-file, --listen-client-urls
```

**Fazer backup (snapshot):**
```bash
ETCDCTL_API=3 etcdctl snapshot save /opt/backup/etcd-snapshot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key
```

**Verificar o snapshot:**
```bash
ETCDCTL_API=3 etcdctl snapshot status /opt/backup/etcd-snapshot.db --write-out=table
```

**Restaurar (para um novo diretório de dados):**
```bash
ETCDCTL_API=3 etcdctl snapshot restore /opt/backup/etcd-snapshot.db \
  --data-dir=/var/lib/etcd-from-backup
```

**Depois de restaurar, editar `/etc/kubernetes/manifests/etcd.yaml`:**
- Alterar `--data-dir` e o `volumes.hostPath.path` correspondente para apontar para `/var/lib/etcd-from-backup`.
- O kubelet detecta a mudança no manifest estático e recria o pod do etcd automaticamente.

**Pontos-chave que sempre caem:**
- Sempre usar `ETCDCTL_API=3` (a v2 tem sintaxe diferente).
- Erro comum: esquecer de passar os 3 certificados (`--cacert`, `--cert`, `--key`) — sem eles, o comando falha com erro de TLS.
- Depois do restore, verificar `kubectl get pods -n kube-system` para confirmar que o etcd voltou saudável, e `kubectl get nodes`/`kubectl get all -A` para confirmar que o estado restaurado é o esperado.

**Exercício 4:**
1. Crie um namespace de teste com 1-2 pods.
2. Faça snapshot do etcd.
3. Delete o namespace e os pods.
4. Restaure o snapshot para um novo `--data-dir`, edite o manifest do etcd, e confirme que o namespace/pods voltaram.

Verificação: `kubectl get ns` deve mostrar o namespace de volta após restore + `kubectl get pods -n kube-system -w` para ver o etcd reiniciar.

---

## 5. kube-apiserver

**Conceito:** Componente central do control plane — todo mundo (kubectl, kubelet, controllers, scheduler) fala com o cluster através dele. Valida e processa requisições REST, e é o único componente que fala diretamente com o etcd.

**Onde está a configuração (cluster kubeadm):**
```bash
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```

**Flags importantes de olhar/editar:**
```yaml
- --advertise-address=<IP>
- --etcd-servers=https://127.0.0.1:2379
- --enable-admission-plugins=NodeRestriction,...
- --authorization-mode=Node,RBAC
- --service-cluster-ip-range=10.96.0.0/12
- --secure-port=6443
- --client-ca-file=/etc/kubernetes/pki/ca.crt
- --tls-cert-file=/etc/kubernetes/pki/apiserver.crt
- --tls-private-key-file=/etc/kubernetes/pki/apiserver.key
```

**Como editar com segurança:**
- É um **static pod** — edite o YAML diretamente em `/etc/kubernetes/manifests/kube-apiserver.yaml`.
- O kubelet detecta a mudança e recria o pod automaticamente (sem precisar de `kubectl apply`).
- Se o YAML ficar inválido, o apiserver não sobe — cuidado, você pode perder acesso ao `kubectl` até corrigir (edite direto no node via SSH).

**Verificar se está saudável:**
```bash
kubectl get pods -n kube-system | grep apiserver
kubectl get --raw='/healthz'
crictl ps | grep apiserver   # direto no node, se kubectl não responder
```

**Exercício 5:**
1. Liste as flags atuais do `kube-apiserver` no seu cluster.
2. Adicione um admission plugin à lista de `--enable-admission-plugins` editando o manifest estático.
3. Confirme que o pod do apiserver reiniciou e a flag está ativa (`kubectl get pods -n kube-system` + describe).

Verificação: `crictl inspect <container-id>` ou `kubectl describe pod kube-apiserver-<node> -n kube-system` mostrando o comando com a nova flag.

---

## 6. Ingress

**Conceito:** Objeto que expõe rotas HTTP/HTTPS externas para Services dentro do cluster, com regras de host/path. Precisa de um **Ingress Controller** rodando (ex: nginx-ingress) para funcionar — o Ingress sozinho é só a "regra", não faz nada sem o controller.

**Exemplo básico:**
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: app-service
            port:
              number: 80
```

**Path baseado (mesmo host, rotas diferentes):**
```yaml
  rules:
  - host: app.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080
      - path: /
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

**Comandos-chave:**
```bash
kubectl get ingressclass
kubectl get ingress
kubectl describe ingress app-ingress
```

**Pontos-chave:**
- `pathType` pode ser `Exact`, `Prefix` ou `ImplementationSpecific` — a prova gosta de testar `Prefix` vs `Exact`.
- Sem `ingressClassName` (ou annotation antiga `kubernetes.io/ingress.class`), o Ingress pode não ser processado por nenhum controller.
- Um Ingress **default backend** trata requisições que não batem com nenhuma regra.

**Exercício 6:**
1. Crie dois Services simples (ex: duas versões de nginx com respostas diferentes).
2. Crie um Ingress com duas regras de path (`/v1` e `/v2`) apontando para cada Service.
3. Teste com `curl` (via `kubectl port-forward` no controller ou IP do ingress) que cada path retorna o serviço certo.

Verificação: `curl http://<ingress-ip>/v1` e `/v2` retornando respostas diferentes; `kubectl describe ingress` mostrando as regras corretas.

---

## 7. PersistentVolume (PV) e PersistentVolumeClaim (PVC)

**Conceito:** PV é o recurso de armazenamento real no cluster (provisionado por admin ou dinamicamente via StorageClass). PVC é o "pedido" de armazenamento feito por um usuário/pod — o Kubernetes faz o **binding** entre PVC e um PV compatível (tamanho, `accessModes`, `storageClassName`).

**Exemplo de PV (estático, hostPath para lab):**
```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-example
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  hostPath:
    path: /mnt/data
```

**Exemplo de PVC:**
```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-example
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
```

**Usando o PVC em um pod:**
```yaml
volumes:
- name: data
  persistentVolumeClaim:
    claimName: pvc-example
containers:
- name: app
  volumeMounts:
  - name: data
    mountPath: /data
```

**Pontos-chave:**
- `accessModes`: `ReadWriteOnce` (RWO), `ReadOnlyMany` (ROX), `ReadWriteMany` (RWX) — o PVC só faz bind com um PV que tenha um modo compatível.
- `persistentVolumeReclaimPolicy`: `Retain` (mantém dados após deletar PVC), `Delete` (apaga o PV e os dados), `Recycle` (deprecated).
- Se não existir PV compatível e houver uma `StorageClass` com provisionamento dinâmico, o PVC cria o PV automaticamente.
- Status do PVC: `Pending` (sem PV compatível) → `Bound` (ligado a um PV).

**Exercício 7:**
1. Crie um PV `hostPath` de 1Gi com `accessModes: ReadWriteOnce`.
2. Crie um PVC pedindo 1Gi com o mesmo `accessMode`.
3. Confirme o `Bound`: `kubectl get pv,pvc`.
4. Monte o PVC em um pod e escreva um arquivo dentro do `mountPath`; delete o pod e recrie-o, confirmando que o arquivo persiste.

Verificação: `kubectl get pv,pvc` mostrando `STATUS: Bound` e o arquivo sobrevivendo à recriação do pod.

---

## Checklist rápido de comandos para decorar

```bash
kubectl create configmap <name> --from-literal=k=v
kubectl create secret generic <name> --from-literal=k=v
kubectl get secret <name> -o jsonpath='{.data.k}' | base64 -d
ETCDCTL_API=3 etcdctl snapshot save <file> --endpoints=... --cacert=... --cert=... --key=...
ETCDCTL_API=3 etcdctl snapshot restore <file> --data-dir=<dir>
cat /etc/kubernetes/manifests/kube-apiserver.yaml
kubectl get ingress
kubectl get pv,pvc
```

Quer que eu monte agora um simulado misturando os tópicos das duas partes (8 anteriores + esses 7), no estilo de cenário de prova?
