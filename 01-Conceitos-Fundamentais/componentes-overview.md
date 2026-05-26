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
- Outros controladores incluem: Deployment Controller, ReplicaSet Controller, Job Controller, DaemonSet Controller, entre outros

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

