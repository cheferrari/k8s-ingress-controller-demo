## Traefik on k8s demo
refers to https://docs.traefik.io/user-guide/kubernetes/
### 1. Deployments add affinity settings
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
### 2. Traffic Splitting --> canary releases
refers to https://docs.traefik.io/user-guide/kubernetes/#traffic-splitting
可以使用服务权重在多个部署之间以细粒度方式拆分Ingress流量。一个规范用例是canary版本，其中代表较新版本的部署将随着时间的推移接收最初较小但不断增加的请求部分。在Traefik中可以这样做的方法是指定应该进入每个部署的请求的百分比。  
例如，假设应用程序my-app在版本1中运行。较新的版本2即将发布，但对生产中运行的新版本的稳健性和可靠性的信心只能逐渐获得。因此，创建了一个新的部署my-app-canary并将其扩展为足以获得20％流量份额的副本计数。与此同时，像往常一样创建一个Service对象。  
canary-test.yaml中 Ingress 定义：
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  annotations:
    kubernetes.io/ingess.class: traefik
    traefik.ingress.kubernetes.io/service-weights: |
      my-app: 80%
      my-app-canary: 20%
  name: my-app
spec:
  rules:
  - http:
      paths:
      - backend:
          serviceName: my-app
          servicePort: 80
        path: /
      - backend:
          serviceName: my-app-canary
          servicePort: 80
        path: /
```
