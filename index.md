---

dicas

Kubernetes v1.34. 

2 horas de prova

utilias alias alias kubctl = k

utlizar a documentação do kubernetes 

estudar vim 

validar questoes

17 questões

ler as instruções pre-prova

kubectl create deploy -h 

fazer backup dos meus arquivos poque na ovou poder voltar neles 

kubectl get pod nginx -o yaml > pod-backup.yaml

kubectl explain pods —recursive

echo “kubectl explain pods —recursive” > result-command.txt

https://kubernetes.io/docs/home/

 Use muito "dry-run=client -o yaml > recurso.yaml". 

dicas acima

---

---

# Domains & Competencies

![CKABluePrintMindMap.png](attachment:62c6ca7b-2ee4-4d1a-91a7-307e6e41bdbd:CKABluePrintMindMap.png)

https://github.com/julianunesp/devops-templates/blob/main/HowToAceCKA/CKABluePrintMindMap.png

https://github.com/cleversonbrsp/preparatorio-cka-ptbr

## Storage 10%

- Implement storage classes and dynamic volume provisioning
- Configure volume types, access modes and reclaim policies
- Manage persistent volumes and persistent volume claims

## Troubleshooting 30%

- Troubleshoot clusters and nodes
- Troubleshoot cluster components
- Monitor cluster and application resource usage
- Manage and evaluate container output streams
- Troubleshoot services and networking

## Workloads & Scheduling 15%

- Understand application deployments and how to perform rolling update and rollbacks
- Use ConfigMaps and Secrets to configure applications
- Configure workload autoscaling
- Understand the primitives used to create robust, self-healing, application deployments
- Configure Pod admission and scheduling (limits, node affinity, etc.)

## Cluster Architecture, Installation & Configuration 25%

- Manage role based access control (RBAC)
- Prepare underlying infrastructure for installing a Kubernetes cluster
- Create and manage Kubernetes clusters using kubeadm
- Manage the lifecycle of Kubernetes clusters
- Implement and configure a highly-available control plane
- Use Helm and Kustomize to install cluster components
- Understand extension interfaces (CNI, CSI, CRI, etc.)
- Understand CRDs, install and configure operators

## Services & Networking 20%

- Understand connectivity between Pods
- Define and enforce Network Policies
- Use ClusterIP, NodePort, LoadBalancer service types and endpoints
- Use the Gateway API to manage Ingress traffic
- Know how to use Ingress controllers and Ingress resources
- Understand and use CoreDNS

https://devopsmind.com.br/kubernetes-pt-br/kubernetes-etcd-revisao-prova-cka/

https://devopsmind.com.br/kubernetes-pt-br/kubernetes-configmaps-secrets-cka/

https://devopsmind.com.br/kubernetes-pt-br/storage-no-kubernetes-prova-cka/

https://devopsmind.com.br/kubernetes-pt-br/monitorando-clusters-kubernetes/

https://medium.com/@italocavalcantechagas/certifica%C3%A7%C3%A3o-kubernetes-cka-como-passei-na-certifica%C3%A7%C3%A3o-come%C3%A7ando-do-zero-ecf3cce5f5ce

https://www.pluralsight.com/paths/certified-kubernetes-administrator

https://fabiobo2005-18669.medium.com/certifica%C3%A7%C3%A3o-cka-dicas-em-pt-br-d9c3461a1e59

https://www.linkedin.com/posts/thiago-g-50938b164_free-kubernetes-certification-exam-practice-activity-7358302073899102208-KZlQ/?originalSubdomain=pthttps://github.com/julianunesp/devops-templates/blob/main/HowToAceCKA/CKABluePrintMindMap.png

https://ichi.pro/pt/como-passar-no-exame-certified-kubernetes-administrator-cka-173731186643896

https://www.youtube.com/watch?v=Fr9GqFwl6NM

https://www.youtube.com/watch?v=4THV5o6ntIE&list=PLTCdA9kpJ7okpLcuBqSJMaJaiaay0mg-F

https://www.youtube.com/watch?v=dlp4YuJ6jwk&list=PLi0QOhIwpoFqFimUI-kpaPhAvF7K1TPJ-

https://www.youtube.com/watch?v=hQXCkTU6Xbw&list=PLvOcEsRqg0tKO1znOiZN5fhdAY2DCTvFY

https://www.youtube.com/watch?v=Lxulhs0Z1r4&list=PLpbwBK0ptsswtM6ihzE6ABGpA-b0NaHXU

https://www.youtube.com/watch?v=rNgTxL5Iqxo&list=PLFAJ6I3pCvzouqwyaIGuu2tzmY_bz8et9

https://github.com/julianunesp/devops-templates/blob/main/HowToAceCKA/CKABluePrintMindMap.png

## Componentes do Control Plane

### kube-scheduler

O **kube-scheduler** é o componente responsável por agendar pods nos nós do cluster. Ele identifica o nó mais adequado para executar um pod com base em:

- Requisitos de recursos dos containers (CPU, memória)
- Capacidade disponível nos nós de trabalho
- Políticas e restrições configuradas (taints, tolerations, node affinity, pod affinity/anti-affinity)
- Prioridades e preempção de pods

### kube-controller-manager

O **kube-controller-manager** executa diversos controladores que regulam o estado do cluster. Exemplos importantes:

- **Node Controller**: Responsável por monitorar a saúde dos nós, integrar novos nós ao cluster e lidar com situações onde os nós ficam indisponíveis ou são destruídos
- **Replication Controller**: Garante que o número desejado de réplicas de pods esteja em execução o tempo todo dentro de um ReplicationController (nota: ReplicaSets são mais comumente usados hoje)
- Outros controladores incluem: Deployment Controller, StatefulSet Controller, Job Controller, entre outros

### kube-apiserver

O **kube-apiserver** é o principal componente de gerenciamento do Kubernetes. Ele:

- Expõe a API do Kubernetes
- Processa e valida requisições REST
- Serve como frontend para o control plane
- É o único componente que interage diretamente com o etcd

## Componentes dos Nós (Worker Nodes)

### kubelet

O **kubelet** é um agente que é executado em cada nó do cluster. Suas responsabilidades incluem:

- Garantir que os containers descritos nos PodSpecs estejam rodando e saudáveis
- Comunicar-se com o kube-apiserver
- Monitorar pods e reportar status
- Executar probes (liveness, readiness, startup)

### kube-proxy

O **kube-proxy** mantém as regras de rede nos nós. Ele:

- Implementa parte do conceito de Service do Kubernetes
- Garante que as regras de rede necessárias estejam configuradas para permitir comunicação entre pods
- Gerencia regras de iptables/ipvs para roteamento de tráfego
- Possibilita a conectividade de rede entre containers em diferentes nós

---

---

![image.png](attachment:f83e1c5a-58fe-468d-bf78-4ff387b97a36:image.png)

# Docker & ContainerD - Guia de Conceitos

## Introdução

Docker e ContainerD são tecnologias fundamentais para containerização. Entender a relação entre elas e os componentes envolvidos é essencial para trabalhar com containers modernos.

---

## 1. O que é Containerização?

Containerização é uma forma de empacotar aplicações e suas dependências em unidades isoladas chamadas **containers**. Esses containers podem rodar de forma consistente em qualquer máquina que tenha o runtime apropriado.

---

## 2. Componentes Principais

### 2.1 OCI (Open Container Initiative)

- **Definição**: Organização de padrões abertos para containers
- **Propósito**: Estabelecer padrões comuns para formatos de imagem e runtime de containers
- **Importância**: Garante que diferentes ferramentas e plataformas possam trabalhar juntas sem estar presas a um único fornecedor

### 2.2 ContainerD

- **Definição**: Container runtime de nível mais alto (high-level)
- **Função**: Gerencia o ciclo de vida completo dos containers (criar, executar, parar, deletar)
- **Características**:
    - Substitui o antigo Docker Engine
    - Mais leve e focado
    - Independente do Docker
    - Aderente aos padrões OCI

### 2.3 runc

- **Definição**: Container runtime de nível mais baixo (low-level)
- **Função**: É o executor real que cria e executa containers
- **Relação**: ContainerD o utiliza por baixo dos panos
- **Padrão**: Implementação padrão do OCI Runtime Specification

---

## 3. Ferramentas CLI (Interface de Linha de Comando)

### 3.1 Docker CLI (docker)

- **Descrição**: Ferramenta principal e amigável para interagir com Docker
- **Uso**: `docker run`, `docker ps`, `docker build`, etc.
- **Públicoalvo**: Desenvolvedores, DevOps
- **Características**: Interface mais simples e documentada

### 3.2 nerdctl

- **Descrição**: CLI compatível com Docker, mas para ContainerD
- **Uso**: Sintaxe praticamente idêntica ao Docker (`nerdctl run`, `nerdctl ps`)
- **Vantagem**: Trabalha diretamente com ContainerD
- **Caso de uso**: Quando você prefere trabalhar com ContainerD em vez de Docker

### 3.3 crictl

- **Descrição**: CLI para Container Runtime Interface (CRI)
- **Uso**: Principalmente em ambientes Kubernetes
- **Função**: Debugar e gerenciar containers através da interface CRI
- **Público-alvo**: Administradores de Kubernetes, engenheiros de SRE

### 3.4 ctr

- **Descrição**: CLI de nível baixo para ContainerD
- **Uso**: Gerenciamento direto de containers no ContainerD
- **Características**: Mais técnica e detalhada que nerdctl
- **Público-alvo**: Engenheiros de infraestrutura

---

## 4. Arquitetura e Relacionamentos

```
┌─────────────────────────────────────────────────────┐
│                     Docker CLI                      │
│                  (docker commands)                   │
└────────────────────────┬────────────────────────────┘
                         │
┌────────────────────────▼────────────────────────────┐
│              Docker Daemon (dockerd)                │
│                                                      │
│  ┌──────────────────────────────────────────────┐  │
│  │         ContainerD (containerd)              │  │
│  │    (gerencia containers de alto nível)       │  │
│  │                                               │  │
│  │  ┌──────────────────────────────────────┐   │  │
│  │  │  runc (OCI Runtime)                 │   │  │
│  │  │  (executa containers de baixo nível)│   │  │
│  │  └──────────────────────────────────────┘   │  │
│  └──────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
         │
         └──────► Kernel Linux (namespaces, cgroups)

```

---

## 5. Evolução: Docker → ContainerD

### 5.1 O que era o DockerShim?

- **Descrição**: Componente intermediário entre Docker e Kubernetes
- **Função**: Permitia que Kubernetes gerenciasse containers através do Docker
- **Problema**: Adicionava uma camada desnecessária de abstração
- **Status**: Descontinuado a partir de Kubernetes 1.24 (2022)

### 5.2 Por que essa mudança?

- **Docker é pesado**: Inclui muitas funcionalidades não essenciais para containers
- **ContainerD é leve**: Focado especificamente em container runtime
- **Simplificação**: Remover dockershim simplificou a arquitetura
- **Melhor performance**: Reduz overhead de camadas intermediárias

---

## 6. Outras Alternativas: rkt

### 6.1 rkt

- **Descrição**: Container runtime alternativo ao Docker/ContainerD
- **Status**: Descontinuado (2019)
- **Características**: Focava em segurança e padrões abertos
- **Legado**: Influenciou o desenvolvimento de padrões OCI

---

## 7. Container Runtime Interface (CRI)

### 7.1 O que é CRI?

- **Definição**: Interface padrão entre Kubernetes e container runtimes
- **Propósito**: Permitir que Kubernetes funcione com qualquer runtime que implemente CRI
- **Runtimes compatíveis**: ContainerD, cri-o, Docker (via dockershim)

### 7.2 Vantagens

- **Flexibilidade**: Escolher o runtime sem mudar o orquestrador
- **Padronização**: Interface consistente
- **Desacoplamento**: Kubernetes não depende de um runtime específico

---

## 8. Resumo das Ferramentas

| Ferramenta | Nível | Função | Público |
| --- | --- | --- | --- |
| **docker** | Alto | CLI amigável para Docker | Desenvolvedores |
| **nerdctl** | Alto | CLI Docker-compatível para ContainerD | DevOps/Engenheiros |
| **ctr** | Baixo | CLI técnica para ContainerD | Infraestrutura |
| **crictl** | Médio | Gerenciar containers via CRI | Kubernetes/SRE |
| **runc** | Muito Baixo | Runtime OCI executor | Sistema |

---

## 9. Fluxo Prático

### Exemplo com Docker:

```
$ docker run -d ubuntu:latest
    ↓
Docker CLI → Docker Daemon → ContainerD → runc → Container

```

### Exemplo com nerdctl:

```
$ nerdctl run -d ubuntu:latest
    ↓
nerdctl → ContainerD → runc → Container

```

### Exemplo com Kubernetes:

```
kubelet → CRI Interface → ContainerD → runc → Container

```

---

## 10. Conclusão

- **OCI** estabelece os padrões
- **ContainerD** gerencia containers de forma eficiente
- **runc** executa os containers seguindo padrões OCI
- **Docker** continua sendo uma ferramenta popularpara desenvolvimento
- **Kubernetes** agora trabalha diretamente com ContainerD via CRI

A arquitetura moderna desacopla as responsabilidades, permitindo maior flexibilidade e eficiência.

---

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

# Kube-API-Server - Guia Completo

## 1. O que é Kube-API-Server?

**Kube-API-Server** é o componente central do Kubernetes que funciona como a **porta de entrada principal** para toda comunicação com o cluster. É o único componente que interage diretamente com o ETCD.

### 1.1 Analogia

Pense no Kube-API-Server como um **porteiro de um banco**:

- Verifica sua identidade (autenticação)
- Valida se você tem permissão (autorização)
- Processa suas solicitações
- Guarda todos os dados (ETCD)

---

## 2. Responsabilidades Principais

### 2.1 Comunicação com o Cluster

- **Único ponto de contato** para interagir com o Kubernetes
- Recebe requisições HTTP/HTTPS de diversos clientes
- Responde com informações do cluster

### 2.2 Autenticação

- Verifica **quem você é**
- Valida credenciais (tokens, certificados, etc.)
- Rejeita requisições não autenticadas

### 2.3 Autorização

- Verifica **o que você pode fazer**
- Implementa políticas de acesso (RBAC)
- Controla permissões por usuário/serviço

### 2.4 Validação

- Valida formato das requisições
- Verifica se os dados estão corretos
- Rejeita requisições inválidas antes de processar

### 2.5 Recuperação e Atualização de Dados

- **Única interface** com o ETCD
- Recupera dados quando solicitado
- Atualiza dados e persiste no ETCD
- Garante consistência

---

## 3. Fluxo de uma Requisição

```
┌────────────────┐
│   Cliente      │
│  (kubectl)     │
└────────┬────────┘
         │ Requisição HTTPS
         ▼
┌───────────────────────────────────────────┐
│      Kube-API-Server                      │
│                                           │
│  1️⃣  Autenticação                        │
│      ├─ Verifica credenciais              │
│      └─ Identifica o usuário/serviço      │
│                                           │
│  2️⃣  Autorização                         │
│      ├─ Verifica permissões (RBAC)        │
│      └─ Permite/nega acesso               │
│                                           │
│  3️⃣  Validação                           │
│      ├─ Valida formato JSON/YAML          │
│      └─ Valida regras de negócio          │
│                                           │
│  4️⃣  Processamento                       │
│      ├─ Processa a solicitação            │
│      └─ Comunica com ETCD                 │
│                                           │
│  5️⃣  Resposta                            │
│      ├─ Retorna dados ou confirmação      │
│      └─ Retorna erros (se houver)         │
└───────────┬─────────────────────────────────┘
            │
            ▼
┌───────────────────────────────────────────┐
│      ETCD (Banco de Dados)                │
│  (Armazena estado do cluster)             │
└───────────────────────────────────────────┘

```

---

## 4. Não é Necessário Usar Kube-API-Server Diretamente

### 4.1 Alternativa: Usar APIs do Cluster Diretamente

Embora o Kube-API-Server seja a interface padrão, você pode usar as **próprias APIs REST do Kubernetes** diretamente:

```bash
# Método tradicional (via kubectl)
kubectl create pod meu-pod --image=nginx

# Método direto via API REST
curl -X POST https://kubernetes-api.example.com/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d @pod.json

```

### 4.2 Casos de Uso para API Direta

**Quando usar a API REST diretamente:**

- Integração em scripts ou aplicações
- Automação programática
- Ferramentas customizadas
- Integração CI/CD
- Monitoramento e observabilidade

### 4.3 Vantagens e Desvantagens

### ✅ Vantagens de Usar Diretamente a API

- Controle total e granular
- Sem dependência do kubectl
- Possibilidade de automação customizada
- Melhor integração com aplicações

### ❌ Desvantagens

- Precisa gerenciar autenticação (tokens/certificados)
- Requer conhecimento detalhado da API
- Sem validação do lado do cliente
- Mais complexo de depurar

---

## 5. Componentes do Kube-API-Server

### 5.1 Arquitetura Interna

```
┌─────────────────────────────────────────┐
│      Kube-API-Server                    │
│                                         │
│  ┌───────────────────────────────────┐ │
│  │  HTTP(S) Handler                  │ │
│  │  (Recebe requisições)             │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  Authentication Modules           │ │
│  │  ├─ Certificate Auth              │ │
│  │  ├─ Token Auth                    │ │
│  │  ├─ Basic Auth                    │ │
│  │  └─ OIDC                          │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  Authorization Modules (RBAC)     │ │
│  │  ├─ Role-Based Access Control     │ │
│  │  ├─ Attribute-Based Access Control│ │
│  │  └─ Webhooks                      │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  Admission Controllers            │ │
│  │  ├─ Validating Webhook            │ │
│  │  ├─ Mutating Webhook              │ │
│  │  └─ Validadores internos          │ │
│  └─────────────┬─────────────────────┘ │
│                │                       │
│  ┌─────────────▼─────────────────────┐ │
│  │  ETCD Interface                   │ │
│  │  └─ Persistência de dados         │ │
│  └─────────────────────────────────────┘ │
│                                         │
└─────────────────────────────────────────┘

```

---

## 6. Exemplos de Uso

### 6.1 Usando kubectl (Via Kube-API-Server)

```bash
# Lista pods
kubectl get pods

# Cria um pod
kubectl create pod meu-pod --image=nginx

# Atualiza um deployment
kubectl set image deployment/meu-app app=nginx:1.21

# Deleta um recurso
kubectl delete pod meu-pod

```

**Nos bastidores:**

- kubectl envia requisição HTTPS para Kube-API-Server
- Kube-API-Server autentica e autoriza
- Valida a solicitação
- Atualiza ETCD
- Retorna resposta

### 6.2 Usando API REST Diretamente

### Obter Token

```bash
# Para service account
kubectl create serviceaccount meu-usuario
TOKEN=$(kubectl describe secret $(kubectl get secret -o name | grep meu-usuario) | grep token: | awk '{print $2}')

```

### Listar Pods via API

```bash
curl -X GET https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  --cacert /etc/kubernetes/pki/ca.crt

```

### Criar Pod via API

```bash
curl -X POST https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  --cacert /etc/kubernetes/pki/ca.crt \
  -d '{
    "apiVersion": "v1",
    "kind": "Pod",
    "metadata": {
      "name": "meu-pod",
      "namespace": "default"
    },
    "spec": {
      "containers": [
        {
          "name": "nginx",
          "image": "nginx:latest"
        }
      ]
    }
  }'

```

### Atualizar Pod via API

```bash
curl -X PATCH https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods/meu-pod \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/strategic-merge-patch+json" \
  --cacert /etc/kubernetes/pki/ca.crt \
  -d '{
    "spec": {
      "containers": [
        {
          "name": "nginx",
          "image": "nginx:1.21"
        }
      ]
    }
  }'

```

### Deletar Pod via API

```bash
curl -X DELETE https://kubernetes-api.example.com:6443/api/v1/namespaces/default/pods/meu-pod \
  -H "Authorization: Bearer $TOKEN" \
  --cacert /etc/kubernetes/pki/ca.crt

```

---

## 7. Métodos HTTP Utilizados

| Método | Operação | Exemplo |
| --- | --- | --- |
| **GET** | Ler/Listar | `GET /api/v1/pods` |
| **POST** | Criar | `POST /api/v1/namespaces/default/pods` |
| **PUT** | Substituir completamente | `PUT /api/v1/namespaces/default/pods/meu-pod` |
| **PATCH** | Atualizar parcialmente | `PATCH /api/v1/namespaces/default/pods/meu-pod` |
| **DELETE** | Deletar | `DELETE /api/v1/namespaces/default/pods/meu-pod` |
| **WATCH** | Monitorar mudanças | `GET /api/v1/pods?watch=true` |

---

## 8. Segurança

### 8.1 Autenticação

**Métodos suportados:**

- **Client Certificates** (X.509)
- **Bearer Tokens** (JWT, OIDC)
- **Basic Authentication** (username:password)
- **Proxy Authentication**
- **OpenID Connect (OIDC)**

### 8.2 Autorização

**Modes:**

- **RBAC** (Role-Based Access Control) - Mais comum
- **ABAC** (Attribute-Based Access Control)
- **Webhook**
- **Node** (para kubelet)

### 8.3 Exemplo de RBAC

```yaml
# Define uma Role com permissões
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "list"]
---
# Associa a Role a um usuário
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: pod-reader
subjects:
- kind: ServiceAccount
  name: default
  namespace: default

```

---

## 9. Interação com Outros Componentes

```
┌──────────────────────┐
│   Componentes        │
│   do Cluster         │
└──────┬───────────────┘
       │
       ├─ kubelet ─────────┐
       │                   │
       ├─ controller-manager
       │                   │
       ├─ scheduler        │
       │                   │
       └─ other clients ───┤
                           │
                    ┌──────▼────────────┐
                    │ Kube-API-Server   │
                    └──────┬────────────┘
                           │
                    ┌──────▼────────────┐
                    │      ETCD         │
                    └───────────────────┘

```

---

## 10. Configuração do Kube-API-Server

O Kube-API-Server pode ser configurado com várias flags:

```bash
kube-apiserver \
  --etcd-servers=https://127.0.0.1:2379 \
  --authorization-mode=RBAC \
  --admission-control=NamespaceLifecycle,LimitRanger \
  --client-ca-file=/etc/kubernetes/pki/ca.crt \
  --tls-cert-file=/etc/kubernetes/pki/apiserver.crt \
  --tls-private-key-file=/etc/kubernetes/pki/apiserver.key \
  --kubelet-client-certificate=/etc/kubernetes/pki/apiserver-kubelet-client.crt \
  --kubelet-client-key=/etc/kubernetes/pki/apiserver-kubelet-client.key

```

---

## 11. Troubleshooting

### Erro: Unauthorized

```
The connection to the server localhost:8080 was refused

```

**Causa**: Token inválido ou expirado
**Solução**: Renovar token ou certificado

### Erro: Forbidden

```
Error from server (Forbidden): pods is forbidden

```

**Causa**: Usuário não tem permissão
**Solução**: Verificar RBAC e RoleBindings

### Erro: Validation Error

```
error validating data: data[spec.containers[0].image]: required value

```

**Causa**: Dados enviados estão incompletos ou inválidos
**Solução**: Revisar formato do JSON/YAML

---

## 12. Resumo das Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Comunicação** | Único ponto de entrada para interagir com o cluster |
| **Autenticação** | Verifica identidade do cliente |
| **Autorização** | Verifica permissões (RBAC) |
| **Validação** | Valida formato e regras de negócio |
| **Persistência** | Interage com ETCD para armazenar dados |
| **Orquestração** | Comunica com outros componentes (scheduler, controller) |

---

## 13. Conclusão

- **Kube-API-Server** é o coração comunicativo do Kubernetes
- É responsável por **autenticação, autorização e validação**
- Comunica com **ETCD** para persistir dados
- Não é necessário usá-lo diretamente se preferir usar as **APIs REST do cluster**
- Implementa segurança através de **RBAC e autenticação**
- Todos os componentes do cluster o utilizam

---

# Kube Controller Manager - Guia Completo

## 1. O que é Kube Controller Manager?

**Kube Controller Manager** é um componente crítico do Kubernetes que funciona como o **"maestro"** do cluster. É responsável por monitorar continuamente o estado de vários componentes e garantir que o sistema sempre funcione no **estado desejado**.

### 1.1 Analogia

Imagine o Kube Controller Manager como um **maestro de orquestra**:

- 🎵 Cada músico (componente) é monitorado continuamente
- 📋 O maestro tem a partitura (estado desejado)
- 🔄 Se algum músico para de tocar, o maestro intervém
- 🎼 Garante que toda a orquestra toque em harmonia

---

## 2. Responsabilidades Principais

### 2.1 Monitoramento Contínuo

- Observa constantemente o estado dos recursos no cluster
- Detecta mudanças e desvios do estado desejado
- Reage automaticamente a problemas

### 2.2 Manutenção do Estado Desejado

- Garante que o estado atual **sempre corresponda** ao estado desejado
- Se há desviação, toma ações corretivas
- Funciona em **loops infinitos** (reconciliation loops)

### 2.3 Processamento e Orquestração

- Processa requisições de recursos
- Orquestra a criação e gerenciamento de recursos
- Comunica com Kube-API-Server para ler e atualizar estado

### 2.4 Automação

- Automatiza tarefas repetitivas
- Redimensiona deployments
- Gerencia replicação de pods
- Renova certificados automaticamente

---

## 3. Arquitetura: Controllers (Controladores)

O Kube Controller Manager **não é um único processo**, mas sim um **conjunto de controladores independentes** que trabalham em conjunto.

### 3.1 Controllers Principais

```
┌──────────────────────────────────────────────────────┐
│         Kube Controller Manager                      │
│  (Conjunto de Controllers)                           │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Deployment Controller                         │ │
│  │  - Monitora deployments                        │ │
│  │  - Garante número correto de replicas          │ │
│  │  - Gerencia rolling updates                    │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  ReplicaSet Controller                         │ │
│  │  - Gerencia replicasets                        │ │
│  │  - Mantém número desejado de pods              │ │
│  │  - Cria/deleta pods automaticamente            │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  StatefulSet Controller                        │ │
│  │  - Gerencia statefulsets                       │ │
│  │  - Mantém identidade estável                   │ │
│  │  - Gerencia ordem de criação                   │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  DaemonSet Controller                          │ │
│  │  - Garante um pod por node                     │ │
│  │  - Executa em todos os nós                     │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Job Controller                                │ │
│  │  - Gerencia jobs únicos                        │ │
│  │  - Garante conclusão bem-sucedida              │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  CronJob Controller                            │ │
│  │  - Schedula jobs periodicamente                │ │
│  │  - Gerencia agendamento                        │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Node Controller                               │ │
│  │  - Monitora saúde dos nós                      │ │
│  │  - Remove pods de nós indisponíveis            │ │
│  │  - Gerencia ciclo de vida dos nós              │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Service Account Controller                    │ │
│  │  - Cria service accounts padrão                │ │
│  │  - Gerencia credenciais                        │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  Namespace Controller                          │ │
│  │  - Gerencia ciclo de vida de namespaces        │ │
│  │  - Limpa recursos ao deletar namespace         │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  PersistentVolume Controller                   │ │
│  │  - Gerencia volumes persistentes               │ │
│  │  - Controla ciclo de vida de PVs               │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
│  ┌────────────────────────────────────────────────┐ │
│  │  E muitos outros... (~30+ controllers)         │ │
│  └────────────────────────────────────────────────┘ │
│                                                      │
└──────────────────────────────────────────────────────┘

```

---

## 4. Como Funciona: Reconciliation Loop

Cada controller executa um **loop infinito de reconciliação**:

### 4.1 Fluxo do Reconciliation Loop

```
┌─────────────────────────────────────────────────┐
│      Reconciliation Loop (Contínuo)             │
│                                                 │
│  1. 👀 OBSERVAR                                 │
│     ├─ Monitorar estado atual                  │
│     ├─ Detectar mudanças                       │
│     └─ Ler eventos do cluster                  │
│                                                 │
│  2. 📋 COMPARAR                                 │
│     ├─ Comparar estado atual vs desejado       │
│     ├─ Identificar diferenças                  │
│     └─ Determinar ações necessárias            │
│                                                 │
│  3. ⚙️  AGIR                                    │
│     ├─ Se há diferença, tomar ação             │
│     ├─ Criar/atualizar/deletar recursos        │
│     ├─ Comunicar via Kube-API-Server           │
│     └─ Persistir mudanças no ETCD              │
│                                                 │
│  4. 🔄 REPETIR                                  │
│     └─ Volta ao passo 1 continuamente          │
│                                                 │
└─────────────────────────────────────────────────┘

```

### 4.2 Exemplo Prático: Deployment Controller

```yaml
# Estado Desejado (YAML que você criou)
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-app
spec:
  replicas: 3  # ← Quer 3 pods
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest

```

**O que o Deployment Controller faz:**

```
Ciclo 1:
┌─────────────────────────┐
│ Observar                │
│ Pods atuais: 0          │  ← Menos do que o desejado (3)
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Comparar                │
│ Desejado: 3             │
│ Atual: 0                │
│ Ação: Criar 3 pods      │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Agir                    │
│ ✓ Criar pod-1           │
│ ✓ Criar pod-2           │
│ ✓ Criar pod-3           │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Estado após ação        │
│ Pods atuais: 3 ✓        │  ← Corresponde ao desejado
└─────────────────────────┘

Ciclo 2 (alguns minutos depois):
┌─────────────────────────┐
│ Observar                │
│ Pods atuais: 2          │  ← Um pod foi deletado! (problema)
│ Pod-2 está Down         │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Comparar                │
│ Desejado: 3             │
│ Atual: 2                │
│ Ação: Criar 1 pod       │
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Agir                    │
│ ✓ Criar novo pod-4      │  ← Repõe o pod que saiu
└────────┬────────────────┘
         │
┌────────▼────────────────┐
│ Estado após ação        │
│ Pods atuais: 3 ✓        │  ← Novamente no estado desejado
└─────────────────────────┘

```

---

## 5. Controllers Importantes Detalhados

### 5.1 Deployment Controller

**Responsabilidades:**

- Monitora deployments
- Cria/atualiza ReplicaSets
- Gerencia rolling updates
- Rollback automático em caso de erro

**Exemplo:**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: meu-app
spec:
  replicas: 3  # ← Deployment Controller garante isso
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # Máximo 1 pod extra durante update
      maxUnavailable: 0  # Nunca deixar 0 pods disponíveis

```

### 5.2 ReplicaSet Controller

**Responsabilidades:**

- Mantém o número exato de pods
- Cria novos pods se alguns forem deletados
- Deleta pods extras

### 5.3 Node Controller

**Responsabilidades:**

- Monitora saúde dos nós
- Remove pods de nós indisponíveis
- Marca nós como não prontos/indisponíveis

**Exemplo de Ação:**

```
Nó se desconecta → Node Controller detecta
    ↓
Marca nó como "NotReady" → Pods começam a ser removidos
    ↓
ReplicaSet Controller detecta pods faltando
    ↓
Cria pods de reposição em nós saudáveis

```

### 5.4 Job Controller

**Responsabilidades:**

- Gerencia jobs que rodam uma única vez
- Garante conclusão bem-sucedida
- Faz retry em caso de falha

**Exemplo:**

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: backup-job
spec:
  template:
    spec:
      containers:
      - name: backup
        image: backup-tool:latest
  backoffLimit: 3  # ← Job Controller refaz até 3 vezes em caso de falha

```

### 5.5 CronJob Controller

**Responsabilidades:**

- Schedula jobs periodicamente
- Gerencia histórico de execução
- Limpa jobs antigos automaticamente

**Exemplo:**

```yaml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: cleanup-job
spec:
  schedule: "0 2 * * *"  # ← Executa todo dia às 2 da manhã
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: cleanup
            image: cleanup-tool:latest

```

### 5.6 Namespace Controller

**Responsabilidades:**

- Gerencia ciclo de vida de namespaces
- Limpa recursos quando namespace é deletado
- Aplica políticas de finalização

---

## 6. Comunicação com Kube-API-Server

O Kube Controller Manager funciona **através do Kube-API-Server**:

### 6.1 Fluxo de Comunicação

```
┌──────────────────────────────┐
│  Kube Controller Manager     │
└────────────┬─────────────────┘
             │
             │ Requisições HTTPS
             │ (Autenticado)
             ▼
┌──────────────────────────────┐
│  Kube-API-Server             │
│                              │
│  1. Autentica requisição     │
│  2. Autoriza ação (RBAC)     │
│  3. Valida dados             │
│  4. Persiste no ETCD         │
│  5. Retorna confirmação      │
└────────────┬─────────────────┘
             │
             ▼
┌──────────────────────────────┐
│  ETCD (Banco de Dados)       │
└──────────────────────────────┘

```

### 6.2 Operações Típicas via API

```bash
# Controller monitora recursos (Watch)
kubectl get deployments --watch

# Controller cria novos pods
kubectl create pod pod-1 --image=nginx

# Controller atualiza status
kubectl patch pod pod-1 -p '{"status":{"phase":"Running"}}'

# Controller deleta pods
kubectl delete pod pod-2

```

---

## 7. Estado Desejado vs Estado Atual

### 7.1 Exemplo: Deployment com 3 Replicas

```
┌─────────────────────────┐
│  Estado Desejado        │
│  (Arquivo YAML)         │
│                         │
│  Deployment: nginx-app  │
│  Replicas: 3            │
│  Image: nginx:1.21      │
│  Labels: app=nginx      │
└────────────┬────────────┘
             │
             │ Salvo no ETCD
             ▼
┌─────────────────────────┐
│  Estado Atual           │
│  (No Cluster)           │
│                         │
│  ReplicaSet-1           │
│  ├─ pod-1 (Running)     │
│  ├─ pod-2 (Running)     │
│  └─ pod-3 (Failed)      │  ← DIFERENÇA! Pod falhou
└────────────┬────────────┘
             │
             │ Controller detecta
             │ e toma ação
             ▼
┌─────────────────────────┐
│  Ação do Controller     │
│                         │
│  ✓ Deletar pod-3        │
│  ✓ Criar pod-4          │
└────────────┬────────────┘
             │
             ▼
┌─────────────────────────┐
│  Novo Estado Atual      │
│                         │
│  ReplicaSet-1           │
│  ├─ pod-1 (Running)     │
│  ├─ pod-2 (Running)     │
│  └─ pod-4 (Running)     │  ← Novamente em equilíbrio
└─────────────────────────┘

```

---

## 8. Exemplos de Atuações do Controller Manager

### 8.1 Caso 1: Pod Deletado Acidentalmente

```
Estado Desejado: 3 pods nginx
                  ↓
[Pod-1] [Pod-2] [Pod-3]

Ação acidental: usuario deleta Pod-2
                  ↓
[Pod-1] [X] [Pod-3]

Controller detecta: Apenas 2 pods (menos que os 3 desejados)
                  ↓
Controller cria: Novo pod-4
                  ↓
[Pod-1] [Pod-4] [Pod-3]  ← Estado desejado restaurado!

```

### 8.2 Caso 2: Nó Indisponível

```
Estado Desejado: 3 pods nginx
                  ↓
Nó-A: [Pod-1] [Pod-2]
Nó-B: [Pod-3]

Nó-A fica offline (falha de hardware)
                  ↓
Nó-A: [X] [X] (inacessível)
Nó-B: [Pod-3]

Node Controller detecta: Nó-A offline
                  ↓
Remove pods do Nó-A
                  ↓
ReplicaSet Controller detecta: Apenas 1 pod (menos que 3)
                  ↓
Cria pods de reposição em Nó-B
                  ↓
Nó-B: [Pod-3] [Pod-4] [Pod-5]  ← Estado desejado restaurado!

```

### 8.3 Caso 3: Atualização de Imagem

```
Estado Desejado Anterior: 3 pods com nginx:1.20
                  ↓
[Pod-1:1.20] [Pod-2:1.20] [Pod-3:1.20]

Atualização: Mudar para nginx:1.21
                  ↓
Estado Desejado Novo: 3 pods com nginx:1.21

Deployment Controller inicia Rolling Update:
Ciclo 1: Criar pod-4:1.21, deletar pod-1:1.20
                  ↓
[X] [Pod-2:1.20] [Pod-3:1.20] [Pod-4:1.21]

Ciclo 2: Criar pod-5:1.21, deletar pod-2:1.20
                  ↓
[X] [X] [Pod-3:1.20] [Pod-4:1.21] [Pod-5:1.21]

Ciclo 3: Criar pod-6:1.21, deletar pod-3:1.20
                  ↓
[Pod-4:1.21] [Pod-5:1.21] [Pod-6:1.21]  ← Todos atualizados!

```

---

## 9. Configuração do Kube Controller Manager

```bash
kube-controller-manager \
  --kubeconfig=/etc/kubernetes/controller-manager.conf \
  --master=https://127.0.0.1:6443 \
  --leader-elect=true \
  --controllers=*,bootstrapsigner,tokencleaner \
  --node-monitor-grace-period=40s \
  --node-monitor-period=5s \
  --pod-eviction-timeout=5m0s \
  --v=2

```

**Flags importantes:**

- `-leader-elect`: Apenas uma instância processa (alta disponibilidade)
- `-controllers`: Quais controllers ativar
- `-node-monitor-grace-period`: Tempo antes de marcar nó como indisponível
- `-pod-eviction-timeout`: Tempo antes de remover pods de nó offline

---

## 10. Monitoramento e Observabilidade

### 10.1 Ver Logs do Controller Manager

```bash
# Em um cluster local
kubectl logs -n kube-system deployment/kube-controller-manager

# Ver eventos do cluster
kubectl get events -A --sort-by='.lastTimestamp'

# Ver status de um deployment
kubectl rollout status deployment/meu-app

# Ver histórico de rollouts
kubectl rollout history deployment/meu-app

```

### 10.2 Métricas

```bash
# Obter métricas do controller manager
curl localhost:8080/metrics

```

---

## 11. Troubleshooting

### Erro: Pods não estão sendo criados

```
Possível causa: Controller Manager offline
Solução: Verificar status do controller manager
kubectl get pod -n kube-system kube-controller-manager-master

```

### Erro: Deployment está travado

```
Possível causa: Imagem inválida, recursos insuficientes
Solução: Verificar eventos
kubectl describe deployment meu-app

```

### Erro: Pods em ErrImagePull

```
Possível causa: Imagem não encontrada
Solução: Controller não conseguiu criar pod com essa imagem
kubectl logs <pod-id> para ver erro

```

---

## 12. Resumo das Responsabilidades

| Responsabilidade | Controller | Descrição |
| --- | --- | --- |
| **Replicas** | ReplicaSet | Mantém número correto de pods |
| **Deployments** | Deployment | Gerencia rolling updates |
| **StatefulSets** | StatefulSet | Mantém identidade e ordem |
| **DaemonSets** | DaemonSet | Garante 1 pod por node |
| **Jobs** | Job | Garante conclusão bem-sucedida |
| **CronJobs** | CronJob | Executa jobs periodicamente |
| **Nós** | Node | Monitora saúde dos nós |
| **Namespaces** | Namespace | Limpa recursos de namespace deletado |

---

## 13. Diagrama Completo de Interação

```
┌────────────────────────────────────────────────────────┐
│         Usuário ou Aplicação                           │
│  (cria deployment, atualiza imagem, etc)               │
└───────────────┬──────────────────────────────────────┘
                │
                │ kubectl apply -f deployment.yaml
                │
                ▼
        ┌──────────────────┐
        │  Kube-API-Server │
        │  (persistir YAML)│
        └────────┬─────────┘
                 │
                 │ Salva em ETCD
                 │
                 ▼
        ┌──────────────────┐
        │ ETCD             │
        │ (Banco de Dados) │
        └────────┬─────────┘
                 │
                 │ Notifica mudança
                 │
                 ▼
    ┌────────────────────────────┐
    │ Kube Controller Manager    │
    │                            │
    │ Deployment Controller:     │
    │ ├─ Lê novo deployment      │
    │ ├─ Cria ReplicaSet         │
    │ └─ Comunica com API-Server │
    │                            │
    │ ReplicaSet Controller:     │
    │ ├─ Lê ReplicaSet           │
    │ ├─ Cria 3 pods             │
    │ └─ Comunica com API-Server │
    └────────┬───────────────────┘
             │
             │ Requisições create pod
             │
             ▼
    ┌──────────────────┐
    │  Kube-API-Server │
    │  (criar pods)    │
    └────────┬─────────┘
             │
             │ ETCD atualizado
             │
             ▼
    ┌────────────────────┐
    │ Scheduler          │
    │ (Coloca pods nos   │
    │  nós apropriados)  │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ Kubelet (cada nó)  │
    │ (executa pods)     │
    └────────────────────┘

```

---

## 14. Conclusão

- **Kube Controller Manager** é o "maestro" do Kubernetes
- Executa **loops infinitos de reconciliação** para cada controller
- **Monitora continuamente** o estado do cluster
- **Toma ações automáticas** para manter o estado desejado
- Comunica **apenas via Kube-API-Server**
- Garante **alta disponibilidade** e **auto-recuperação** do cluster
- É **essencial** para o funcionamento automático do Kubernetes

---

# Kube Scheduler - Guia Completo

## 1. O que é Kube Scheduler?

**Kube Scheduler** é um componente do Kubernetes responsável por **decidir em qual nó (máquina) cada pod será executado**. É o "distribuidor inteligente" do cluster, colocando os pods nos lugares certos.

### 1.1 Analogia

Imagine o Kube Scheduler como um **gerente de recursos de um hotel**:

- 🏨 Tem vários quartos (nós) disponíveis
- 👥 Chegam novos hóspedes (pods) para alojar
- 🔍 Analisa qual quarto é melhor para cada hóspede
- ✅ Aloca o hóspede ao quarto mais apropriado

---

## 2. Responsabilidades Principais

### 2.1 Seleção de Nós

- **Escolher qual nó** executará cada pod
- Considerar **recursos disponíveis** (CPU, memória)
- Levar em conta **restrições e preferências** do pod

### 2.2 Análise de Restrições

- **Verificar se o nó atende aos requisitos** do pod
- Considerar afinidade (node affinity)
- Considerar repulsão (pod affinity/anti-affinity)
- Validar tolerâncias (tolerations)

### 2.3 Distribuição Eficiente

- **Distribuir pods** de forma equilibrada
- Evitar **sobrecarga** em nós específicos
- Melhorar **utilização de recursos** do cluster
- Permitir **escalabilidade** do sistema

### 2.4 Otimização

- Considerar **locais onde dados já existem** (data locality)
- **Agrupar ou separar** pods conforme necessário
- Balancear **performance e utilização**

---

## 3. Fluxo do Processo de Scheduling

### 3.1 Quando o Scheduler Entra em Ação

```
1. 👤 Usuário cria um Pod/Deployment
   ↓
2. 📝 Kube-API-Server recebe e persiste no ETCD
   ↓
3. 🔍 Kube Controller Manager cria pods (se for deployment)
   ↓
4. 📌 Pod fica em estado "Pending" (aguardando scheduler)
   ↓
5. 🎯 Kube Scheduler detecta pod sem nó atribuído
   ↓
6. ⚙️  Scheduler analisa nós e escolhe o melhor
   ↓
7. ✅ Scheduler atribui o pod a um nó (via Kube-API-Server)
   ↓
8. 🚀 Kubelet no nó vê o pod e começa a executar
   ↓
9. 📦 Pod entra em estado "Running"

```

### 3.2 Estados do Pod no Processo

```
Pod Life Cycle:

         ┌─────────────┐
         │  Pending    │ ← Pod criado, aguardando scheduler
         └──────┬──────┘
                │ (Scheduler escolhe nó)
         ┌──────▼──────┐
         │  Container  │ ← Kubelet começa a executar
         │  Creating   │
         └──────┬──────┘
                │ (Container iniciado)
         ┌──────▼──────┐
         │  Running    │ ← Pod está executando
         └──────┬──────┘
                │ (Pod termina)
         ┌──────▼──────┐
         │  Succeeded  │ ← Completado com sucesso
         │  ou Failed  │   ou Falhou
         └─────────────┘

```

---

## 4. Arquitetura do Scheduler

### 4.1 Componentes Internos

```
┌──────────────────────────────────────────────────────┐
│         Kube Scheduler                               │
│                                                      │
│  ┌─────────────────────────────────────────────┐   │
│  │  1. Informação do Pod                       │   │
│  │     - Requisitos de recursos (CPU, memória)│   │
│  │     - Restrições (nodeSelector)            │   │
│  │     - Tolerâncias (tolerations)            │   │
│  │     - Afinidade (affinity)                 │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  2. Filtragem (Filter Phase)               │   │
│  │     - Remove nós que NÃO podem executar    │   │
│  │     - Verifica recursos mínimos            │   │
│  │     - Valida tolerâncias                   │   │
│  │     - Verifica taints do nó                │   │
│  │     Result: Lista de nós "candidatos"     │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  3. Scoring (Scoring Phase)                │   │
│  │     - Avalia cada nó candidato             │   │
│  │     - Calcula score para cada nó           │   │
│  │     - Considera múltiplos fatores          │   │
│  │     - Plugins de priorização               │   │
│  │     Result: Nós ranqueados                │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  4. Seleção (Selection)                    │   │
│  │     - Escolhe nó com maior score           │   │
│  │     - Em caso de empate, escolhe aleatório │   │
│  │     Result: Nó final escolhido             │   │
│  └──────────────┬────────────────────────────┘   │
│                 │                                │
│  ┌──────────────▼────────────────────────────┐   │
│  │  5. Binding (Vinculação)                   │   │
│  │     - Comunica com Kube-API-Server         │   │
│  │     - Atribui pod ao nó via ETCD           │   │
│  │     Result: Pod vinculado ao nó            │   │
│  └────────────────────────────────────────────┘   │
│                                                      │
└──────────────────────────────────────────────────────┘

```

---

## 5. Filtragem (Filter Phase)

### 5.1 O que é Filtragem?

Na fase de filtragem, o scheduler **remove nós que NÃO podem executar o pod**.

### 5.2 Critérios de Filtragem

```
┌─────────────────────────────────────┐
│    Pod a Agendar                    │
│                                     │
│  Requisitos:                        │
│  - CPU: 500m                        │
│  - Memória: 256Mi                   │
│  - NodeSelector: disktype=ssd       │
│  - Tolerations: none                │
│  - Affinity: none                   │
└─────────────────────────────────────┘
                  │
                  ▼
┌──────────────────────────────────────────┐
│  Fase de Filtragem                       │
│                                          │
│  Nó-1:                                   │
│  ├─ CPU disponível: 100m  ❌             │
│  │  (Insuficiente: 500m > 100m)          │
│  └─ ELIMINADO                            │
│                                          │
│  Nó-2:                                   │
│  ├─ CPU disponível: 1000m ✓              │
│  ├─ Memória disponível: 512Mi ✓          │
│  ├─ Disco: SSD ✓                         │
│  ├─ Não tainted ✓                        │
│  └─ CANDIDATO                            │
│                                          │
│  Nó-3:                                   │
│  ├─ CPU disponível: 800m ✓               │
│  ├─ Memória disponível: 512Mi ✓          │
│  ├─ Disco: HDD ❌                        │
│  │  (NodeSelector requer SSD)            │
│  └─ ELIMINADO                            │
│                                          │
│  Nó-4:                                   │
│  ├─ CPU disponível: 1500m ✓              │
│  ├─ Memória disponível: 1Gi ✓            │
│  ├─ Disco: SSD ✓                         │
│  ├─ Taint: NoExecute ❌                  │
│  │  (Sem tolerância para este taint)     │
│  └─ ELIMINADO                            │
│                                          │
│  Resultado: Apenas Nó-2 é candidato    │
└──────────────────────────────────────────┘

```

### 5.3 Predicados Comuns (Filtros)

| Filtro | Descrição | Exemplo |
| --- | --- | --- |
| **PodFitsResources** | Verifica CPU/Memória | Nó tem 512Mi, pod precisa 256Mi ✓ |
| **PodFitsHost** | Verifica port bindings | Porta já em uso ✗ |
| **PodSelectorMatches** | Verifica nodeSelector | nodeSelector: gpu=true |
| **NoDiskConflict** | Verifica volumes | Volumes não em conflito |
| **PodToleratesNodeTaints** | Verifica tolerações | Tolerations vs Taints |
| **NodeAffinity** | Verifica afinidade | Nó atende requerimentos |

---

## 6. Scoring (Scoring Phase)

### 6.1 O que é Scoring?

Na fase de scoring, o scheduler **avalia os nós candidatos** e dá uma pontuação a cada um. O nó com maior pontuação é escolhido.

### 6.2 Fatores de Scoring

```
Após filtragem, restam:
├─ Nó-2 ✓
├─ Nó-5 ✓
└─ Nó-7 ✓

Fase de Scoring:

┌────────────────────────────────┐
│  Nó-2                          │
├────────────────────────────────┤
│ • Recursos disponíveis: Alto   │ +80
│ • CPU livre: 500m de 1000m     │
│ • Preferência de zona: Mesma   │ +20
│ • Localidade de dados: Ótima   │ +30
├────────────────────────────────┤
│ TOTAL: 130 pontos              │
└────────────────────────────────┘

┌────────────────────────────────┐
│  Nó-5                          │
├────────────────────────────────┤
│ • Recursos disponíveis: Médio  │ +50
│ • CPU livre: 200m de 1000m     │
│ • Preferência de zona: Distante│ +10
│ • Localidade de dados: Ruim    │ +5
├────────────────────────────────┤
│ TOTAL: 65 pontos               │
└────────────────────────────────┘

┌────────────────────────────────┐
│  Nó-7                          │
├────────────────────────────────┤
│ • Recursos disponíveis: Médio  │ +50
│ • CPU livre: 100m de 1000m     │
│ • Preferência de zona: Mesma   │ +20
│ • Localidade de dados: Média   │ +15
├────────────────────────────────┤
│ TOTAL: 85 pontos               │
└────────────────────────────────┘

Resultado: Nó-2 vencedor (130 pontos) ✓

```

### 6.3 Plugins de Priorização Comuns

| Plugin | Descrição | Objetivo |
| --- | --- | --- |
| **LeastRequested** | Nós com menos recursos usados | Distribuir carga |
| **BalancedResourceAllocation** | Balancear CPU e memória | Uso uniforme |
| **NodePreferAvoidPods** | Evitar nós marcados | Graceful shutdown |
| **TaintToleration** | Preferir nós com taints compatíveis | Isolamento |
| **NodeAffinity** | Nós que batem afinidade | Controle fino |
| **PodAffinity** | Agrupar pods relacionados | Localidade |
| **PodAntiAffinity** | Separar pods diferentes | Disponibilidade |

---

## 7. Restrições e Preferências

### 7.1 NodeSelector (Simples)

**NodeSelector** é a forma mais simples de restringir em quais nós um pod pode rodar.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-ssd
spec:
  nodeSelector:
    disktype: ssd  # ← Pod DEVE rodar em nó com esta label
  containers:
  - name: nginx
    image: nginx:latest
    resources:
      requests:
        cpu: 500m
        memory: 256Mi

```

**Como preparar o nó:**

```bash
# Adicionar label ao nó
kubectl label nodes nó-2 disktype=ssd

# Verificar
kubectl get nodes --show-labels

```

### 7.2 Node Affinity (Avançado)

**Node Affinity** oferece controle mais fino sobre placement.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: webapp
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: kubernetes.io/hostname
            operator: In
            values:
            - nó-2
            - nó-3
            - nó-4
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
  containers:
  - name: webapp
    image: webapp:latest

```

**Tipos:**

- `requiredDuringSchedulingIgnoredDuringExecution` - **DEVE** atender (hard)
- `preferredDuringSchedulingIgnoredDuringExecution` - **PREFERE** atender (soft)

### 7.3 Taints e Tolerations

**Taints** marcam nós como indisponíveis para certos pods. **Tolerations** permitem que pods ignorem taints.

```yaml
# Marcar nó com taint (via CLI)
kubectl taint nodes nó-gpu gpu=true:NoSchedule

# Pod com toleração
apiVersion: v1
kind: Pod
metadata:
  name: gpu-job
spec:
  tolerations:
  - key: gpu
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: gpu-worker
    image: gpu-worker:latest

```

**Efeitos de Taints:**

- `NoSchedule` - Não agenda novos pods (existentes continuam)
- `PreferNoSchedule` - Prefere não agendar (soft)
- `NoExecute` - Remove pods existentes (hard)

### 7.4 Pod Affinity / Anti-Affinity

Controla se pods devem estar **juntos ou separados**.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-frontend
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - database
        topologyKey: kubernetes.io/hostname
  containers:
  - name: frontend
    image: frontend:latest

```

**Interpretação:**

- Este frontend **DEVE** rodar no **mesmo nó** que o pod com label `app=database`

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-cache
spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchExpressions:
            - key: app
              operator: In
              values:
              - database
          topologyKey: kubernetes.io/hostname
  containers:
  - name: cache
    image: cache:latest

```

**Interpretação:**

- Este cache **PREFERE** rodar em **nó diferente** do pod com label `app=database`

---

## 8. Exemplo Prático Completo

### 8.1 Cenário: Deployment Web

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
        tier: frontend
    spec:
      # 1. Requisitos de recursos
      containers:
      - name: web
        image: nginx:latest
        resources:
          requests:
            cpu: 250m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 256Mi

      # 2. Afinidade de nó (preferir nós SSD)
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            preference:
              matchExpressions:
              - key: disktype
                operator: In
                values:
                - ssd

      # 3. Anti-afinidade de pod (pods em nós diferentes)
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - web
              topologyKey: kubernetes.io/hostname

```

### 8.2 Processo de Scheduling para Este Deployment

```
1. Replicaset Controller cria 3 pods
   Pod-1, Pod-2, Pod-3 (todos com Pending)

2. Scheduler processa Pod-1:

   Filtragem:
   ├─ Nó-1: 250m + 128Mi de recursos ✓
   ├─ Nó-2: 250m + 128Mi de recursos ✓
   ├─ Nó-3: Indisponível (Nó offline) ✗
   └─ Nó-4: 250m + 128Mi de recursos ✓

   Scoring:
   ├─ Nó-1: disktype=hdd, score=50
   ├─ Nó-2: disktype=ssd, score=100
   └─ Nó-4: disktype=hdd, score=50

   Vencedor: Nó-2 ✓

3. Scheduler processa Pod-2:

   Filtragem: Nó-1 ✓, Nó-2 ✓, Nó-4 ✓

   Scoring:
   ├─ Nó-1: disktype=hdd (50) + não tem web (100) = 150
   ├─ Nó-2: disktype=ssd (100) + já tem web (0) = 100 ❌ Anti-afinidade
   └─ Nó-4: disktype=hdd (50) + não tem web (100) = 150

   Vencedor: Nó-1 (ou Nó-4, aleatório entre 150) ✓

4. Scheduler processa Pod-3:

   Filtragem: Nó-2 ✓, Nó-4 ✓

   Scoring:
   ├─ Nó-2: disktype=ssd (100) + já tem web (0) = 100
   └─ Nó-4: disktype=hdd (50) + não tem web (100) = 150

   Vencedor: Nó-4 ✓

Resultado Final:
├─ Pod-1: Nó-2 ✓ (SSD preferido)
├─ Pod-2: Nó-1 ✓ (Separado de Pod-1)
└─ Pod-3: Nó-4 ✓ (Separado dos outros)

Todos os 3 pods em nós diferentes (Anti-affinity respeitada)!

```

---

## 9. Algoritmos de Scheduling

### 9.1 Scheduler Framework (Padrão Atual)

```
Pod → Filter Plugins → Scoring Plugins → Nó Final

┌────────────────────────────────────────────┐
│  Pod com requisitos de scheduling          │
└────────────┬───────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────┐
│  Filter Plugins                            │
│  ├─ ResourceFit                            │
│  ├─ NodeUnschedulable                      │
│  ├─ NodeName                               │
│  ├─ TaintToleration                        │
│  ├─ NodeAffinity                           │
│  └─ PodAffinity (filtragem prévia)         │
└────────────┬───────────────────────────────┘
             │ (remove nós inadequados)
             ▼
┌────────────────────────────────────────────┐
│  Scoring Plugins                           │
│  ├─ NodeResourcesFit                       │
│  ├─ ImageLocality                          │
│  ├─ InterPodAffinity                       │
│  ├─ NodeAffinity                           │
│  ├─ TaintToleration                        │
│  └─ PodTopologySpread                      │
└────────────┬───────────────────────────────┘
             │ (ordena nós por score)
             ▼
┌────────────────────────────────────────────┐
│  Seleção Final                             │
│  (Nó com maior score)                      │
└────────────┬───────────────────────────────┘
             │
             ▼
┌────────────────────────────────────────────┐
│  Binding                                   │
│  (Vincular pod ao nó via API-Server)       │
└────────────────────────────────────────────┘

```

---

## 10. Configuração do Scheduler

### 10.1 Configuração Básica

```bash
kube-scheduler \
  --kubeconfig=/etc/kubernetes/scheduler.conf \
  --leader-elect=true \
  --v=2

```

### 10.2 Configuração com Arquivo Config

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- name: default
  plugins:
    preFilter:
      enabled:
      - name: DefaultPreemption
    filter:
      enabled:
      - name: NodeUnschedulable
      - name: NodeAffinity
      - name: TaintToleration
    postFilter:
      enabled:
      - name: DefaultPreemption
    preScore:
      enabled:
      - name: NodeAffinity
    score:
      enabled:
      - name: NodeResourcesFit
        weight: 1
      - name: ImageLocality
        weight: 1
      - name: NodeAffinity
        weight: 1
    reserve:
      enabled:
      - name: VolumeBinding

```

---

## 11. Monitoramento e Troubleshooting

### 11.1 Ver Status de Pods Pendentes

```bash
# Ver pods não agendados
kubectl get pods -A --field-selector=status.phase=Pending

# Ver detalhes do pod (eventos)
kubectl describe pod meu-pod -n meu-namespace

# Ver logs do scheduler
kubectl logs -n kube-system -l component=kube-scheduler

```

### 11.2 Exemplos de Problemas

### Problema 1: Pod Não Consegue Ser Agendado

```
$ kubectl describe pod app-pod

Events:
  Type     Reason            Age                From               Message
  ----     ------            ----               ----               -------
  Warning  FailedScheduling  2m                 default-scheduler  no nodes available to schedule pods

```

**Causa possível**: Insuficiente recursos
**Solução**: Aumentar recursos do nó ou reduzir requisitos do pod

### Problema 2: Pod Aguardando NodeSelector

```
$ kubectl describe pod gpu-pod

Events:
  Warning  FailedScheduling  2m  default-scheduler  0/5 nodes are available: 5 node(s) didn't match Pod's node selector.

```

**Causa**: Label `gpu=true` não existe em nó algum
**Solução**: `kubectl label nodes nó-1 gpu=true`

### Problema 3: Pod com Taint Incompatível

```
$ kubectl describe pod normal-pod

Events:
  Warning  FailedScheduling  2m  default-scheduler  0/3 nodes are available: 3 node(s) have taints that the pod does not tolerate: {gpu=true:NoSchedule}

```

**Causa**: Nó tem taint, pod não tem toleração
**Solução**: Adicionar toleração ao pod ou remover taint

### 11.3 Comandos Úteis

```bash
# Ver recursos do nó
kubectl top nodes

# Ver recursos do pod
kubectl top pods

# Ver labels dos nós
kubectl get nodes --show-labels

# Ver taints dos nós
kubectl describe node nó-1 | grep Taints

# Simular scheduling (dry-run)
kubectl create pod test --image=nginx --dry-run=client -o yaml | kubectl apply -f - --dry-run=client

```

---

## 12. Otimizações Avançadas

### 12.1 Pod Topology Spread

Espalhar pods entre diferentes zonas/domínios:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-spread
spec:
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app: web
  containers:
  - name: web
    image: nginx:latest

```

**Interpretação:**

- Máximo 1 pod por zona (distribuído uniformemente)
- Se não conseguir distribuir, não agenda

### 12.2 Priority Classes

Definir prioridade de scheduling:

```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 1000
globalDefault: false
description: "Alto priority para apps críticas"
---
apiVersion: v1
kind: Pod
metadata:
  name: critical-app
spec:
  priorityClassName: high-priority
  containers:
  - name: app
    image: app:latest

```

---

## 13. Resumo de Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Filtragem** | Remove nós que não podem executar o pod |
| **Scoring** | Avalia nós candidatos e ordena por score |
| **Seleção** | Escolhe nó com maior score |
| **Binding** | Vincula pod ao nó via API-Server |
| **Otimização** | Distribui pods de forma eficiente |

---

## 14. Fluxo Completo do Cluster

```
┌─────────────────────────────────────────────────────────────┐
│  1. Usuário cria Deployment                                │
│     kubectl create deployment app --image=nginx             │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  2. Kube-API-Server recebe e persiste no ETCD              │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  3. Kube Controller Manager                                │
│     └─ ReplicaSet Controller cria 3 pods (Pending)         │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  4. Kube Scheduler                                          │
│     ├─ Filtra nós candidatos                               │
│     ├─ Scored nós                                          │
│     ├─ Escolhe melhor nó para cada pod                     │
│     └─ Vincula pod ao nó                                   │
└────────────────┬────────────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────────────┐
│  5. Kubelet (em cada nó)                                   │
│     ├─ Detecta novo pod agendado                           │
│     ├─ Inicia containers                                   │
│     └─ Pod entra em estado Running                         │
└─────────────────────────────────────────────────────────────┘

```

---

## 15. Comparação com Outros Schedulers

| Scheduler | Caso de Uso | Características |
| --- | --- | --- |
| **Default Scheduler** | Propósito geral | Equilibrado, plugin-based |
| **Custom Scheduler** | Casos específicos | Customizado, lógica própria |
| **Karpenter** | Escalamento eficiente | Otimizado para custo/performance |
| **Crane** | Economia de recursos | Bin-packing agressivo |

---

## 16. Conclusão

- **Kube Scheduler** é o "distribuidor inteligente" do cluster
- **Processa em 2 fases**: Filtragem e Scoring
- **Garante distribuição eficiente** de pods
- **Considera múltiplos fatores**: Recursos, afinidade, taints, etc.
- **Altamente configurável** via restrições e preferências
- É **essencial** para o funcionamento ótimo do Kubernetes

---

# Kubelet - Guia Completo

## 1. O que é Kubelet?

**Kubelet** é um agente que roda em **cada nó do cluster Kubernetes**. É responsável por **executar e gerenciar os containers** nos nós, garantindo que os pods rodem conforme desejado.

### 1.1 Analogia

Imagine o Kubelet como um **gerente de um restaurante**:

- 👨‍🍳 Cada nó é um restaurante
- 📋 Kubelet recebe pedidos (pods) da central
- ⚙️ Executa os pedidos (containers)
- 👀 Monitora a qualidade (saúde dos containers)
- 🔧 Faz ajustes quando necessário

---

## 2. Responsabilidades Principais

### 2.1 Execução de Containers

- **Executar containers** em seu nó
- Trabalhar com **container runtime** (Docker, containerd, cri-o)
- Gerenciar **ciclo de vida** dos containers

### 2.2 Monitoramento de Pods

- Monitorar **saúde dos containers**
- Detectar **falhas e reiniciar** containers
- Relatar **status do pod** ao Kube-API-Server

### 2.3 Gerenciamento de Recursos

- Garantir **limites de CPU/memória**
- Gerenciar **volumes**
- Alocar **IPs aos pods**

### 2.4 Comunicação com API Server

- **Registrar o nó** no cluster
- **Informar status** do nó e pods
- **Receber instruções** (novos pods, deletar pods)

### 2.5 Health Checks

- Executar **liveness probes** (pod vivo?)
- Executar **readiness probes** (pronto para tráfego?)
- Executar **startup probes** (iniciado?)

### 2.6 Gerenciamento de Volumes

- **Montar volumes** em containers
- **Gerenciar storage** local

---

## 3. Arquitetura do Kubelet

### 3.1 Componentes Internos

```
┌──────────────────────────────────────────────────────────┐
│                    Kubelet (em cada nó)                  │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  1. API Server Watcher                             │ │
│  │     - Monitora novo pods agendados para este nó    │ │
│  │     - Detecta pods para deletar                    │ │
│  │     - Recebe atualizações                          │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  2. Pod Manager                                   │ │
│  │     - Gerencia ciclo de vida do pod               │ │
│  │     - Rastreia pods em execução                   │ │
│  │     - Mapeia especificações ao runtime            │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  3. Volume Manager                                │ │
│  │     - Monta volumes                               │ │
│  │     - Gerencia armazenamento                      │ │
│  │     - Unmonta quando necessário                   │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  4. Container Runtime Interface (CRI)             │ │
│  │     - Comunica com runtime (Docker, containerd)   │ │
│  │     - Abstração para diferentes runtimes          │ │
│  │     - Gerencia containers                         │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  5. Probe Manager                                 │ │
│  │     - Executa liveness probes                     │ │
│  │     - Executa readiness probes                    │ │
│  │     - Executa startup probes                      │ │
│  │     - Reinicia containers que falharem            │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  6. CGROUP Manager                                │ │
│  │     - Aplica limites de recursos (CPU, memória)   │ │
│  │     - Isolamento de recursos                      │ │
│  │     - QoS (Quality of Service)                    │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  7. Status Manager                                │ │
│  │     - Rastreia status de pods/containers          │ │
│  │     - Reporta ao API-Server                       │ │
│  │     - Monitora saúde                              │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  8. Container Runtime                             │ │
│  │     - Docker/containerd/cri-o                     │ │
│  │     - Executa containers realmente                │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
└──────────────────────────────────────────────────────────┘

```

---

## 4. Ciclo de Vida de um Pod no Kubelet

### 4.1 Fluxo Completo

```
1. 👀 WATCH API-Server
   └─ Kubelet monitora novo pod agendado

2. 📥 RECEBE POD
   ├─ Nome, image, volumes, etc
   └─ Armazena em Pod Manager

3. 🔌 CRIA INFRA
   ├─ Cria namespace
   ├─ Configura rede (obtém IP)
   └─ Monta volumes

4. 📦 PULL IMAGE
   ├─ Baixa imagem do container
   └─ Armazena localmente

5. ⚙️  CRIA CONTAINER
   ├─ Comunica com CRI/Runtime
   ├─ Cria container com configs
   └─ Aplica limites de recursos

6. 🚀 INICIA CONTAINER
   └─ Container começa a rodar

7. 💚 MONITORA
   ├─ Executa health checks
   ├─ Valida readiness
   └─ Reinicia se necessário

8. 📊 RELATA STATUS
   ├─ Envia status ao API-Server
   └─ Atualiza ETCD

```

### 4.2 Exemplo Visual: Pod Sendo Executado

```
┌─────────────────────────────────────────────┐
│  Pod nginx-app (Agendado no Nó-1)          │
└────────────┬────────────────────────────────┘
             │
    ┌────────▼────────┐
    │  Kubelet (Nó-1) │
    └────────┬────────┘
             │
    1️⃣  Vê novo pod via API-Server
             │
    2️⃣  Pod Manager armazena especificação
             │
    3️⃣  Volume Manager monta volumes (se houver)
             │
    4️⃣  CRI/Runtime: Pull image nginx:latest
             │
             ▼
    ┌──────────────────┐
    │ Registry          │
    │ nginx:latest      │
    │ (200MB)          │
    └────────┬─────────┘
             │ Download
             ▼
    5️⃣  CRI/Runtime: Cria container
    ├─ CPU limit: 500m
    ├─ Memory limit: 256Mi
    ├─ Port 80
    └─ Mount /data
             │
    6️⃣  CRI/Runtime: Inicia container
             │
             ▼
    ┌──────────────────┐
    │ Container nginx  │
    │ PID: 1234        │
    │ IP: 10.0.0.5     │
    │ Status: Running  │
    └─────────────────┘
             │
    7️⃣  Probe Manager: Executa readiness probe
    ├─ curl http://localhost/health
    ├─ Sucesso: Container ready
    └─ Pod entra em estado Running
             │
    8️⃣  Status Manager: Reporta ao API-Server
    └─ Pod status: Running

```

---

## 5. Monitoramento e Health Checks

### 5.1 Tipos de Probes

Kubelet executa 3 tipos de health checks:

### Liveness Probe

**Pergunta**: O container ainda está vivo?
**Se falhar**: Kubelet **reinicia** o container
**Uso**: Detectar containers travados

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-liveness
spec:
  containers:
  - name: app
    image: app:latest
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      initialDelaySeconds: 10  # Espera 10s antes de começar
      periodSeconds: 5          # Verifica a cada 5s
      failureThreshold: 3       # Falha após 3 tentativas

```

### Readiness Probe

**Pergunta**: O container está pronto para receber tráfego?
**Se falhar**: Pod é **removido do service** (não recebe tráfego)
**Uso**: Detectar quando container está inicializando

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-readiness
spec:
  containers:
  - name: app
    image: app:latest
    readinessProbe:
      exec:
        command:
        - /bin/check-health.sh
      initialDelaySeconds: 5
      periodSeconds: 10
      failureThreshold: 2

```

### Startup Probe

**Pergunta**: O container terminou de inicializar?
**Se falhar**: Kubelet **não executa** outras probes até sucesso
**Uso**: Apps que levam tempo para iniciar

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-slow-startup
spec:
  containers:
  - name: app
    image: app:latest
    startupProbe:
      tcpSocket:
        port: 8080
      failureThreshold: 30
      periodSeconds: 10
    livenessProbe:
      httpGet:
        path: /health
        port: 8080
      periodSeconds: 5

```

### 5.2 Tipos de Verificação

| Tipo | Descrição | Exemplo |
| --- | --- | --- |
| **httpGet** | Faz HTTP request | GET /health |
| **tcpSocket** | Tenta conectar TCP | Porta 8080 aberta? |
| **exec** | Executa comando | `/bin/check.sh` |
| **grpc** | Chamada gRPC | Check service |

### 5.3 Exemplo Prático: Ciclo de Probes

```
Container inicializa
         │
         ▼
    ┌─────────────────────┐
    │ Startup Probe       │ ← Verifica init
    │ (TCP 8080)          │
    └─────────────────────┘
         │ Sucesso após 3s
         ▼
    ┌─────────────────────┐
    │ Readiness Probe     │ ← Verifica pronto
    │ (GET /ready)        │
    └─────────────────────┘
         │ Sucesso
         ▼
    Pod entra em service (recebe tráfego)
         │
         ▼ (a cada 5s)
    ┌─────────────────────┐
    │ Liveness Probe      │ ← Verifica vivo
    │ (GET /health)       │
    └─────────────────────┘
         │ Falha 3 vezes
         ▼
    Container é reiniciado

```

---

## 6. Gerenciamento de Recursos

### 6.1 Requests vs Limits

Kubelet aplica restrições de recursos através de cgroups:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: resource-limited-app
spec:
  containers:
  - name: app
    image: app:latest
    resources:
      # Quantidade GARANTIDA ao container
      requests:
        cpu: 250m        # 0.25 cores de CPU
        memory: 128Mi    # 128 MB de memória

      # Limite MÁXIMO que pode usar
      limits:
        cpu: 500m        # Máximo 0.5 cores
        memory: 256Mi    # Máximo 256 MB

```

### 6.2 QoS Classes (Quality of Service)

Kubelet classifica pods em 3 classes de QoS:

```
┌─────────────────────────────────────────────┐
│  Pod com requests == limits                 │
│  → QoS: Guaranteed (Máxima Prioridade)      │
│  → Nunca é evicted                          │
│                                             │
│  resources:                                 │
│    requests: {cpu: 500m, memory: 256Mi}     │
│    limits: {cpu: 500m, memory: 256Mi}       │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Pod com requests < limits                  │
│  → QoS: Burstable (Prioridade Média)        │
│  → Evicted se houver sobrecarga             │
│                                             │
│  resources:                                 │
│    requests: {cpu: 250m, memory: 128Mi}     │
│    limits: {cpu: 500m, memory: 256Mi}       │
└─────────────────────────────────────────────┘

┌─────────────────────────────────────────────┐
│  Pod sem requests/limits                    │
│  → QoS: BestEffort (Menor Prioridade)       │
│  → Primeiro a ser evicted                   │
│                                             │
│  resources: {}                              │
│  (Nenhum recurso garantido)                 │
└─────────────────────────────────────────────┘

```

### 6.3 Eviction Policy (Quando Recursos Escassos)

Quando nó fica sem recursos, Kubelet **evita (deleta) pods** pela ordem:

```
Ordem de Eviction:
1. BestEffort pods (sem requests)
2. Burstable pods (usando mais que requests)
3. Guaranteed pods (só se realmente crítico)

```

---

## 7. Gerenciamento de Volumes

### 7.1 Tipos de Volumes Gerenciados

```
Kubelet gerencia:

┌────────────────────────────────┐
│ Volume Types                   │
├────────────────────────────────┤
│ • emptyDir (local, temporário) │
│ • configMap (configurações)    │
│ • secret (dados sensíveis)     │
│ • downwardAPI (metadata)       │
│ • hostPath (arquivo do host)   │
│ • persistentVolumeClaim (PVC)  │
│ • nfs, awsEBS, gcePersistentDisk
│ • E outros...                  │
└────────────────────────────────┘

```

### 7.2 Exemplo: Pod com Volumes

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-with-volumes
spec:
  containers:
  - name: app
    image: app:latest
    volumeMounts:
    - name: config          # ← Monta ConfigMap
      mountPath: /etc/config
    - name: logs            # ← Monta Volume Local
      mountPath: /var/log
    - name: secret-data     # ← Monta Secret
      mountPath: /secrets

  volumes:
  - name: config
    configMap:
      name: app-config

  - name: logs
    emptyDir: {}            # Volume vazio, deletado ao finalizar pod

  - name: secret-data
    secret:
      secretName: db-password

```

**O que Kubelet faz:**

1. Obtém ConfigMap `app-config` do API-Server
2. Cria diretório local `/var/log`
3. Obtém Secret `db-password` do API-Server
4. Monta tudo no container
5. Quando pod termina, remove volumes emptyDir

---

## 8. Registro do Nó

### 8.1 Heartbeat ao API-Server

Kubelet envia heartbeat regularmente:

```
Kubelet (Nó-1)
    │
    │ A cada 10s (default)
    │
    ▼
┌──────────────────────────────────────┐
│ Kube-API-Server                      │
│                                      │
│ Node status:                         │
│ - Name: nó-1                         │
│ - Status: Ready                      │
│ - CPU: 4 cores                       │
│ - Memory: 8Gi                        │
│ - Disk: 100Gi                        │
│ - Pods running: 15                   │
│ - Pods capacity: 110                 │
└──────────────────────────────────────┘
         │
         ▼ (Persiste em ETCD)

```

### 8.2 Node Readiness

Se Kubelet parar de enviar heartbeat:

```
Tempo 0s: Kubelet normal
    │ Heartbeat OK
    ▼
    Status: Ready

Tempo 40s: Kubelet desconecta
    │ Sem heartbeat
    ▼
    Status: NotReady (Node Controller detecta)

Tempo 5m: Continua sem heartbeat
    │ Node Controller toma ação
    ▼
    1. Mark node "NotReady"
    2. Node Controller evita pods
    3. ReplicaSet Controller cria pods em outro nó

```

---

## 9. Configuração do Kubelet

### 9.1 Arquivo de Configuração

```bash
kubelet \
  --kubeconfig=/etc/kubernetes/kubelet.conf \
  --node-name=nó-1 \
  --container-runtime=remote \
  --container-runtime-endpoint=unix:///run/containerd/containerd.sock \
  --cgroup-driver=systemd \
  --pod-manifest-path=/etc/kubernetes/manifests \
  --cluster-dns=10.0.0.10 \
  --cluster-domain=cluster.local \
  --allow-privileged=false \
  --max-pods=110 \
  --image-gc-high-threshold=85 \
  --image-gc-low-threshold=80 \
  --v=2

```

**Flags Importantes:**

- `-node-name` - Nome do nó no cluster
- `-container-runtime-endpoint` - Onde conectar ao runtime
- `-pod-manifest-path` - Pods estáticos (lê do disco)
- `-cluster-dns` - Servidor DNS do cluster
- `-max-pods` - Máximo de pods no nó
- `-image-gc-high-threshold` - Quando limpar imagens (85%)

### 9.2 KubeletConfiguration via YAML

```yaml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
nodeStatusUpdateFrequency: 10s
nodeStatusReportFrequency: 1m0s
address: 0.0.0.0
port: 10250
readOnlyPort: 0
serverTLSBootstrap: true
tlsCipherSuites:
- TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
maxPods: 110
podsPerCore: 0
resyncFrequency: 1m0s
fileCheckFrequency: 20s
healthzPort: 10248
healthzBindAddress: 127.0.0.1
makeIPTablesUtilChains: true
iptablesMasqueradeBit: 14
iptablesDropBit: 15
featureGates: {}
failSwapOn: true
memorySwap: {}
containerLogMaxSize: 10Mi
containerLogMaxFiles: 5
eventRecordQPS: 5
eventBurst: 10
kubeReserved: {}
systemReserved: {}
hardEvictionThresholds:
- memory.available<100Mi
- nodefs.available<10%
softEvictionThresholds:
- memory.available<500Mi
softEvictionGracePeriod:
- memory.available=1m30s
imagefs:
  inodesFree: 5%

```

---

## 10. Static Pods

### 10.1 O que são Static Pods?

Static Pods são definidos em arquivos YAML no nó e gerenciados **diretamente pelo Kubelet**, sem necessidade do API-Server.

```bash
# Arquivos estáticos em:
/etc/kubernetes/manifests/

# Kubelet monitora esta pasta e executa qualquer Pod YAML

```

### 10.2 Uso Comum: Control Plane

```
/etc/kubernetes/manifests/
├── etcd.yaml           ← Static Pod do ETCD
├── kube-apiserver.yaml ← Static Pod do API-Server
├── kube-controller-manager.yaml ← Static Pod do Controller Manager
└── kube-scheduler.yaml ← Static Pod do Scheduler

```

**Por que?** O control plane precisa rodar mesmo antes do cluster estar 100% funcional.

### 10.3 Exemplo: Static Pod

```yaml
# /etc/kubernetes/manifests/my-app.yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-app-static
spec:
  containers:
  - name: app
    image: app:latest
    ports:
    - containerPort: 8080

```

Kubelet automaticamente:

1. Lê arquivo
2. Cria pod
3. Monitora
4. Reinicia se necessário
5. Se arquivo for deletado, pod é deletado

---

## 11. Container Runtime Interface (CRI)

### 11.1 Como Kubelet Comunica com Runtime

```
┌─────────────┐
│   Kubelet   │
│             │
│ Quer criar: │
│ Container   │
│ nginx:1.21  │
└──────┬──────┘
       │ gRPC (CRI)
       │
       ▼
┌─────────────────────────────────────┐
│  Container Runtime Interface (CRI)  │
│  Socket: unix:///run/containerd...  │
└─────────────────────────────────────┘
       │
       ▼ (Traduz para)
┌─────────────────────────┐
│  Container Runtime      │
│  • containerd           │
│  • cri-o                │
│  • Docker (via cri)     │
└─────────────────────────┘
       │
       ▼
    Cria container realmente

```

### 11.2 Operações CRI Comuns

```
kubelet chama:
├─ CreateContainer()    ← Cria novo container
├─ StartContainer()     ← Inicia container
├─ StopContainer()      ← Para container
├─ RemoveContainer()    ← Deleta container
├─ ListContainers()     ← Lista containers
├─ GetContainerStatus() ← Status do container
├─ ExecSync()          ← Executa comando no container
└─ PullImage()         ← Baixa imagem do registry

```

---

## 12. Monitoramento e Troubleshooting

### 12.1 Ver Status do Nó

```bash
# Ver status do nó
kubectl get nodes

# Ver detalhes do nó
kubectl describe node nó-1

# Ver recursos do nó
kubectl top nodes

# Ver pods no nó
kubectl get pods -o wide | grep nó-1

```

### 12.2 Ver Logs do Kubelet

```bash
# Logs do kubelet
sudo journalctl -u kubelet -f

# Logs em arquivo
tail -f /var/log/pods/*/*/kubelet.log

# Kubectl logs de pod
kubectl logs -n kube-system <pod-name>

```

### 12.3 Problemas Comuns

### Problema 1: Node is NotReady

```
$ kubectl get nodes
NAME    STATUS     ROLES   AGE
nó-1    NotReady   master  5d

```

**Causa possível**: Kubelet parou ou não consegue conectar ao API-Server
**Solução**:

```bash
# Verificar kubelet
sudo systemctl status kubelet

# Ver logs
sudo journalctl -u kubelet -n 50

# Reiniciar
sudo systemctl restart kubelet

```

### Problema 2: Pod Stuck in Pending

```
$ kubectl describe pod meu-pod
Status: Pending
Events:
  FailedScheduling: no nodes available

```

**Causa**: Kubelet indisponível ou falta de recursos
**Solução**: Verificar status de nós com `kubectl get nodes`

### Problema 3: Pod Stuck in ImagePullBackOff

```
$ kubectl describe pod meu-pod
Status: ImagePullBackOff
Events:
  Failed to pull image "app:typo": image not found

```

**Causa**: Imagem não existe ou typo no nome
**Solução**: Corrigir nome da imagem

### Problema 4: Container Restarting Continuously

```
$ kubectl get pods
NAME      RESTARTS
app-pod   5 (5m ago)

```

**Causa**: Liveness probe falhando, ou aplicação quebrando
**Solução**: Verificar logs com `kubectl logs <pod> --previous`

### 12.4 Comando Útil: exec (Debugar Container)

```bash
# Executar comando em container
kubectl exec -it meu-pod -- /bin/bash

# Executar comando específico
kubectl exec meu-pod -- ps aux

# Em pod com múltiplos containers
kubectl exec -it meu-pod -c container-name -- /bin/bash

```

---

## 13. Fluxo Completo: Pod Do Começo Ao Fim

### 13.1 Vida do Pod: Passo a Passo

```
1. Usuário cria Pod
   $ kubectl apply -f pod.yaml

   ↓

2. Kube-API-Server recebe e persiste no ETCD

   ↓

3. Kube Scheduler:
   ├─ Encontra pod sem nó
   ├─ Escolhe nó-1
   └─ Atribui pod ao nó-1

   ↓

4. Kubelet (nó-1):
   ├─ Watch API-Server detecta novo pod
   ├─ Pod Manager armazena especificação
   ├─ Volume Manager monta volumes (se houver)
   ├─ CRI: Pull image
   ├─ CRI: Create container
   ├─ CRI: Start container
   ├─ Pod status = ContainerCreating
   ├─ Startup Probe: Verifica se iniciou
   ├─ Pod status = Running
   ├─ Readiness Probe: Verifica se pronto
   ├─ Pod status = Ready (pode receber tráfego)
   └─ Status Manager: Reporta ao API-Server

   ↓

5. Kube Controller Manager:
   ├─ Detecta pod criado
   └─ Service Controller adiciona pod ao service

   ↓

6. Pod Recebendo Tráfego
   ├─ Liveness Probe executado a cada 5s
   ├─ Container monitora logs
   └─ Kubelet continua observando

   ↓

7. Usuário Deleta Pod
   $ kubectl delete pod meu-pod

   ↓

8. Kube-API-Server marca para deleção

   ↓

9. Kubelet (nó-1):
   ├─ Detecta marcação de deleção
   ├─ Inicia graceful shutdown (30s default)
   ├─ Container recebe SIGTERM
   ├─ Espera container finalizar
   ├─ Se não finalizar em 30s, SIGKILL
   ├─ CRI: Stop container
   ├─ CRI: Remove container
   ├─ Volume Manager unmount volumes
   └─ Status Manager: Reporta deleção

   ↓

10. Pod Removido do ETCD

```

---

## 14. Resumo das Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Execução** | Executar containers via CRI |
| **Monitoramento** | Health checks e status |
| **Recursos** | Limites de CPU/memória via cgroups |
| **Volumes** | Montar e gerenciar volumes |
| **Comunicação** | Relatar status ao API-Server |
| **Probes** | Liveness, readiness, startup |
| **Cleanup** | Remover containers quando necessário |
| **Node Reg** | Registrar-se no cluster |

---

## 15. Comparação: Kubelet vs Outros Componentes

```
┌──────────────────────────────────────────────────────┐
│  Componente        │  Localização  │  Função        │
├──────────────────────────────────────────────────────┤
│  Kube-API-Server   │  Master       │  Comunicação   │
│  Controller Manager│  Master       │  Orquestração  │
│  Kube-Scheduler    │  Master       │  Placement     │
│  Kubelet           │  Cada Nó      │  Execução      │ ← Aqui!
│  Kube-Proxy        │  Cada Nó      │  Networking    │
└──────────────────────────────────────────────────────┘

```

---

## 16. Diagrama Completo da Orquestração

```
┌────────────────────────────────────────────────────────┐
│  1. Usuário: kubectl apply -f pod.yaml                 │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  2. Kube-API-Server                                    │
│     └─ Salva Pod no ETCD                               │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  3. Kube-Scheduler                                     │
│     └─ Atribui pod ao nó-1                             │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  4. Kubelet (nó-1)  ← VOCÊ ESTÁ AQUI                  │
│                                                        │
│     Watch: Detecta novo pod                           │
│     ↓                                                  │
│     Volume Manager: Monta volumes                     │
│     ↓                                                  │
│     CRI: Pull image                                   │
│     ↓                                                  │
│     CRI: Create + Start container                     │
│     ↓                                                  │
│     Probe Manager: Executa health checks              │
│     ↓                                                  │
│     Status Manager: Reporta ao API-Server             │
│     ↓                                                  │
│     Continua monitorando container                    │
└────────┬───────────────────────────────────────────────┘
         │
┌────────▼───────────────────────────────────────────────┐
│  5. Container Rodando                                  │
│     Application executando no nó                       │
└────────────────────────────────────────────────────────┘

```

---

## 17. Conclusão

- **Kubelet** é o agente que executa containers em cada nó
- **Responsável por monitorar** saúde e estado dos containers
- **Comunica com API-Server** para receber instruções e relatar status
- **Gerencia recursos** (CPU, memória) via cgroups
- **Executa health checks** (liveness, readiness, startup)
- **Gerencia volumes** e armazenamento
- É o **ponto de execução real** do Kubernetes

---

# Kube Proxy - Guia Completo

## 1. O que é Kube Proxy?

**Kube Proxy** é um componente que roda em **cada nó do cluster Kubernetes**. É responsável por **manter regras de rede** que permitem comunicação entre pods e serviços. Ele implementa o modelo de rede do Kubernetes.

### 1.1 Analogia

Imagine o Kube Proxy como um **porteiro de switchboard de telefone**:

- 📞 Cada chamada (requisição) chega ao porteiro
- 📋 Ele consulta a lista de ramais (endpoints)
- 🔄 Redireciona a chamada para o ramal correto
- 🔁 Distribui chamadas entre múltiplos ramais
- ☎️ Se um ramal cair, redireciona para outro

---

## 2. Responsabilidades Principais

### 2.1 Implementação de Serviços

- **Criar regras de rede** que mapeiam Service para Pods
- **Permitir comunicação** entre pods
- **Expor serviços** dentro do cluster

### 2.2 Load Balancing

- **Distribuir tráfego** entre múltiplos pods
- **Balancear conexões** entre endpoints
- **Round-robin** ou outras estratégias

### 2.3 Network Translation

- **Traducir endereços IP** (SNAT/DNAT)
- **Reescrever portas** quando necessário
- **Manter estado de conexão** (stateful)

### 2.4 Service Discovery

- **Descobrir endpoints** (pods que implementam o service)
- **Monitorar mudanças** (pods adicionados/removidos)
- **Atualizar regras** dinamicamente

### 2.5 Roteamento de Tráfego

- **Rotear tráfego** para o pod correto
- **Garantir conectividade** dentro do cluster
- **Isolar tráfego** entre namespaces (via policy)

---

## 3. Problema que Kube Proxy Resolve

### 3.1 Sem Kube Proxy (Não Funciona)

```
┌─────────────────────────────────────────┐
│  Pod A (10.0.0.5)                       │
│                                         │
│  Quer conectar ao serviço "web"         │
│  que tem 3 pods:                        │
│  - Pod B (10.0.0.10)                    │
│  - Pod C (10.0.0.11)                    │
│  - Pod D (10.0.0.12)                    │
└────────────┬────────────────────────────┘
             │
    Pod A tenta conectar em 10.0.0.10
             │
    ✗ Problema: Qual IP usar?
    ✗ Se usar 10.0.0.10, sempre vai para Pod B
    ✗ Se Pod B cai, ninguém fica sabendo
    ✗ Sem abstração de serviço

```

### 3.2 Com Kube Proxy (Funciona)

```
┌─────────────────────────────────────────┐
│  Pod A (10.0.0.5)                       │
│                                         │
│  Quer conectar ao serviço "web"         │
│  (Service IP: 10.96.0.5)                │
└────────────┬────────────────────────────┘
             │
    Pod A conecta em 10.96.0.5 (Service IP)
             │
    ┌────────▼──────────────────────────┐
    │  Kube Proxy (no mesmo nó)         │
    │                                  │
    │  "10.96.0.5 é um serviço!"        │
    │  Encontra endpoints:              │
    │  - 10.0.0.10 (Pod B)              │
    │  - 10.0.0.11 (Pod C)              │
    │  - 10.0.0.12 (Pod D)              │
    │                                  │
    │  Round-robin:                     │
    │  1ª req → 10.0.0.10               │
    │  2ª req → 10.0.0.11               │
    │  3ª req → 10.0.0.12               │
    │  4ª req → 10.0.0.10 (repete)      │
    └────────┬──────────────────────────┘
             │
    ✓ Abstração de serviço
    ✓ Load balancing automático
    ✓ Tolerância a falhas
    ✓ Escalabilidade

```

---

## 4. Arquitetura do Kube Proxy

### 4.1 Componentes Internos

```
┌──────────────────────────────────────────────────────────┐
│              Kube Proxy (em cada nó)                     │
│                                                          │
│  ┌────────────────────────────────────────────────────┐ │
│  │  1. API Server Watcher                             │ │
│  │     - Monitora Services                            │ │
│  │     - Monitora Endpoints (pods que implementam)    │ │
│  │     - Detecta mudanças                             │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  2. Service Manager                               │ │
│  │     - Rastreia serviços do cluster                │ │
│  │     - Mapeia Service → Endpoints                  │ │
│  │     - Configura load balancing                    │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  3. Proxier (Implementação de Proxy)              │ │
│  │                                                   │ │
│  │     Pode ser:                                    │ │
│  │     ├─ iptables (mais comum)                     │ │
│  │     ├─ ipvs (performance)                        │ │
│  │     ├─ kernelspace (eficiente)                   │ │
│  │     └─ userspace (legado, lento)                 │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  4. Conntrack Manager                             │ │
│  │     - Mantém estado de conexões                   │ │
│  │     - Rastreia fluxo de pacotes                   │ │
│  │     - Garbage collection de conexões velhas       │ │
│  └────────────────────────────────────────────────────┘ │
│                         │                               │
│  ┌──────────────────────▼────────────────────────────┐ │
│  │  5. Network Rules Engine                          │ │
│  │     - iptables/ipvs rules                         │ │
│  │     - Firewall rules                              │ │
│  │     - Masquerading (SNAT)                         │ │
│  │     - Destination NAT (DNAT)                      │ │
│  └────────────────────────────────────────────────────┘ │
│                                                          │
└──────────────────────────────────────────────────────────┘

```

---

## 5. Modos de Funcionamento do Kube Proxy

### 5.1 Userspace Mode (Legado, Lento)

```
Pod A ──┐
        │ Conexão
        ▼
    Kernel (iptables rule)
        │
        ▼
    Kube Proxy (userspace)  ← Processo em userspace
        │ Cópia de dados entre kernel e userspace
        │ (Lento!)
        ▼
    Kernel
        │
        ▼
Pod B (Endpoint)

❌ Lento: Cópia de dados kernel ↔ userspace
❌ Overhead: Mudança de contexto
✓ Compatibilidade: Funciona em qualquer SO

```

### 5.2 Iptables Mode (Padrão, Rápido)

```
Pod A ──┐
        │ Conexão
        ▼
    Kernel iptables rules  ← Tudo no kernel (Fast!)
    ├─ Match: destino 10.96.0.5?
    ├─ Sim: Load balancing
    ├─ Escolhe endpoint: 10.0.0.10
    ├─ DNAT (muda destino)
    │
        ▼
Pod B (10.0.0.10)

✓ Rápido: Tudo no kernel
✓ Sem userspace: Sem overhead
✓ Padrão em Kubernetes
❌ Escalabilidade: Muitas regras = lento

```

### 5.3 IPVS Mode (Melhor Performance)

```
Pod A ──┐
        │ Conexão
        ▼
    Kernel IPVS (Linux Virtual Server)
    ├─ Usa hash table (O(1))
    ├─ Load balancing eficiente
    ├─ Suporta algoritmos avançados
    ├─ Escalável para muitos serviços
    │
        ▼
Pod B (Endpoint)

✓ Muito Rápido: Hash table em kernel
✓ Escalável: Suporta milhares de serviços
✓ Algoritmos: Round-robin, LRU, etc
✓ Stateless: Não precisa de conntrack
❌ Requer IPVS kernel module

```

### 5.4 Comparação dos Modos

| Modo | Velocidade | Escalabilidade | Complexidade | Padrão |
| --- | --- | --- | --- | --- |
| **Userspace** | Lenta | Baixa | Simples | ❌ |
| **Iptables** | Média | Média | Média | ✅ |
| **IPVS** | Rápida | Alta | Alta | ⭐ Recomendado |

---

## 6. Como Kube Proxy Funciona: Fluxo de Tráfego

### 6.1 Cenário: Pod Acessando Service

```
Cluster:
├─ Pod A (10.0.0.5) - em Nó-1
├─ Pod B (10.0.0.10) - em Nó-2
├─ Pod C (10.0.0.11) - em Nó-2
└─ Pod D (10.0.0.12) - em Nó-3

Service "web":
├─ Service IP (ClusterIP): 10.96.0.5
├─ Port: 80
├─ Endpoints: [10.0.0.10, 10.0.0.11, 10.0.0.12]

```

### 6.2 Passo a Passo: Pod A Conectando ao Service

```
1️⃣  Pod A inicia requisição
    $ curl web  (resolve para 10.96.0.5:80)

2️⃣  Pacote sai do Pod A
    Source: 10.0.0.5:xxxxx
    Dest: 10.96.0.5:80

3️⃣  Chega no Kube Proxy (Nó-1)

    Kube Proxy vê: "10.96.0.5:80 é um serviço!"

    Consulta endpoints do serviço "web"
    Endpoints: [10.0.0.10, 10.0.0.11, 10.0.0.12]

    Load balancing (Round-robin):
    Escolhe: 10.0.0.10 (Pod B)

4️⃣  DNAT (Destination NAT)

    Pacote original:
    ┌─────────────────────────────┐
    │ Source: 10.0.0.5:xxxxx      │
    │ Dest: 10.96.0.5:80          │
    └─────────────────────────────┘
              ↓ (DNAT)
    Pacote transformado:
    ┌─────────────────────────────┐
    │ Source: 10.0.0.5:xxxxx      │
    │ Dest: 10.0.0.10:8080        │ ← Endpoint real
    └─────────────────────────────┘

5️⃣  Pacote chega em Pod B

    Pod B responde:
    Source: 10.0.0.10:8080
    Dest: 10.0.0.5:xxxxx

6️⃣  SNAT (Source NAT) - Retorno

    Resposta original:
    ┌─────────────────────────────┐
    │ Source: 10.0.0.10:8080      │
    │ Dest: 10.0.0.5:xxxxx        │
    └─────────────────────────────┘
              ↓ (SNAT)
    Resposta transformada:
    ┌─────────────────────────────┐
    │ Source: 10.96.0.5:80        │ ← Serviço
    │ Dest: 10.0.0.5:xxxxx        │
    └─────────────────────────────┘

7️⃣  Pod A recebe resposta

    Parece que veio de 10.96.0.5 (o serviço)
    Pod A não sabe que realmente veio de 10.0.0.10

```

---

## 7. Services e Endpoints

### 7.1 Service: Abstração de Rede

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP           # ← Tipo de serviço
  selector:
    app: web               # ← Seleciona pods com esta label
  ports:
  - port: 80               # ← Porta do serviço
    targetPort: 8080       # ← Porta no container
    protocol: TCP

```

**O que Kube Proxy faz:**

1. Obtém label selector `app=web`
2. Encontra todos os pods com esta label
3. Cria endpoint para cada pod encontrado
4. Configura regras de rede (DNAT/SNAT)

### 7.2 Endpoints: IPs dos Pods

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: web              # ← Nome do serviço
subsets:
- addresses:
  - ip: 10.0.0.10        # ← Pod B
    targetRef:
      kind: Pod
      name: pod-b
      namespace: default
  - ip: 10.0.0.11        # ← Pod C
    targetRef:
      kind: Pod
      name: pod-c
  - ip: 10.0.0.12        # ← Pod D
    targetRef:
      kind: Pod
      name: pod-d
  ports:
  - port: 8080           # ← Porta no container
    protocol: TCP

```

**Atualização Dinâmica:**

- Pod criado → Endpoint adicionado automaticamente
- Pod deletado → Endpoint removido automaticamente
- Pod falha → Removed from endpoints (readiness probe)

---

## 8. Tipos de Services

### 8.1 ClusterIP (Padrão)

```
┌────────────────────────────────────────┐
│  ClusterIP Service                     │
├────────────────────────────────────────┤
│ Service IP (Virtual): 10.96.0.5        │
│ Apenas acessível dentro do cluster     │
│                                        │
│ Pod dentro do cluster:                 │
│ $ curl web  (10.96.0.5:80)     ✓       │
│                                        │
│ Fora do cluster:                       │
│ $ curl 10.96.0.5  (impossível)  ✗      │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: ClusterIP
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080

```

### 8.2 NodePort

```
┌────────────────────────────────────────┐
│  NodePort Service                      │
├────────────────────────────────────────┤
│ Service IP: 10.96.0.5                  │
│ Port: 80 (no service)                  │
│ NodePort: 30080 (em cada nó)           │
│                                        │
│ De dentro do cluster:                  │
│ $ curl web:80  ✓                       │
│ $ curl 10.96.0.5:80  ✓                 │
│                                        │
│ De fora do cluster:                    │
│ $ curl <node-ip>:30080  ✓              │
│ $ curl nó-1-ip:30080   (vai até pod)   │
│ $ curl nó-2-ip:30080   (vai até pod)   │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: NodePort
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080
    nodePort: 30080    # ← Porta em cada nó (30000-32767)

```

### 8.3 LoadBalancer

```
┌────────────────────────────────────────┐
│  LoadBalancer Service                  │
├────────────────────────────────────────┤
│ Service IP: 10.96.0.5 (interno)        │
│ External IP: 203.0.113.5 (fornecedor)  │
│                                        │
│ De dentro:                             │
│ $ curl web:80  ✓                       │
│                                        │
│ De fora:                               │
│ $ curl 203.0.113.5:80  ✓               │
│                                        │
│ Fluxo:                                 │
│ Internet → Load Balancer (nuvem)       │
│           → NodePort em cada nó        │
│           → Service → Pods             │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080

```

### 8.4 ExternalName

```
┌────────────────────────────────────────┐
│  ExternalName Service                  │
├────────────────────────────────────────┤
│ Mapeia nome de serviço para DNS externo│
│                                        │
│ $ curl database (dentro do cluster)    │
│  → CNAME database.external.example.com │
│  → Conecta a banco externo             │
└────────────────────────────────────────┘

```

**Exemplo:**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: database
spec:
  type: ExternalName
  externalName: database.external.example.com

```

---

## 9. Service Discovery

### 9.1 DNS (Recomendado)

Kubernetes inclui um CoreDNS que resolve nomes automaticamente:

```bash
# Pod dentro do cluster pode acessar:

# Serviço no mesmo namespace
curl web           # → 10.96.0.5

# Serviço em outro namespace
curl web.payments  # → 10.96.0.100

# Nome completo
curl web.payments.svc.cluster.local
# → 10.96.0.100

```

**Como funciona:**

```
1. Pod quer conectar em "web"
2. Kubelet injeta CoreDNS como resolver
3. Consulta CoreDNS (10.0.0.10:53)
4. CoreDNS consulta ETCD
5. Retorna IP: 10.96.0.5
6. Pod conecta em 10.96.0.5
7. Kube Proxy faz load balancing

```

### 9.2 Environment Variables (Legado)

Kubelet injeta variáveis de ambiente:

```bash
# Dentro do container
$ env | grep SERVICE

WEB_SERVICE_HOST=10.96.0.5
WEB_SERVICE_PORT=80

# Útil para apps legadas

```

---

## 10. Iptables Rules (Exemplo Detalhado)

### 10.1 Como Kube Proxy Usa Iptables

```
Quando você cria:
apiVersion: v1
kind: Service
metadata:
  name: web
spec:
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 8080

Kube Proxy cria regras iptables:

# Cadeia de entrada
*nat
:PREROUTING ACCEPT
:KUBE-SERVICES - [0:0]

# Se pacote vai para Service IP
-A PREROUTING -m comment --comment "service"
  -j KUBE-SERVICES

# Service IP 10.96.0.5:80 → Endpoints
-A KUBE-SERVICES -d 10.96.0.5/32
  -m comment --comment "web"
  -m tcp -p tcp --dport 80
  -j KUBE-SVC-XXXXXXXXXXXX

# Endpoints (Round-robin com probability)
-A KUBE-SVC-XXXXXXXXXXXX -m statistic
  --mode random --probability 0.3333
  -j KUBE-SEP-YYYYYY  # Pod B (10.0.0.10:8080)
-A KUBE-SVC-XXXXXXXXXXXX -m statistic
  --mode random --probability 0.5000
  -j KUBE-SEP-ZZZZZZ  # Pod C (10.0.0.11:8080)
-A KUBE-SVC-XXXXXXXXXXXX
  -j KUBE-SEP-WWWWWW  # Pod D (10.0.0.12:8080)

# DNAT (muda destino)
-A KUBE-SEP-YYYYYY -m comment --comment "web"
  -m tcp -p tcp
  -j DNAT --to-destination 10.0.0.10:8080

```

### 10.2 Ver Regras Iptables

```bash
# Ver todas as regras NAT
sudo iptables -t nat -L -n -v

# Ver regras de um serviço específico
sudo iptables -t nat -L KUBE-SERVICES -n -v

# Ver estatísticas
sudo iptables -t nat -L -n -v -x

```

---

## 11. Configuração do Kube Proxy

### 11.1 Escolher Modo

```bash
kube-proxy \
  --kubeconfig=/etc/kubernetes/kube-proxy.conf \
  --proxy-mode=iptables \  # ou ipvs, userspace
  --cluster-cidr=10.0.0.0/8 \
  --masquerade-all=false \
  --v=2

```

### 11.2 Configuração via YAML

```yaml
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
bindAddress: 0.0.0.0
clientConnection:
  kubeconfig: /etc/kubernetes/kube-proxy.conf
clusterCIDR: 10.0.0.0/8
configSyncPeriod: 15m0s
conntrack:
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
detectLocalMode: ClusterCIDR
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
mode: iptables  # ou ipvs
nodePortAddresses: null
metricsBindAddress: 127.0.0.1:10249
udpIdleTimeout: 250ms

```

---

## 12. Monitoramento e Troubleshooting

### 12.1 Ver Status do Kube Proxy

```bash
# Ver pods do kube-proxy
kubectl get pods -n kube-system -l k8s-app=kube-proxy

# Ver logs
kubectl logs -n kube-system -l k8s-app=kube-proxy

# Ver métricas
curl localhost:10249/metrics

```

### 12.2 Diagnosticar Problemas de Conectividade

### Problema 1: Não consegue Conectar ao Service

```bash
# Verificar se service existe
kubectl get svc web

# Ver endpoints do service
kubectl get endpoints web

# Verificar IP do service
kubectl get svc web -o jsonpath='{.spec.clusterIP}'

# Testar conectividade
kubectl run -it --rm test --image=curl --restart=Never \
  -- curl web:80

# Ver logs do service
kubectl describe svc web

```

### Problema 2: Kube Proxy Não Funciona

```bash
# Verificar se kube-proxy está rodando
kubectl get pods -n kube-system kube-proxy-<node>

# Ver logs
kubectl logs -n kube-system kube-proxy-<node>

# Reiniciar
kubectl delete pod -n kube-system kube-proxy-<node>

# Ver regras iptables
sudo iptables -t nat -L KUBE-SERVICES -n

```

### Problema 3: Service IP Não Resolve

```bash
# Verificar CoreDNS
kubectl get pods -n kube-system -l k8s-app=kube-dns

# Testar DNS
kubectl run -it --rm test --image=alpine --restart=Never \
  -- nslookup web

# Ver logs do DNS
kubectl logs -n kube-system -l k8s-app=kube-dns

```

### 12.3 Comandos Úteis de Debug

```bash
# Entrar em pod e debugar
kubectl exec -it <pod> -- /bin/bash

# Ver conectividade de um pod
kubectl run -it --rm debug --image=alpine --restart=Never \
  -- sh

# Dentro do debug container:
# Testar DNS
nslookup web
nslookup web.default.svc.cluster.local

# Testar conectividade
curl -v web:80

# Ver iptables (se em host)
sudo iptables -t nat -L -n | grep SERVICE

```

---

## 13. Network Policies (Segurança)

### 13.1 Limitando Tráfego com NetworkPolicy

Kube Proxy respeita Network Policies para filtrar tráfego:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-from-frontend
spec:
  podSelector:
    matchLabels:
      app: web    # ← Aplica a pods com esta label

  policyTypes:
  - Ingress      # ← Tráfego de entrada

  ingress:
  - from:
        # Apenas de pods com label tier=frontend
        - podSelector:
            matchLabels:
              tier: frontend
    ports:
    - protocol: TCP
      port: 80

```

**Efeito:**

```
Tráfego permitido:
Frontend Pod → Web Pod ✓

Tráfego bloqueado:
Database Pod → Web Pod ✗
External → Web Pod ✗

```

---

## 14. Performance e Otimizações

### 14.1 IPVS vs Iptables

```
Iptables Mode:
┌──────────────────┐
│ 10 serviços      │ Rápido
│ 100 pods         │ Muitas regras (1000+)
└──────────────────┘

IPVS Mode:
┌──────────────────┐
│ 1000 serviços    │ Ainda rápido
│ 10000 pods       │ Hash table (O(1))
└──────────────────┘

```

### 14.2 Otimizações

```bash
# 1. Use IPVS para clusters grandes
kube-proxy --proxy-mode=ipvs

# 2. Ajuste conntrack
--conntrack-max=262144

# 3. Use session affinity se necessário
apiVersion: v1
kind: Service
spec:
  sessionAffinity: ClientIP  # ← Mesmo cliente → mesmo pod
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 10800

```

---

## 15. Fluxo Completo do Cluster

### 15.1 Requisição de Pod A para Service

```
┌─────────────────────────────────────────────────────────┐
│  1. Pod A quer acessar service "web"                    │
│     curl web                                            │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  2. CoreDNS Resolve Nome                               │
│     web → 10.96.0.5 (Service IP)                       │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  3. Pod A Conecta em 10.96.0.5:80                      │
│     Pacote: Src=10.0.0.5, Dst=10.96.0.5:80            │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  4. Kube Proxy (iptables mode)                         │
│     ├─ Detecta: 10.96.0.5:80 é um serviço             │
│     ├─ Encontra endpoints: [10.0.0.10, 10.0.0.11, ...]│
│     ├─ Load balancing (round-robin)                    │
│     ├─ Escolhe: 10.0.0.10 (Pod B)                      │
│     └─ DNAT: Reescreve Dst para 10.0.0.10:8080        │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  5. Pacote Chega em Pod B                              │
│     Vê: Src=10.0.0.5, Dst=10.0.0.10:8080              │
│     Processa requisição                                 │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  6. Pod B Responde                                     │
│     Pacote: Src=10.0.0.10:8080, Dst=10.0.0.5          │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  7. Kube Proxy SNAT (Retorno)                          │
│     Reescreve Src: 10.96.0.5 (Service IP)             │
│     Pacote: Src=10.96.0.5:80, Dst=10.0.0.5            │
└────────┬────────────────────────────────────────────────┘
         │
┌────────▼────────────────────────────────────────────────┐
│  8. Pod A Recebe Resposta                              │
│     Vê como se veio de 10.96.0.5 (o serviço)          │
│     Não sabe que realmente veio de 10.0.0.10           │
└─────────────────────────────────────────────────────────┘

```

---

## 16. Comparação: Service Discovery Methods

| Método | Velocidade | Flexibilidade | Uso |
| --- | --- | --- | --- |
| **DNS** | Rápida | Alta | Padrão, recomendado |
| **Env Vars** | Muito Rápida | Baixa | Apps legadas |
| **API Direct** | Média | Muito Alta | Casos especiais |

---

## 17. Resumo das Responsabilidades

| Responsabilidade | Descrição |
| --- | --- |
| **Load Balancing** | Distribui tráfego entre endpoints |
| **Service IP** | Cria IP virtual para cada serviço |
| **DNS** | Resolve nomes de serviços |
| **NAT** | DNAT/SNAT para tradução de endereços |
| **Networking** | Conecta pods dentro do cluster |
| **Monitoramento** | Atualiza regras quando pods mudam |
| **Policy** | Respeita NetworkPolicies |

---

## 18. Conclusão

- **Kube Proxy** é o "porteiro de switchboard" do cluster
- **Implementa o modelo de rede** do Kubernetes
- **Cria abstrações de serviço** (Service IPs)
- **Faz load balancing** entre pods automaticamente
- **Usa NAT** (DNAT/SNAT) para tradução de endereços
- **3 modos**: Userspace (lento), Iptables (padrão), IPVS (rápido)
- **Trabalha com Kubelet** para descobrir endpoints
- É **essencial** para comunicação entre pods

---

# Pod no Kubernetes - Resumo para Estudos CKA

## O que é um Pod?

Um **Pod** é a menor unidade implantável no Kubernetes. É um wrapper ao redor de um ou mais containers que rodam juntos na mesma rede.

## Características Principais

**Containers por Pod**

- Geralmente um container por Pod (padrão recomendado)
- Pode ter múltiplos containers, mas eles compartilham recursos
- Containers no mesmo Pod se comunicam via `localhost`

**Networking**

- Cada Pod tem um IP único
- Compartilham o mesmo namespace de rede
- Comunicação via localhost entre containers

**Storage**

- Volumes podem ser montados em múltiplos containers
- Dados persistem enquanto o Pod existe

## Definição em YAML

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: meu-pod
  namespace: default
spec:
  containers:
  - name: nginx
    image: nginx:latest
    ports:
    - containerPort: 80
    env:
    - name: ENV_VAR
      value: "valor"
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
    volumeMounts:
    - name: config
      mountPath: /etc/config
  volumes:
  - name: config
    configMap:
      name: meu-config

```

## Ciclo de Vida de um Pod

| Estado | Descrição |
| --- | --- |
| **Pending** | Pod foi criado mas não está rodando (esperando recursos) |
| **Running** | Containers estão rodando |
| **Succeeded** | Pod completou com sucesso (batch jobs) |
| **Failed** | Um ou mais containers falharam |
| **Unknown** | Estado desconhecido |

## Comandos Essenciais para CKA

```bash
# Criar um Pod
kubectl run nginx --image=nginx

# Criar e gerar YAML
kubectl run nginx --image=nginx --dry-run=client -o yaml

# Listar Pods
kubectl get pods
kubectl get pods -n default
kubectl get pods -o wide  # mostra IPs e nós

# Descrever um Pod
kubectl describe pod nginx

# Ver logs
kubectl logs nginx
kubectl logs nginx -c container-name  # múltiplos containers

# Executar comando no Pod
kubectl exec -it nginx -- /bin/bash

# Editar Pod em tempo real
kubectl edit pod nginx

# Deletar Pod
kubectl delete pod nginx
kubectl delete pod nginx --grace-period=0 --force  # force delete

# Port forward
kubectl port-forward nginx 8080:80

```

## Multi-Container Pod

Usado quando containers precisam trabalhar juntos (sidecar pattern):

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
  - name: app
    image: app:latest
    ports:
    - containerPort: 8080
  - name: sidecar
    image: sidecar:latest
    volumeMounts:
    - name: shared
      mountPath: /shared
  volumes:
  - name: shared
    emptyDir: {}

```

## Init Containers

Containers que rodam antes dos containers principais:

```yaml
spec:
  initContainers:
  - name: init-db
    image: busybox
    command: ['sh', '-c', 'echo Inicializando DB']
  containers:
  - name: app
    image: app:latest

```

## Resources (CPU/Memory)

```yaml
resources:
  requests:  # mínimo garantido
    memory: "64Mi"
    cpu: "250m"
  limits:    # máximo permitido
    memory: "128Mi"
    cpu: "500m"

```

## Health Checks

```yaml
livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10

readinessProbe:
  httpGet:
    path: /ready
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5

```

---

# ReplicaSet, ReplicaSet Controller e Deployments - Resumo para Estudos CKA

## Hierarquia de Objetos

```
Deployment (nível alto - recomendado)
    ↓
ReplicaSet (gerenciado pelo Deployment)
    ↓
Pods (gerenciados pelo ReplicaSet)

```

---

## ReplicaSet

### O que é?

Um **ReplicaSet** garante que um número especificado de réplicas de um Pod estejam rodando em qualquer momento. É o responsável por manter o número desejado de Pods.

### Definição em YAML

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: meu-replicaset
  labels:
    app: nginx
spec:
  replicas: 3  # número de Pods desejados
  selector:
    matchLabels:
      app: nginx  # Pods com esta label serão gerenciados
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"

```

### Como Funciona

1. ReplicaSet monitora Pods com labels que correspondem ao `selector`
2. Se menos Pods existem que `replicas`: cria novos
3. Se mais Pods existem que `replicas`: deleta os extras
4. Se um Pod morre: cria um novo automaticamente
5. **Não faz rolling updates** (atualizações gradualmente)

### Seletor (Selector)

```yaml
selector:
  matchLabels:
    app: nginx
    version: v1

# Ou com matchExpressions (mais complexo)
selector:
  matchExpressions:
  - key: app
    operator: In
    values: [nginx, apache]
  - key: tier
    operator: NotIn
    values: [frontend]

```

### Comandos Essenciais

```bash
# Criar ReplicaSet
kubectl create -f replicaset.yaml

# Listar ReplicaSets
kubectl get replicaset
kubectl get rs  # forma curta

# Descrever ReplicaSet
kubectl describe rs meu-replicaset

# Escalar (mudar número de replicas)
kubectl scale rs meu-replicaset --replicas=5

# Editar ReplicaSet
kubectl edit rs meu-replicaset

# Deletar ReplicaSet (mantém Pods)
kubectl delete rs meu-replicaset
kubectl delete rs meu-replicaset --cascade=orphan

# Deletar ReplicaSet e Pods
kubectl delete rs meu-replicaset --cascade=background

```

---

## ReplicaSet Controller

### O que é?

O **ReplicaSet Controller** é um componente do Kubernetes (roda no control plane) que monitora continuamente os ReplicaSets e garante que o estado atual corresponda ao estado desejado.

### Responsabilidades

- **Monitorar Pods**: Verifica status de todos os Pods constantemente
- **Criar Pods**: Se houver menos Pods do que `replicas`, cria novos
- **Deletar Pods**: Se houver mais Pods, deleta o excesso
- **Recriar Pods**: Se um Pod falha, recria automaticamente
- **Lidar com Seletores**: Identifica quais Pods gerenciar via labels

### Como o Controller Sabe o que Fazer

```yaml
# O ReplicaSet define o estado desejado
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx

# O Controller:
# 1. Conta quantos Pods com label app=nginx existem
# 2. Compara com replicas: 3
# 3. Toma ação: criar/deletar Pods

```

### Fluxo de Reconciliação

```
Controller monitora
    ↓
Detecta mudança (Pod morreu, Deployment escalado, etc)
    ↓
Calcula: Pods atuais vs replicas desejadas
    ↓
Toma ação: criar ou deletar Pods
    ↓
Monitora novamente

```

### Status do ReplicaSet

```bash
kubectl describe rs meu-replicaset

# Saída:
# Replicas: 3 desired | 3 updated | 3 total | 3 available
#   desired: número definido no spec
#   updated: Pods com spec atualizado
#   total: número total de Pods
#   available: Pods prontos

```

---

## Deployments (RECOMENDADO!)

### O que é?

Um **Deployment** é um objeto de nível superior que gerencia ReplicaSets. Fornece atualizações declarativas para Pods e ReplicaSets com recursos como rolling updates, rollbacks e histórico.

### Quando Usar

- ✅ Quase sempre! (é a forma recomendada)
- ✅ Quando precisa de atualizações gradativas
- ✅ Quando quer manter histórico de versões
- ❌ Não use ReplicaSet diretamente na maioria dos casos

### Definição em YAML

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  strategy:
    type: RollingUpdate  # ou Recreate
    rollingUpdate:
      maxSurge: 1        # máximo de Pods além do desejado
      maxUnavailable: 1  # máximo de Pods indisponíveis
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80
        resources:
          requests:
            memory: "64Mi"
            cpu: "250m"
          limits:
            memory: "128Mi"
            cpu: "500m"
      terminationGracePeriodSeconds: 30

```

### Estratégias de Deploy

### RollingUpdate (Padrão)

```yaml
strategy:
  type: RollingUpdate
  rollingUpdate:
    maxSurge: 1           # máximo de Pods extras durante atualização
    maxUnavailable: 1     # máximo de Pods que podem estar down

```

Resultado: Pods são substituídos gradualmente, sem downtime

```
Atualização V1 → V2
V1: 3 Pods
→ Cria 1 Pod V2, deleta 1 Pod V1 (total: 3)
→ Cria 1 Pod V2, deleta 1 Pod V1 (total: 3)
→ Cria 1 Pod V2, deleta 1 Pod V1 (total: 3)
Resultado: 3 Pods V2

```

### Recreate

```yaml
strategy:
  type: Recreate

```

Resultado: Deleta todos os Pods antigos, depois cria novos (downtime!)

```
Atualização V1 → V2
→ Deleta todos os 3 Pods V1
→ Cria 3 Pods V2
(período com 0 Pods = DOWNTIME)

```

### Comandos Essenciais

```bash
# Criar Deployment
kubectl create -f deployment.yaml
kubectl create deployment nginx --image=nginx --replicas=3

# Listar Deployments
kubectl get deployments
kubectl get deploy

# Descrever Deployment
kubectl describe deploy nginx-deployment

# Ver status do rollout
kubectl rollout status deployment/nginx-deployment

# Ver histórico de rollouts
kubectl rollout history deployment/nginx-deployment

# Ver detalhes de uma versão específica
kubectl rollout history deployment/nginx-deployment --revision=2

# Escalar Deployment
kubectl scale deployment nginx-deployment --replicas=5

# Atualizar imagem (trigger de novo rollout)
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.16.0 --record

# Editar Deployment
kubectl edit deployment nginx-deployment

# Fazer rollback para versão anterior
kubectl rollout undo deployment/nginx-deployment

# Fazer rollback para versão específica
kubectl rollout undo deployment/nginx-deployment --to-revision=2

# Pausar rollout
kubectl rollout pause deployment/nginx-deployment

# Resumir rollout
kubectl rollout resume deployment/nginx-deployment

# Deletar Deployment
kubectl delete deployment nginx-deployment

```

### Atualizando um Deployment

### Método 1: kubectl set image (Recomendado)

```bash
kubectl set image deployment/nginx-deployment \
  nginx=nginx:1.16.0 \
  --record

# Verificar status
kubectl rollout status deployment/nginx-deployment

```

### Método 2: Editar YAML

```bash
kubectl edit deployment nginx-deployment
# Muda a imagem no editor, salva e sai
# Kubernetes faz o deploy automaticamente

```

### Método 3: Aplicar novo arquivo

```bash
# Modifica deployment.yaml
kubectl apply -f deployment.yaml

```

### Histórico e Rollback

```bash
# Ver histórico
kubectl rollout history deployment/nginx-deployment
# REVISION  CHANGE-CAUSE
# 1         kubectl create --record=true
# 2         kubectl set image deployment/nginx-deployment nginx=nginx:1.16.0

# Rollback para versão anterior
kubectl rollout undo deployment/nginx-deployment

# Rollback para versão específica
kubectl rollout undo deployment/nginx-deployment --to-revision=1

# Verificar mudanças
kubectl diff -f deployment.yaml

```

### Observar Mudanças em Tempo Real

```bash
# Watch dos Pods durante atualização
kubectl get pods -w

# Watch do Deployment
kubectl get deploy -w

# Logs de todos os Pods de um Deployment
kubectl logs -l app=nginx --tail=20 -f

```

---

## Comparação: ReplicaSet vs Deployment

| Aspecto | ReplicaSet | Deployment |
| --- | --- | --- |
| **Propósito** | Garante replicas | Gerencia ReplicaSets + updates |
| **Rolling Updates** | ❌ Não | ✅ Sim |
| **Rollback** | ❌ Não | ✅ Sim |
| **Histórico** | ❌ Não | ✅ Sim |
| **Escalabilidade** | ✅ Sim | ✅ Sim |
| **Recriar Pods** | ✅ Sim | ✅ Sim |
| **Uso em Produção** | ❌ Raramente | ✅ Sempre |
| **Quando Usar** | Casos especiais | 99% dos casos |

---

## Fluxo Completo: Deployment → ReplicaSet → Pods

```yaml
# 1. Você cria um Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14

```

```
O Kubernetes faz:

2. Deployment Controller cria ReplicaSet
   ReplicaSet: nginx-5d59d67564 (hash da spec)

3. ReplicaSet Controller cria 3 Pods
   Pod: nginx-5d59d67564-abc12
   Pod: nginx-5d59d67564-def45
   Pod: nginx-5d59d67564-ghi78

Você atualiza a imagem para nginx:1.16:

4. Deployment Controller cria novo ReplicaSet
   ReplicaSet: nginx-5d59d67564 (antiga)
   ReplicaSet: nginx-8f7b2c3d9e (nova)

5. Novo ReplicaSet cria Pods com nginx:1.16
   ReplicaSet nova: 1 Pod
   ReplicaSet antiga: 2 Pods (em destruição)

6. Continua até:
   ReplicaSet nova: 3 Pods
   ReplicaSet antiga: 0 Pods

7. Deployment mantém histórico:
   REVISION 1: ReplicaSet nginx-5d59d67564
   REVISION 2: ReplicaSet nginx-8f7b2c3d9e

Você faz rollback:

8. Deployment Controller recria ReplicaSet antiga
   ReplicaSet: nginx-5d59d67564 (volta a rodar)

9. Pods voltam para nginx:1.14

```

---

## Pontos Importantes para Prova CKA

✅ **Sempre use Deployment** - não ReplicaSet direto
✅ **RollingUpdate** é a estratégia padrão e recomendada
✅ **maxSurge** e **maxUnavailable** controlam a velocidade de atualização
✅ **--record** mantém histórico (use em set image)
✅ **kubectl rollout undo** desfaz atualizações
✅ **kubectl scale** muda número de replicas
✅ Deployment cria ReplicaSets automaticamente (não mexa neles)
✅ ReplicaSet Controller mantém o número exato de Pods rodando
✅ **Selector** usa labels para identificar quais Pods gerenciar
✅ Entender a diferença entre **desired**, **updated**, **total**, **available**
✅ Pods orphans podem ser adotados por novo ReplicaSet se labels correspondem
✅ **Recreate** causa downtime, evite em produção

---

## Casos de Uso Comuns

### Caso 1: Deploy Simples com 3 Replicas

```bash
kubectl create deployment web --image=nginx --replicas=3

```

### Caso 2: Atualizar Imagem

```bash
kubectl set image deployment/web nginx=nginx:1.20 --record
kubectl rollout status deployment/web

```

### Caso 3: Rollback se Algo Deu Errado

```bash
kubectl rollout undo deployment/web

```

### Caso 4: Escalar Rápido

```bash
kubectl scale deployment/web --replicas=10

```

### Caso 5: Pausar Atualização

```bash
kubectl rollout pause deployment/web
# Faz testes
kubectl rollout resume deployment/web

```

---

## Debug Comum

```bash
# Deployment não está fazendo rollout?
kubectl describe deploy nginx-deployment
# Verifique: replicas desejadas vs current

# Pods não estão criando?
kubectl describe rs nginx-deployment-XXX
# Verifique: events no final

# Imagem não atualizou?
kubectl get pods -o jsonpath='{.items[0].spec.containers[0].image}'

# Ver mudanças entre revisions
kubectl rollout history deployment/nginx --revision=1
kubectl rollout history deployment/nginx --revision=2

```

---

# Services no Kubernetes - Resumo para Estudos CKA

## O que é um Service?

Um **Service** é um objeto que define como acessar Pods. Ele fornece uma abstração estável (IP e DNS) para um conjunto de Pods efêmeros. Sem Services, você não conseguiria acessar seus Pods de forma confiável, pois eles aparecem e desaparecem constantemente.

## Por Que Precisa de Services?

```
Problema:
- Pods têm IPs temporários
- Pods morrem e novos nascem com IPs diferentes
- Pods estão distribuídos em múltiplos nós
- Como conectar entre aplicações?

Solução:
- Service fornece IP fixo (Virtual IP - VIP)
- Service fornece DNS estável (nome.namespace.svc.cluster.local)
- Service distribui tráfego entre Pods (load balancing)
- Service abstrai os detalhes dos Pods

```

---

## Tipos de Services

### 1. ClusterIP (Padrão)

Expõe o Service apenas dentro do cluster. Outros Pods podem acessar via IP ou DNS, mas nada de fora do cluster consegue acessar.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: default
spec:
  type: ClusterIP  # padrão, pode omitir
  selector:
    app: nginx      # seleciona Pods com esta label
  ports:
  - protocol: TCP
    port: 80        # porta do Service
    targetPort: 80  # porta do Pod

```

**Acesso:**

```bash
# De dentro do cluster
curl nginx-service         # mesmo namespace
curl nginx-service.default # namespace explícito
curl nginx-service.default.svc.cluster.local  # FQDN completo

# De outro namespace
curl nginx-service.default.svc.cluster.local

```

**Use quando:**

- Backend communication (Pod → Pod)
- Databases, caches, APIs internas
- Não precisa de acesso externo

---

### 2. NodePort

Expõe o Service em uma porta fixa em todos os nós do cluster. Tráfego externo pode acessar via `<node-ip>:<nodeport>`.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: NodePort
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80        # porta interna do Service
    targetPort: 80  # porta do Pod
    nodePort: 30008 # porta no nó (30000-32767)
                    # se omitir, Kubernetes escolhe

```

**Fluxo de tráfego:**

```
Cliente externo (30.0.0.1:30008)
    ↓
Qualquer nó do cluster (node-1, node-2, node-3)
    ↓
Service NodePort
    ↓
Pods selecionados (pode ser em qualquer nó)

```

**Acesso:**

```bash
# De fora do cluster
curl node-1-ip:30008
curl node-2-ip:30008
curl node-3-ip:30008

# Qualquer nó funciona, mesmo se Pod não está lá
# (Kubernetes redireciona automaticamente)

```

**Use quando:**

- Precisa acessar serviço de fora do cluster
- Desenvolvimento/teste
- Sem load balancer externo disponível
- Porta fixa é importante

**Limitações:**

- Portas altas (30000-32767)
- Sem load balancing externo
- Expõe IPs dos nós

---

### 3. LoadBalancer

Expõe o Service via load balancer externo (AWS ELB, GCP Load Balancer, Azure LB, etc).

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  selector:
    app: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30008  # opcional

```

**Fluxo de tráfego:**

```
Cliente externo (internet)
    ↓
Load Balancer Externo (tem IP público)
    ↓
Qualquer nó do cluster
    ↓
Service LoadBalancer
    ↓
Pods selecionados

```

**Após criar:**

```bash
kubectl get svc nginx-service

# Output:
# NAME            TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)
# nginx-service   LoadBalancer   10.0.1.100      a1b2c3d4e5f6.elb.amazonaws.com   80:30008/TCP

```

**Acesso:**

```bash
# Via LoadBalancer (recomendado)
curl a1b2c3d4e5f6.elb.amazonaws.com:80

# Via qualquer nó (também funciona)
curl node-ip:30008

```

**Use quando:**

- Produção com load balancer disponível
- Precisa de IP externo único
- Quer distribuição de tráfego externo
- Serviço é público/principal

**Custo:** Cada LoadBalancer é um recurso separado (pode ser caro)

---

### 4. ExternalName

Cria um alias DNS para um serviço externo. Redireciona tráfego para um serviço fora do cluster.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: external-db
spec:
  type: ExternalName
  externalName: database.example.com  # FQDN externo
  port: 5432

```

**Uso:**

```bash
# De dentro do cluster
kubectl run psql --image=postgres --rm -it \
  -- psql -h external-db -U user

# Internamente resolve para database.example.com

```

**Use quando:**

- Precisa conectar em serviços externos
- Quer usar DNS interno do Kubernetes
- Migrando de sistema externo para Kubernetes
- Abstratir localização do serviço externo

---

## Definição Completa de um Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
  namespace: default
  labels:
    app: web
  annotations:
    description: "Service para aplicação web"
spec:
  type: ClusterIP
  clusterIP: 10.0.1.100  # IP específico (geralmente auto)
  selector:
    app: webapp           # seleciona Pods
    tier: frontend
  ports:
  - name: http           # nome da porta (opcional)
    protocol: TCP
    port: 80             # porta do Service
    targetPort: 8080     # porta do Pod
  - name: https
    protocol: TCP
    port: 443
    targetPort: 8443
  sessionAffinity: None   # ou ClientIP para sticky sessions
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600

```

---

## Service Discovery (Como Pods Encontram Services)

### Método 1: DNS (Recomendado)

```bash
# Dentro do cluster
curl http-service           # mesmo namespace
curl http-service.default   # outro namespace
curl http-service.default.svc.cluster.local  # FQDN

```

**DNS Format:**

```
<service-name>.<namespace>.svc.cluster.local

```

### Método 2: Environment Variables

```bash
# Kubernetes injeta automaticamente
# No Pod:
echo $NGINX_SERVICE_HOST      # IP do Service
echo $NGINX_SERVICE_PORT      # Porta do Service

# Convenção:
# <SERVICE_NAME>_SERVICE_HOST
# <SERVICE_NAME>_SERVICE_PORT

```

### Método 3: Selecionar por IP

```bash
# Não recomendado - mude para DNS se possível
curl 10.0.1.100:80

```

---

## Endpoints e seletor de Service

Um **Endpoint** conecta o Service aos Pods reais.

```bash
# Ver endpoints de um Service
kubectl get endpoints
kubectl get ep

# Descrever um Service (mostra endpoints)
kubectl describe svc nginx-service
# Endpoints: 10.244.1.10:80,10.244.1.11:80,10.244.1.12:80

# Os Pods com label app=nginx têm esses IPs
# Se adicionar/remover Pod, endpoints atualizam automaticamente

```

### Service sem Selector

```yaml
# Service sem selector (gerencia Endpoints manualmente)
apiVersion: v1
kind: Service
metadata:
  name: external-api
spec:
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
---
# Endpoint manual
apiVersion: v1
kind: Endpoints
metadata:
  name: external-api
subsets:
- addresses:
  - ip: 192.168.1.100
  ports:
  - port: 8080

```

**Use quando:**

- Conectar em sistema legado/externo
- Controle total sobre Endpoints
- Múltiplos backends que não são Pods

---

## Comandos Essenciais

```bash
# Criar Service
kubectl create service clusterip web --tcp=80:8080
kubectl create service nodeport web --tcp=80:8080 --node-port=30008
kubectl create service loadbalancer web --tcp=80:8080

# Ou via arquivo YAML
kubectl create -f service.yaml
kubectl apply -f service.yaml

# Listar Services
kubectl get services
kubectl get svc
kubectl get svc -n default
kubectl get svc -o wide

# Descrever Service (muito importante!)
kubectl describe svc nginx-service

# Ver Endpoints
kubectl get endpoints
kubectl get ep
kubectl describe ep nginx-service

# Editar Service
kubectl edit svc nginx-service

# Expor um Deployment como Service
kubectl expose deployment nginx-dep --type=ClusterIP --port=80 --target-port=8080
kubectl expose deployment nginx-dep --type=NodePort --port=80 --target-port=8080

# Deletar Service
kubectl delete svc nginx-service
kubectl delete -f service.yaml

# Testar conectividade
kubectl run -it --rm debug --image=busybox -- wget -O- http://nginx-service:80

# Port forward (acesso local a um Service)
kubectl port-forward svc/nginx-service 8080:80
# Agora acessa via localhost:8080

```

---

## Estratégias de Load Balancing

### 1. Round-Robin (Padrão)

Distribui tráfego igualmente entre Pods.

```
Requisição 1 → Pod A
Requisição 2 → Pod B
Requisição 3 → Pod C
Requisição 4 → Pod A
Requisição 5 → Pod B
...

```

### 2. Session Affinity (Sticky Sessions)

Mantém cliente conectado ao mesmo Pod.

```yaml
spec:
  sessionAffinity: ClientIP
  sessionAffinityConfig:
    clientIP:
      timeoutSeconds: 3600  # reset após 1 hora

```

**Use quando:**

- Aplicação precisa de estado (sessão)
- Evitar compartilhamento de sessão entre Pods

---

## Service Policy: externalTrafficPolicy

Controla como tráfego externo é roteado.

### Local (Sem saltos)

```yaml
spec:
  type: NodePort
  externalTrafficPolicy: Local  # rota para Pods no mesmo nó

```

```
Cliente → Node A:30008
    ↓
Se Pod está em Node A: acessa direto
Se Pod está em outro nó: conexão rejeitada/sem resposta

```

**Vantagem:** Preserva IP do cliente (source IP real)
**Desvantagem:** Distribuição desigual entre nós

### Cluster (Padrão)

```yaml
spec:
  type: NodePort
  externalTrafficPolicy: Cluster  # pode pular para outro nó

```

```
Cliente → Node A:30008
    ↓
Se Pod está em Node A: acessa direto
Se Pod está em outro nó: rota para lá (hop extra)

```

**Vantagem:** Distribuição igual entre Pods
**Desvantagem:** Source IP é mascarado (não é o cliente real)

---

## Comparação de Tipos de Services

| Tipo | Acesso Interno | Acesso Externo | IP Fixo | DNS | Use Case |
| --- | --- | --- | --- | --- | --- |
| **ClusterIP** | ✅ Sim | ❌ Não | ✅ Sim | ✅ Sim | Backend, APIs internas |
| **NodePort** | ✅ Sim | ✅ Sim (nó) | ✅ Sim | ✅ Sim | Dev/test, sem LB |
| **LoadBalancer** | ✅ Sim | ✅ Sim (LB) | ✅ Sim | ✅ Sim | Produção, serviços públicos |
| **ExternalName** | ✅ Sim | ⚠️ Via externo | ⚠️ Não | ✅ Sim | Redirect externo |

---

## Troubleshooting Services

### Service criado mas não funciona?

```bash
# 1. Verificar se Service existe
kubectl get svc nginx-service
kubectl describe svc nginx-service

# 2. Verificar Endpoints (Pods selecionados)
kubectl get ep nginx-service
# Se vazio: selector não match com nenhum Pod

# 3. Verificar labels dos Pods
kubectl get pods --show-labels
# Compare com selector do Service

# 4. Testar dentro do cluster
kubectl run -it --rm test --image=busybox -- sh
$ wget -O- http://nginx-service
$ nslookup nginx-service

# 5. Verificar iptables (kube-proxy)
kubectl describe node node-name
# Procure por portas e regras

# 6. Ver logs do kube-proxy
kubectl logs -n kube-system -l component=kube-proxy

# 7. Verificar conectividade de Pod para Pod
kubectl exec -it pod-name -- sh
$ wget http://nginx-service

```

### NodePort não responde?

```bash
# 1. Verificar porta
kubectl get svc nginx-service
# Procure por node-port na saída

# 2. Testar de dentro do cluster primeiro
kubectl run -it --rm test --image=busybox -- wget -O- http://nginx-service

# 3. Testar via IP do nó
kubectl get nodes -o wide
# Use NODE-EXTERNAL-IP:nodeport

# 4. Verificar firewall
# Porta nodeport (30000-32767) deve estar aberta

# 5. Verificar se Pod está rodando
kubectl get pods -o wide
# Procure por Running e Ready

```

### LoadBalancer não recebe IP externo?

```bash
# Pode levar alguns minutos
kubectl get svc -w
# Aguarde EXTERNAL-IP aparecer

# Se ficar como <pending>:
kubectl describe svc nginx-service
# Procure por Events e mensagens de erro

# Provedor de nuvem pode não estar configurado
# (verificar credenciais do cloud controller)

```

---

## Casos de Uso Comuns

### Caso 1: API Backend (ClusterIP)

```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: api
  template:
    metadata:
      labels:
        app: api
    spec:
      containers:
      - name: api
        image: api:v1
        ports:
        - containerPort: 5000
---
# Service ClusterIP
apiVersion: v1
kind: Service
metadata:
  name: api-service
spec:
  type: ClusterIP
  selector:
    app: api
  ports:
  - port: 80
    targetPort: 5000

```

**Acesso de outro Pod:**

```bash
curl http://api-service:80

```

---

### Caso 2: Aplicação Web Pública (LoadBalancer)

```yaml
---
# Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
# Service LoadBalancer
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  type: LoadBalancer
  selector:
    app: web
  ports:
  - port: 80
    targetPort: 80

```

**Acesso externo:**

```bash
curl a1b2c3d4e5f6.elb.amazonaws.com

```

---

### Caso 3: Teste Local (NodePort)

```bash
# Criar deployment
kubectl create deployment test --image=nginx --replicas=2

# Expor como NodePort
kubectl expose deployment test --type=NodePort --port=80 --target-port=80

# Obter informações
kubectl get svc test

# Acessar
curl node-ip:node-port

```

---

## Pontos Importantes para Prova CKA

✅ **ClusterIP é o padrão** - use quando service é interno
✅ **Selector em Service** - deve match com labels dos Pods
✅ **Endpoints** - criados automaticamente quando selector match
✅ **port vs targetPort** - port é do Service, targetPort é do Pod
✅ **nodePort range** - 30000-32767
✅ **DNS interno** - `<service>.<namespace>.svc.cluster.local`
✅ **LoadBalancer** - precisa de cloud provider configurado
✅ **ExternalName** - redireciona para serviço externo
✅ **sessionAffinity** - sticky sessions com ClientIP
✅ **Verificar Endpoints** - se vazio, selector não match
✅ **kubectl expose** - cria Service de um Deployment
✅ **service discovery** - DNS é mais confiável que env vars
✅ **Load balancing padrão** - round-robin entre Pods
✅ **externalTrafficPolicy: Local** - sem hop extra, preserva source IP

---

## Quick Reference: Criar Services

```bash
# ClusterIP (padrão)
kubectl expose deployment nginx --type=ClusterIP --port=80

# NodePort
kubectl expose deployment nginx --type=NodePort --port=80 --node-port=30008

# LoadBalancer
kubectl expose deployment nginx --type=LoadBalancer --port=80

# Com YAML
kubectl create -f service.yaml
kubectl apply -f service.yaml

# Descrever para debugging
kubectl describe svc nginx

```

---

## DNS Resolution Chain

```
Pod tenta: curl nginx-service

1. Procura em /etc/resolv.conf do Pod
   search default.svc.cluster.local svc.cluster.local cluster.local
   nameserver 10.0.0.10  (coredns)

2. Tenta: nginx-service.default.svc.cluster.local

3. CoreDNS resolve para IP do Service
   nginx-service → 10.0.1.100

4. Tráfego vai para 10.0.1.100:80
   (kube-proxy redireciona para Pods reais)

Se estiver em outro namespace:
curl nginx-service.other-namespace.svc.cluster.local

```

---

# Scheduling

# Scheduling Manual no Kubernetes/EKS - Resumo para Estudos CKA

## O que é Scheduling Manual?

**Scheduling** é o processo de decidir em qual **nó** um Pod vai rodar. Normalmente o Kubernetes faz isso automaticamente, mas você pode forçar um Pod a rodar em um nó específico.

## Por Que Precisa?

```
Cenários comuns:
- Pod precisa de GPU (machine learning)
- Pod precisa de muita CPU/RAM
- Pod precisa estar em nó específico (SSD rápido)
- App precisa estar junto/separada de outra app
- Hardware especial (TPU, FPGA, etc)

```

---

## Formas de Fazer Scheduling Manual

### 1. nodeSelector (Mais Simples) ⭐

Força o Pod a rodar em nós com labels específicas.

### Passo 1: Etiquetar o Nó

```bash
# Adicionar label ao nó
kubectl label nodes node-1 tipo=gpu-ready

# Ver labels do nó
kubectl get nodes --show-labels
kubectl label nodes node-1 --list

```

### Passo 2: Usar nodeSelector no Pod

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-pod
spec:
  nodeSelector:
    tipo: gpu-ready    # Pod roda APENAS em nós com essa label
  containers:
  - name: tensorflow
    image: tensorflow:latest
    resources:
      requests:
        nvidia.com/gpu: 1  # precisa de 1 GPU

```

### Criar via kubectl

```bash
kubectl run gpu-app --image=tensorflow --dry-run=client -o yaml > pod.yaml
# Editar e adicionar nodeSelector
kubectl create -f pod.yaml

```

### Verificar

```bash
# Ver em qual nó o Pod está rodando
kubectl get pods -o wide

# Descrever Pod para ver se foi agendado
kubectl describe pod ml-pod
# Procure por "Node:" e "Events"

```

---

### 2. nodeAffinity (Mais Poderoso) ⭐⭐

Mais flexível que nodeSelector. Permite múltiplas condições e preferências.

### Affinity Obrigatória (requiredDuringSchedulingIgnoredDuringExecution)

Pod **NÃO inicia** se a condição não for atendida.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: web-pod
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
            - nvme
          - key: zona
            operator: In
            values:
            - us-east-1a
  containers:
  - name: web
    image: nginx:latest

```

**Operadores disponíveis:**

- `In`: valor está na lista
- `NotIn`: valor não está na lista
- `Exists`: key existe (não verifica valor)
- `DoesNotExist`: key não existe
- `Gt`: valor > número
- `Lt`: valor < número

### Affinity com Preferência (preferredDuringSchedulingIgnoredDuringExecution)

Pod **prefere** rodar nesse nó, mas pode rodar em outro se necessário.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: cache-pod
spec:
  affinity:
    nodeAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100          # peso 1-100 (maior = mais preferência)
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 50
        preference:
          matchExpressions:
          - key: zona
            operator: In
            values:
            - us-east-1a
  containers:
  - name: redis
    image: redis:latest

```

### Combinando Obrigatória + Preferência

```yaml
spec:
  affinity:
    nodeAffinity:
      # Tem que cumprir ISSO
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: tipo
            operator: In
            values:
            - producao
      # E prefere ISSO
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd

```

---

### 3. Pod Affinity/Anti-Affinity

Força um Pod estar junto com outro Pod (ou afastado).

### Pod Affinity (Junto)

Pod A prefere/requer estar no mesmo nó/zona que Pod B.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: app-pod
spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - cache
        topologyKey: kubernetes.io/hostname  # mesmo nó
  containers:
  - name: app
    image: app:latest

```

**topologyKey:**

- `kubernetes.io/hostname`: mesmo nó
- `topology.kubernetes.io/zone`: mesma zona/AZ
- `topology.kubernetes.io/region`: mesma região

### Pod Anti-Affinity (Separado)

Pod A requer estar em nó diferente de Pod B.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database-1
  labels:
    db: postgresql
spec:
  affinity:
    podAntiAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: db
            operator: In
            values:
            - postgresql
        topologyKey: kubernetes.io/hostname  # nó diferente
  containers:
  - name: postgres
    image: postgres:latest

```

**Use quando:**

- Não quer dois databases no mesmo nó
- Quer alta disponibilidade (distribuir entre nós)
- Quer latência baixa (próximo)

---

### 4. Taints e Tolerations

Força certos Pods a NÃO rodarem em um nó (ou só Pods específicos).

### Taint no Nó

```bash
# Adicionar taint ao nó
kubectl taint nodes node-1 tipo=gpu:NoSchedule
kubectl taint nodes node-1 tipo=gpu:NoExecute
kubectl taint nodes node-1 tipo=gpu:PreferNoSchedule

# Ver taints
kubectl describe node node-1 | grep Taints

# Remover taint
kubectl taint nodes node-1 tipo=gpu:NoSchedule-

```

**Efeitos:**

- `NoSchedule`: Não coloca Pod novo (mas Pods existentes continuam)
- `NoExecute`: Remove Pod imediatamente se não tolera
- `PreferNoSchedule`: Evita colocar, mas pode se necessário

### Toleration no Pod

Pod que **tolera** (aguenta) o taint pode rodar lá.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: gpu-task
spec:
  tolerations:
  - key: tipo
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: gpu-app
    image: gpu-app:latest

```

### Exemplo Prático

```bash
# 1. Marcar nó como GPU-only
kubectl taint nodes node-gpu tipo=gpu:NoSchedule

# 2. Pod normal tenta rodar
kubectl run normal-app --image=nginx
# Fica Pending (não consegue scheduling)

# 3. Pod com toleration consegue
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: gpu-app
spec:
  tolerations:
  - key: tipo
    operator: Equal
    value: gpu
    effect: NoSchedule
  containers:
  - name: app
    image: gpu-app:latest
EOF
# Roda normalmente

```

---

## Resumo Rápido: Quando Usar Cada Um

| Método | Complexidade | Uso | Quando |
| --- | --- | --- | --- |
| **nodeSelector** | Simples | Label simples no nó | Labels 1-2 condições |
| **nodeAffinity** | Médio | Multiple labels, operadores | Múltiplas condições, preferências |
| **Pod Affinity** | Médio-Alto | Pod próximo a outro Pod | Apps que precisam estar juntas |
| **Pod Anti-Affinity** | Médio-Alto | Pod longe de outro Pod | Alta disponibilidade, distribuir |
| **Taints/Tolerations** | Médio | Bloquear nós | Nós especializados (GPU, SSD) |

---

## Exemplos Práticos para CKA

### Exemplo 1: GPU para Machine Learning

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ml-training
spec:
  nodeSelector:
    hardware: gpu
  containers:
  - name: tensorflow
    image: tensorflow:latest
    resources:
      requests:
        nvidia.com/gpu: 1
      limits:
        nvidia.com/gpu: 1

```

**Setup:**

```bash
kubectl label nodes node-gpu hardware=gpu
kubectl create -f ml-training.yaml
kubectl get pods -o wide  # verifica em qual nó está

```

---

### Exemplo 2: Database com Persistência

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: database
  labels:
    app: postgres
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
  - name: postgres
    image: postgres:latest
    volumeMounts:
    - name: data
      mountPath: /var/lib/postgresql
  volumes:
  - name: data
    hostPath:
      path: /fast-ssd

```

---

### Exemplo 3: Alta Disponibilidade com Anti-Affinity

```yaml
# 3 replicas de database spread entre nós
apiVersion: apps/v1
kind: Deployment
metadata:
  name: db-cluster
spec:
  replicas: 3
  selector:
    matchLabels:
      app: db
  template:
    metadata:
      labels:
        app: db
    spec:
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: app
                operator: In
                values:
                - db
            topologyKey: kubernetes.io/hostname  # nó diferente
      containers:
      - name: postgres
        image: postgres:latest

```

---

### Exemplo 4: Nó Especializado com Taints

```bash
# Marcar nó como especializado
kubectl taint nodes node-special reserved=only:NoSchedule

# Pod que precisa desse nó
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: special-workload
spec:
  tolerations:
  - key: reserved
    operator: Equal
    value: only
    effect: NoSchedule
  containers:
  - name: app
    image: special-app:latest
EOF

```

---

## Comandos Essenciais para CKA

```bash
# ========== LABELS ==========
# Adicionar label a nó
kubectl label nodes node-1 gpu=true

# Ver labels de nó
kubectl get nodes --show-labels
kubectl label nodes node-1 --list

# Remover label
kubectl label nodes node-1 gpu-

# ========== TAINTS ==========
# Adicionar taint
kubectl taint nodes node-1 gpu=true:NoSchedule

# Ver taints
kubectl describe node node-1 | grep Taints

# Remover taint
kubectl taint nodes node-1 gpu=true:NoSchedule-

# ========== SCHEDULING ==========
# Criar Pod com nodeSelector (via YAML)
kubectl apply -f pod-with-selector.yaml

# Ver em qual nó Pod está
kubectl get pods -o wide

# Descrever Pod (ver se foi agendado)
kubectl describe pod nome-pod

# Ver events de Pod (agendamento)
kubectl describe pod nome-pod | grep Events

# ========== DEBUGGING ==========
# Pod stuck em Pending?
kubectl describe pod nome-pod
# Procure por "node affinity", "taint", "status"

# Teste scheduling manualmente
kubectl create --dry-run=client -o yaml -f pod.yaml | kubectl apply -f -

```

---

## Troubleshooting Comum

### Pod em Pending (não agenda)

```bash
# Causa: nodeSelector não match
kubectl describe pod meu-pod
# Procure por: "no nodes match"

# Solução: verificar labels
kubectl get nodes --show-labels
kubectl label nodes node-1 gpu=true  # adicionar label

# Verificar taints
kubectl describe node node-1 | grep Taints
# Se houver taint, Pod precisa de toleration

```

### Pod em CrashLoopBackOff

```bash
# Pode ser taint com NoExecute (mata Pod)
kubectl describe pod meu-pod
# Procure por: "tainted"

# Verificar taints do nó onde Pod estava
kubectl describe node onde-pod-estava

# Remover taint se necessário
kubectl taint nodes node-1 chave=valor:NoExecute-

```

### Anti-Affinity Impede Replicação

```bash
# Problema: Pod prefere spread, mas menos nós que replicas
kubectl describe pod meu-pod
# Se diz "no nodes", anti-affinity está prorrogando

# Solução: usar preferredDuringScheduling ao invés de required
# Ou adicionar mais nós

```

---

## Pontos Importantes para Prova CKA

✅ **nodeSelector** - forma mais simples, labels diretas no nó
✅ **nodeAffinity** - forma mais poderosa, múltiplas condições
✅ **Pod Affinity** - colocar Pods juntos (mesma zona/nó)
✅ **Pod Anti-Affinity** - separar Pods (diferentes nós/zonas)
✅ **Taints/Tolerations** - bloquear nós + permitir exceções
✅ **topologyKey** - define nível: hostname, zone, region
✅ **requiredDuringScheduling** - obrigatório (Pod fica Pending se não quer)
✅ **preferredDuringScheduling** - preferência (Pod pode rodar em outro lugar)
✅ **NoSchedule vs NoExecute** - schedule impede novo, execute mata existente
✅ **kubectl label nodes** - adicionar labels para seleção
✅ **kubectl taint nodes** - marcar nó como especial
✅ Ver labels: `kubectl get nodes --show-labels`
✅ Ver taints: `kubectl describe node | grep Taints`
✅ Debugging: `kubectl describe pod` vê eventos de agendamento
✅ Pod Pending = problema de scheduling, ver describe pod

---

## Quick Reference

```bash
# Setup nó com label
kubectl label nodes node-1 hardware=gpu

# Pod simples (nodeSelector)
kubectl run gpu-app --image=app --dry-run=client -o yaml | \
  sed 's/containers:/nodeSelector:\n    hardware: gpu\n  containers:/' | \
  kubectl apply -f -

# Ver resultado
kubectl get pods -o wide

# Taint nó
kubectl taint nodes node-1 reserved=true:NoSchedule

# Pod com toleration
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  tolerations:
  - key: reserved
    operator: Equal
    value: "true"
    effect: NoSchedule
  containers:
  - name: app
    image: nginx
EOF

```

---

## Resumo em Uma Frase

**Scheduling manual = você diz EXATAMENTE em qual nó (ou tipo de nó) o Pod vai rodar, em vez de deixar o Kubernetes escolher automaticamente.**