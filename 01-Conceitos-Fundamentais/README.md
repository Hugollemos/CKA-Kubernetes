# Conceitos Fundamentais

Esta pasta contém os conceitos básicos e informações essenciais para começar os estudos para a certificação CKA.

## 📚 Conteúdo

### [dicas-e-links.md](./dicas-e-links.md)
Dicas importantes para a prova CKA, incluindo:
- Informações sobre a prova (duração, questões, versão do Kubernetes)
- Aliases úteis e comandos para economizar tempo
- Links para documentação oficial
- Recursos de estudo (vídeos, cursos, artigos em português)
- Links para preparatórios e materiais de referência

### [componentes-overview.md](./componentes-overview.md)
Visão geral dos componentes do Kubernetes:
- **Componentes do Control Plane**: kube-scheduler, kube-controller-manager, kube-apiserver
- **Componentes dos Worker Nodes**: kubelet, kube-proxy
- Responsabilidades de cada componente

### [docker-containerd.md](./docker-containerd.md)
Fundamentos de containerização:
- O que é containerização
- OCI (Open Container Initiative)
- ContainerD, runc e componentes
- Ferramentas CLI: docker, nerdctl, crictl, ctr
- Container Runtime Interface (CRI)
- Evolução de Docker para ContainerD

### [docker-storage.md](./docker-storage.md)
Sistema de armazenamento do Docker:
- Arquitetura de storage do Docker
- Tipos de storage: Volumes, Bind Mounts, tmpfs
- Layered Filesystem e Copy-on-Write (CoW)
- Storage Drivers (overlay2, aufs, devicemapper)
- Estrutura de diretórios `/var/lib/docker/`
- Comandos de gerenciamento de volumes
- Troubleshooting de problemas de storage
- Relação com Persistent Volumes do Kubernetes

## 🎯 Como Estudar

1. **Comece com**: [dicas-e-links.md](./dicas-e-links.md) para entender o formato da prova
2. **Depois**: [componentes-overview.md](./componentes-overview.md) para ter uma visão geral da arquitetura
3. **Continue com**: [docker-containerd.md](./docker-containerd.md) para entender a base de containers
4. **Finalize com**: [docker-storage.md](./docker-storage.md) para entender storage de containers

---

➡️ **Próximo passo**: [Componentes-Control-Plane](../Componentes-Control-Plane/)
