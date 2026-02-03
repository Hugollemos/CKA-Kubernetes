# ETCD - Guia Completo

## 1. O que é ETCD?

**ETCD** é um banco de dados distribuído de **chave-valor** (key-value store) altamente disponível e consistente. É a "memória" do Kubernetes, armazenando todo o estado do cluster.

### 1.1 Características Principais

- **Distribuído**: Pode rodar em múltiplos nós
- **Consistente**: Garante que todos os nós têm os mesmos dados
- **Rápido**: Otimizado para leitura e escrita
- **Seguro**: Suporta autenticação e criptografia
- **Confiável**: Persistência de dados em disco

---

## 2. O que o ETCD Armazena no Kubernetes?

O ETCD é o banco de dados central do Kubernetes. Ele armazena:

| Tipo de Dado | Descrição | Exemplo |
| --- | --- | --- |
| **Nodes** | Informações dos nós do cluster | IP, status, capacidade |
| **Pods** | Definição e estado dos pods | Imagem, container, recursos |
| **Services** | Configuração dos serviços | IP, porta, seletor |
| **Deployments** | Configuração de deployments | Réplicas, imagem, estratégia |
| **Configs** | ConfigMaps e variáveis | Configurações da aplicação |
| **Secrets** | Dados sensíveis | Senhas, tokens, certificados |
| **Service Accounts** | Contas de serviço | Identidades de pods |
| **Roles & RoleBindings** | Controle de acesso | Permissões RBAC |
| **Namespaces** | Isolamento lógico | Separação de recursos |
| **Persistent Volumes** | Armazenamento | Volumes persistentes |

### 2.1 Importância

Se o ETCD for perdido, todo o estado do Kubernetes é perdido. Por isso, **backups regulares são essenciais**.

---

## 3. ETCDCTL - Ferramenta CLI

**ETCDCTL** é a ferramenta de linha de comando para interagir com o ETCD.

### 3.1 Versões de API

O ETCDCTL suporta 2 versões da API com comandos diferentes:

### Versão 2 (Legada)

```bash
export ETCDCTL_API=2

```

**Comandos disponíveis:**

- `etcdctl backup` - Faz backup do banco de dados
- `etcdctl cluster-health` - Verifica a saúde do cluster
- `etcdctl mk` - Cria uma chave
- `etcdctl mkdir` - Cria um diretório
- `etcdctl set` - Define/atualiza um valor

### Versão 3 (Atual)

```bash
export ETCDCTL_API=3

```

**Comandos disponíveis:**

- `etcdctl snapshot save` - Cria snapshot/backup do banco
- `etcdctl endpoint health` - Verifica saúde dos endpoints
- `etcdctl get` - Obtém valor de uma chave
- `etcdctl put` - Insere/atualiza um valor

### 3.2 Versão Padrão

- **Padrão**: Versão 2
- **Recomendado**: Usar Versão 3 (mais nova e poderosa)

---

## 4. Configuração do ETCDCTL

### 4.1 Definir a Versão da API

Para usar os comandos da versão 3, é necessário definir a variável de ambiente:

```bash
export ETCDCTL_API=3

```

**Importante**:

- Se não for definido, assume versão 2
- Comandos da v3 não funcionam com v2
- Comandos da v2 não funcionam com v3

### 4.2 Autenticação com Certificados

O ETCD no Kubernetes usa certificados TLS para autenticação. É necessário especificar 3 arquivos de certificado:

```bash
--cacert /etc/kubernetes/pki/etcd/ca.crt       # Certificado da CA
--cert /etc/kubernetes/pki/etcd/server.crt     # Certificado do servidor
--key /etc/kubernetes/pki/etcd/server.key      # Chave privada

```

---

## 5. Exemplos Práticos de Uso

### 5.1 Verificar Saúde do Cluster ETCD

```bash
# Versão 3
kubectl exec etcd-master -n kube-system -- sh -c \
  "ETCDCTL_API=3 etcdctl endpoint health \
   --cacert /etc/kubernetes/pki/etcd/ca.crt \
   --cert /etc/kubernetes/pki/etcd/server.crt \
   --key /etc/kubernetes/pki/etcd/server.key"

```

**Saída esperada:**

```
127.0.0.1:2379 is healthy: successfully committed proposal: took = 5.473ms

```

### 5.2 Listar Todas as Chaves (Comando Completo)

```bash
kubectl exec etcd-master -n kube-system -- sh -c \
  "ETCDCTL_API=3 etcdctl get / \
   --prefix \
   --keys-only \
   --limit=10 \
   --cacert /etc/kubernetes/pki/etcd/ca.crt \
   --cert /etc/kubernetes/pki/etcd/server.crt \
   --key /etc/kubernetes/pki/etcd/server.key"

```

**Opções explicadas:**

- `get /` - Obtém chaves a partir da raiz
- `-prefix` - Busca por prefixo (tudo a partir de /)
- `-keys-only` - Mostra apenas as chaves, não os valores
- `-limit=10` - Limita a 10 resultados
- `-cacert`, `-cert`, `-key` - Arquivos de certificado para autenticação

### 5.3 Obter Valor de uma Chave Específica

```bash
kubectl exec etcd-master -n kube-system -- sh -c \
  "ETCDCTL_API=3 etcdctl get /registry/pods/default/meu-pod \
   --cacert /etc/kubernetes/pki/etcd/ca.crt \
   --cert /etc/kubernetes/pki/etcd/server.crt \
   --key /etc/kubernetes/pki/etcd/server.key"

```

### 5.4 Fazer Backup do ETCD

```bash
kubectl exec etcd-master -n kube-system -- sh -c \
  "ETCDCTL_API=3 etcdctl snapshot save /tmp/etcd-backup.db \
   --cacert /etc/kubernetes/pki/etcd/ca.crt \
   --cert /etc/kubernetes/pki/etcd/server.crt \
   --key /etc/kubernetes/pki/etcd/server.key"

```

### 5.5 Restaurar de um Backup

```bash
# Parar o ETCD primeiro
kubectl delete pod etcd-master -n kube-system

# Restaurar
etcdctl snapshot restore /tmp/etcd-backup.db \
  --data-dir /var/lib/etcd \
  --cacert /etc/kubernetes/pki/etcd/ca.crt \
  --cert /etc/kubernetes/pki/etcd/server.crt \
  --key /etc/kubernetes/pki/etcd/server.key

```

---

## 6. Estrutura de Dados no ETCD

O ETCD armazena dados em uma estrutura hierárquica:

```
/
├── /registry
│   ├── /registry/pods
│   │   ├── /registry/pods/default
│   │   │   └── /registry/pods/default/meu-pod
│   │   └── /registry/pods/kube-system
│   ├── /registry/services
│   └── /registry/secrets
├── /kubernetes.io
│   ├── /kubernetes.io/config
│   └── /kubernetes.io/roles
└── /events
    └── /events/...

```

---

## 7. Boas Práticas

### 7.1 Backups Regulares

```bash
# Agendar backup diário
0 2 * * * /scripts/etcd-backup.sh

```

### 7.2 Monitoramento

- Monitorar saúde do cluster ETCD regularmente
- Alertar se qualquer membro ficar offline
- Verificar espaço em disco disponível

### 7.3 Segurança

- Usar certificados TLS válidos
- Restringir acesso ao ETCD
- Fazer backup criptografado
- Manter o ETCD isolado na rede

### 7.4 Performance

- Ajustar quotas de armazenamento
- Remover dados antigos regularmente
- Usar compactação (`etcdctl compact`)

---

## 8. Troubleshooting Comum

### Erro: Certificado Inválido

```
x509: certificate has expired

```

**Solução**: Renovar certificados do ETCD

### Erro: Versão API incompatível

```
{"level":"warn","ts":"...","caller":"...","msg":"cannot decode member entry"}

```

**Solução**: Verificar se `ETCDCTL_API` está corretamente definido

### Erro: Conexão Recusada

```
Error: context deadline exceeded

```

**Solução**: Verificar se ETCD está rodando e os certificados estão corretos

---

## 9. Resumo de Comandos ETCDCTL v3

```bash
# Configurar API version
export ETCDCTL_API=3

# Verificar saúde
etcdctl endpoint health

# Listar chaves
etcdctl get / --prefix --keys-only

# Obter valor
etcdctl get /registry/pods/default/meu-pod

# Inserir valor
etcdctl put /registry/chave "valor"

# Deletar chave
etcdctl del /registry/chave

# Fazer backup
etcdctl snapshot save backup.db

# Restaurar backup
etcdctl snapshot restore backup.db --data-dir=/var/lib/etcd

```

---

## 10. Conclusão

- **ETCD** é o coração do Kubernetes
- **ETCDCTL** é a ferramenta para gerenciar ETCD
- **Sempre fazer backup** é crítico
- **Monitorar saúde** do cluster ETCD
- **Usar certificados** para autenticação segura

---

---

