## Traefik on k8s demo
refers to https://docs.traefik.io/user-guide/kubernetes/
### Deployments add affinity settings
Deployments add affinity settings to ensure that two pods don't end up on the same node.  
spec.template.spec.affinity
```
      affinity:
        podAntiAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchExpressions:
              - key: k8s-app
                operator: In
                values:
                - traefik-ingress-lb
            topologyKey: kubernetes.io/hostname
```
