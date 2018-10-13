#### 生成自签证书，这里为了方便直接使用CA的证书配置到che.webserver.com服务器上
#### 1、生成CA私钥
```
[root@k8s-node1 tls-demo]# (umask 077; openssl genrsa -out tls.key 2048)
Generating RSA private key, 2048 bit long modulus
.............................................................................+++
...........................................................+++
e is 65537 (0x10001)
[root@k8s-node1 tls-demo]# ll
total 16
-rw-r--r-- 1 root root  423 Oct 13 14:58 ingress-webserver-tls.yaml
-rw-r--r-- 1 root root   66 Oct 13 13:33 namespace.yaml
-rw------- 1 root root 1679 Oct 13 15:00 tls.key
-rw-r--r-- 1 root root  778 Oct 13 13:32 webserver-deployment.yaml
```

#### 2、生成CA自签证书
```
[root@k8s-node1 tls-demo]# openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Guangdond/L=Shenzhen/O=DevOps/CN=che.webserver.com
[root@k8s-node1 tls-demo]# 
[root@k8s-node1 tls-demo]# ll tls*
-rw-r--r-- 1 root root 1302 Oct 13 15:03 tls.crt
-rw------- 1 root root 1679 Oct 13 15:00 tls.key
```

#### 3、生成tls类型的secret
##### The resulting secret will be of type kubernetes.io/tls
```
[root@k8s-node1 tls-demo]# kubectl create secret tls webserver-tls-secret --cert=tls.crt --key=tls.key -n ingress-tls-demo 
secret/webserver-tls-secret created
[root@k8s-node1 tls-demo]# kubectl get secrets -n ingress-tls-demo 
NAME                   TYPE                                  DATA   AGE
default-token-gp6fm    kubernetes.io/service-account-token   3      5m47s
webserver-tls-secret   kubernetes.io/tls                     2      35s
```

#### 4、生成 webserver-deployment.yaml
```
# cat webserver-deployment.yaml 
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webserver-tls-deployment
  namespace: ingress-tls-demo
  labels:
    app: webserver-tls
spec:
  replicas: 3
  selector:
    matchLabels:
      app: webserver-tls
  template:
    metadata:
      labels:
        app: webserver-tls
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80

---
kind: Service
apiVersion: v1
metadata:
  name: webserver-tls-svc
  namespace: ingress-tls-demo
spec:
  selector:
    app: webserver-tls
  ports:
  - protocol: TCP
    port: 80                                    #映射到集群内部IP(cluster ip)的监听端口是80
    targetPort: 80                              #容器内服务本身的端口80  
  type: NodePort
```
##### 应用文件，查看
```
# kubectl apply -f webserver-deployment.yaml
# kubectl get deploy,svc -n ingress-tls-demo
```

#### 5、生成Ingress
```
# cat ingress-webserver-tls.yaml 
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-webserver-tls
  namespace: ingress-tls-demo
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  tls:
  - hosts:
    - che.webserver.com
    secretName: webserver-tls-secret
  rules:
  - host: che.webserver.com
    http:
      paths:
      - path: /
        backend:
          serviceName: webapp-tls-demo
          servicePort: 80
```


#### 6、查看 nginx-ingress-controller Pod的log
```
# kubectl logs nginx-ingress-controller-6659966c64-qrw6q 
...
I1013 07:20:32.677597       6 controller.go:168] backend reload required
I1013 07:20:32.785606       6 controller.go:177] ingress backend successfully reloaded...
I1013 07:21:12.869274       6 backend_ssl.go:181] updating local copy of ssl certificate ingress-tls-demo/webserver-tls-secret with missing intermediate CA certs
I1013 07:21:12.869409       6 controller.go:168] backend reload required
I1013 07:21:12.937191       6 controller.go:177] ingress backend successfully reloaded...
```
