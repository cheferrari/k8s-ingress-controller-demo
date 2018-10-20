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
#### canary-test.yaml中 Ingress 定义：
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
Traefik的canary实现是通过在同一路径下（如 path: /）指定两个后端服务（每个服务代表的版本不一样），然后设置每个服务的权重，从而是每个服务获取的流量不同，达到金丝雀发布的效果。  
#### Traefik-canary示意图
![traefik-canary](https://github.com/cheferrari/k8s-ingress-controller-demo/tree/master/Traefik/img/Traefik.PNG)  
这跟 kubernetes 原生的 canary deployment 是由区别的，原生的是在同一服务Service后边通过Pod标签挑选出具有相同标签的 两个不同版本的的deployment来实现，流量分发完全是通过每个deployment的副本数来控制，如版本1的deployment副本数是10，较新的版本2的deployment副本数是5，那么新版本就会获取1/3的流量。  参考：https://github.com/cheferrari/k8s-demo/tree/master/canary-deployment
#### k8s-canary示意图
![k8s-canary](https://github.com/cheferrari/k8s-ingress-controller-demo/blob/master/Traefik/img/k8s-canary.png)
```
[root@k8s-node1 ~]# kubectl get svc
NAME            TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
cheddar         ClusterIP   10.104.37.200    <none>        80/TCP    3d3h
my-app          ClusterIP   10.96.57.127     <none>        80/TCP    22h
my-app-canary   ClusterIP   10.106.36.190    <none>        80/TCP    21h
stilton         ClusterIP   10.107.104.196   <none>        80/TCP    3d3h
wensleydale     ClusterIP   10.103.39.190    <none>        80/TCP    3d3h
[root@k8s-node1 ~]# kubectl get pod --show-labels |grep web
web-v1-56db97cfd-vpjnw         1/1     Running   1          22h    app=my-app,pod-template-hash=56db97cfd,track=stable,version=my-app-version-1
web-v1-56db97cfd-wl65c         1/1     Running   1          22h    app=my-app,pod-template-hash=56db97cfd,track=stable,version=my-app-version-1
web-v2-54c65655f-c49bz         1/1     Running   1          22h    app=my-app-canary,pod-template-hash=54c65655f,track=canary,version=my-app-version-2
web-v2-54c65655f-mp2m9         1/1     Running   1          22h    app=my-app-canary,pod-template-hash=54c65655f,track=canary,version=my-app-version-2
[root@k8s-node1 istio-canary]# while true; do curl http://192.168.75.152:30820/; sleep 1; done
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v2!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
<h1>My websever Version: v1!</h1>
```
前20个访问中，stable：v1版本响应了16个，canary：v2版本响应了4个，正好v1版本的流占到30%，canary版本占到20%
