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

