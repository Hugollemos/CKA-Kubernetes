# TLS no Kubernetes

## O que é TLS?

**TLS (Transport Layer Security)** é o protocolo que garante comunicação criptografada e autenticada entre componentes. No Kubernetes, toda comunicação entre componentes é protegida com TLS.

---

## Conceitos Fundamentais de Criptografia

### Criptografia Simétrica

Usa a **mesma chave** para criptografar e descriptografar. Problema: a chave precisa ser trocada de forma segura entre as partes.

### Criptografia Assimétrica (Par de Chaves)

Usa um par de chaves:
- **Chave Privada** — mantida em segredo (nunca compartilhada)
- **Chave Pública** — pode ser compartilhada com qualquer um

O que é criptografado com a chave pública só pode ser descriptografado com a chave privada correspondente.

### Certificate Authority (CA)

Uma **Autoridade Certificadora** é uma entidade confiável que assina certificados, garantindo sua autenticidade. Exemplos: DigiCert, GlobalSign, Comodo. No Kubernetes, usamos uma CA própria do cluster.

### Convenção de Nomes de Arquivos

| Tipo | Extensão comum |
|------|----------------|
| Chave privada | `.key`, `-key.pem` |
| Certificado público (chave pública) | `.crt`, `.pem` |

---

## TLS no Kubernetes — Visão Geral

Cada componente precisa de **dois tipos** de certificados:
- **Certificados de Servidor** — para autenticar o componente como servidor
- **Certificados de Cliente** — para autenticar o componente como cliente de outro

```
CA do Cluster
├── kube-apiserver (server cert + client cert para etcd)
├── etcd (server cert)
├── kubelet (server cert + client cert)
├── kube-scheduler (client cert)
├── kube-controller-manager (client cert)
├── kube-proxy (client cert)
└── admin (client cert)
```

---

## Gerando Certificados com OpenSSL

### 1. Criar a CA do Cluster

```bash
# Gerar chave privada da CA
openssl genrsa -out ca.key 2048

# Gerar CSR (Certificate Signing Request)
openssl req -new -key ca.key -subj "/CN=KUBERNETES-CA" -out ca.csr

# Auto-assinar o certificado da CA
openssl x509 -req -in ca.csr -signkey ca.key -out ca.crt
```

### 2. Criar Certificado de Admin (cliente)

```bash
# Gerar chave privada
openssl genrsa -out admin.key 2048

# Gerar CSR (com grupo system:masters para permissões admin)
openssl req -new -key admin.key \
  -subj "/CN=kube-admin/O=system:masters" \
  -out admin.csr

# Assinar com a CA do cluster
openssl x509 -req -in admin.csr \
  -CA ca.crt -CAkey ca.key \
  -out admin.crt
```

### 3. Certificado do kube-apiserver (servidor)

O apiserver tem vários nomes alternativos (SANs):

```bash
# Arquivo openssl.cnf para SANs
[req]
req_extensions = v3_req
[v3_req]
subjectAltName = @alt_names
[alt_names]
DNS.1 = kubernetes
DNS.2 = kubernetes.default
DNS.3 = kubernetes.default.svc
DNS.4 = kubernetes.default.svc.cluster.local
IP.1 = 10.96.0.1       # ClusterIP do apiserver
IP.2 = 192.168.1.10    # IP do nó master

# Gerar com as SANs
openssl req -new -key apiserver.key \
  -subj "/CN=kube-apiserver" \
  -out apiserver.csr \
  -config openssl.cnf

openssl x509 -req -in apiserver.csr \
  -CA ca.crt -CAkey ca.key \
  -CAcreateserial \
  -out apiserver.crt \
  -extensions v3_req \
  -extfile openssl.cnf
```

---

## Visualizar Detalhes de um Certificado

```bash
# Ver informações completas de um certificado
openssl x509 -in /etc/kubernetes/pki/apiserver.crt -text -noout

# Campos importantes:
# - Subject: CN (Common Name)
# - Issuer: quem assinou
# - Validity: datas de validade
# - Subject Alternative Names (SANs)
```

---

## Certificate API — Gerenciar Certificados via Kubernetes

O Kubernetes tem uma API built-in para gerenciar requisições de certificados.

### Fluxo completo

```bash
# 1. Usuário gera sua chave privada
openssl genrsa -out jane.key 2048

# 2. Gera um CSR
openssl req -new -key jane.key -subj "/CN=jane" -out jane.csr

# 3. Codifica o CSR em base64
cat jane.csr | base64 -w 0
```

```yaml
# 4. Cria o objeto CertificateSigningRequest
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: jane
spec:
  groups:
  - system:authenticated
  usages:
  - digital signature
  - key encipherment
  - server auth
  signerName: kubernetes.io/kube-apiserver-client
  request: <base64-do-csr-aqui>
```

```bash
# 5. Admin aprova a requisição
kubectl get csr                    # listar CSRs
kubectl certificate approve jane  # aprovar

# 6. Extrair o certificado aprovado
kubectl get csr jane -o yaml
# O certificado está em .status.certificate (base64)

# 7. Decodificar
echo "<certificado-base64>" | base64 --decode > jane.crt
```

### O Controller Manager assina os certificados

O `kube-controller-manager` é responsável por assinar os certificados. Ele precisa das chaves da CA:

```bash
# Parâmetros do controller-manager
--cluster-signing-cert-file=/etc/kubernetes/pki/ca.crt
--cluster-signing-key-file=/etc/kubernetes/pki/ca.key
```

---

## Inspecionar Logs de Certificados

```bash
# Cluster com systemd (bare metal)
journalctl -u etcd.service -l
journalctl -u kube-apiserver -l

# Cluster com kubeadm (pods)
kubectl logs etcd-master -n kube-system
kubectl logs kube-apiserver-master -n kube-system

# Containers diretamente (se kubectl não funcionar)
docker ps -a
docker logs <container-id>

# crictl (alternativa ao docker)
crictl ps -a
crictl logs <container-id>
```

---

## Localização dos Certificados (kubeadm)

```
/etc/kubernetes/pki/
├── ca.crt                    # CA do cluster
├── ca.key                    # Chave privada da CA
├── apiserver.crt             # Certificado do apiserver (servidor)
├── apiserver.key             # Chave privada do apiserver
├── apiserver-kubelet-client.crt  # apiserver como cliente do kubelet
├── apiserver-etcd-client.crt     # apiserver como cliente do etcd
├── etcd/
│   ├── ca.crt                # CA do etcd
│   ├── server.crt            # Certificado do etcd
│   └── peer.crt              # Certificado de peer etcd
└── front-proxy-ca.crt        # CA do front proxy
```

---

## Referências

- https://kubernetes.io/docs/setup/best-practices/certificates/
- https://kubernetes.io/docs/reference/access-authn-authz/certificate-signing-requests/
- https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/
