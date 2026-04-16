# Troubleshooting no Kubernetes

Esta seção cobre as principais abordagens de troubleshooting para o CKA.

## Tópicos

- [Falha de Aplicação](application-failure.md) — Diagnosticar problemas em aplicações rodando no cluster
- [Falha do Control Plane](control-plane-failure.md) — Diagnosticar componentes do control plane
- [Falha de Worker Node](worker-node-failure.md) — Diagnosticar nós que saem do estado Ready

## Abordagem Geral

```
1. Identificar o sintoma (o que está falhando?)
2. Verificar o estado dos recursos (kubectl get/describe)
3. Ver os logs (kubectl logs, journalctl)
4. Corrigir e verificar
```
