# Ingress-controller



## ingress

### 基本概念

1、外部访问集群内的服务一般通过http协议，由Api负责处理，Ingress提供负载均衡，ssl，简单来说外部访问集群内的服务的路由规则由ingress资源来控制， 

2、ingress暴露http和https路由到集群内部的service，流量路线由定义在ingress资源的规则来确定

3、ingress可以为集群内的services提供外部url，负载均衡，SSL和基于主机名的虚拟主机；ingress contrller负责ingress的实现，一般是一个负载均衡器，ingress controller可以是基于haproxy，nginx等组件 

4、代码1是一个最简单的ingress资源定义清单，其中spec中的一系列rules用来匹配所有进来的请求，ingress只提供传输HTTP的流量 

5、ingress不提供任何端口或协议，如果是需要提供端口服务，使用 Service.Type=NodePort 

### ingress类型

**单服务ingress**：K8S允许你暴露单个服务，你也可以通过ingress定义一个没有规则的默认后端，代码1

**URL ingress**：此路由规则是将流量从一个IP地址到多个服务的机制，这个机制通过HTTP URI来实现，代码2

 **虚拟主机ingress**：虚拟主机支持HTTP请求通过一个IP地址到多个主机名，从而路由到不同的services，代码3

代码1

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
spec:
  backend:
    serviceName: testsvc
    servicePort: 80
```

代码2

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: simple-fanout-example
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - path: /foo
        backend:
          serviceName: service1
          servicePort: 4200
      - path: /bar
        backend:
          serviceName: service2
          servicePort: 8080

```

代码3

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: name-virtual-host-ingress
spec:
  rules:
  - host: foo.bar.com
    http:
      paths:
      - backend:
          serviceName: service1
          servicePort: 80
  - host: bar.foo.com
    http:
      paths:
      - backend:
          serviceName: service2
          servicePort: 80

```



## ingress-controller

### 部署controller

1、部署ingress-nginx controller

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml 

代码见链接

2、部署ingress-controller service使用nodeport模式

kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/baremetal/service-nodeport.yaml

代码如下：

```yaml
kind: Service
metadata:
  name: ingress-nginx
  namespace: ingress-nginx
  labels:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx
spec:
  type: NodePort
  ports:
    - name: http
      port: 80
      targetPort: 80
      protocol: TCP
    - name: https
      port: 443
      targetPort: 443
      protocol: TCP
  selector:
    app.kubernetes.io/name: ingress-nginx
    app.kubernetes.io/part-of: ingress-nginx

---
```

3、部署ingress

Kubectl create -f ingress.yaml

代码如下：

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/x-forwarded-prefix: "/path"
spec:
  rules:
  - host: netlogin.corp.xxxx.com
    http:
      paths:
      - backend:
          serviceName: my-nginx-php-test
          servicePort: 8099
  - host: xxxxhz.com
    http:
      paths:
      - backend:
          serviceName: my-nginx-php-test
          servicePort: 8099
  - host: login.corp.xxxx.com
    http:
      paths:
      - backend:
          serviceName: my-nginx-php-test
          servicePort: 8099

```



### 支持https

  如果要使ingress支持https，需要在k8s集群中创建secret，创建secret需要指定指定证书喝密钥，这个证书和密钥可以是存在的，也可以是自己创建的



1、创建基于存在的证书&自建证书的secret

```shell
cp server.crt /root/sunpeng/k8s/ca
cp server.key /root/sunpeng/k8s/ca
cd /root/sunpeng/k8s/
kubectl create secret tls supeng-nginx-secret-1 --cert=ca/server.crt --key=ca/server.key
```

```shell
openssl genrsa -out tls.key 2048
openssl req -new -x509 -key tls.key -out tls.crt -subj /C=CN/ST=Shanghai/L=Shanghai/O=DevOps/CN=netlogin.corp.xxxx.com
kubectl create secret tls supeng-nginx-secret-1 --cert=tls.crt --key=tls.key
```

2、查看证书信息

```shell
kubectl get secret
kubectl describe  secret supeng-nginx-secret-1
```

3、修改ingress.yaml清单使用secret

```yaml
cat ingress.yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: test-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/x-forwarded-prefix: "/path”
    nginx.ingress.kubernetes.io/ssl-redirect: “false”  #这个参数会让ingress同时支持http和https，不强制重定向
spec:
  tls:
  - hosts:
    - netlogin.corp.xxxx.com
    secretName: supeng-nginx-secret-1
  rules:
  - host: netlogin.corp.xxxx.com
    http:
      paths:
      - backend:
          serviceName: my-nginx-php-test
          servicePort: 8099
  - host: xxxxhz.com
    http:
      paths:
      - backend:
          serviceName: my-nginx-php-test
          servicePort: 8099
  - host: login.corp.xxxx.com
    http:
      paths:
      - backend:
          serviceName: my-nginx-php-test
          servicePort: 8099

```



4、部署完ingress之后，查看ingress-controller来查看暴露的http和https端口来进行访问，注意，浏览器需要使用无痕模式来访问

```shell
kubectl get svc -n ingress-nginx
NAME            TYPE       CLUSTER-IP     EXTERNAL-IP   PORT(S)                      AGE
ingress-nginx   NodePort   172.30.42.41   <none>        80:23456/TCP,443:23457/TCP   110m

```



### canary金丝雀

  ingress的canary功能是利用ingress来实现流量的灰度发布功能，通过ingress 的annotations来实现ingress的canary功能和流量控制，目前ingress-nginx支持基于访问流量的控制，比如1/10的发布，还支持基于header的灰度发布

1、创建资源文件 app-v1.yaml  app-v2.yaml  ingress-v1.yaml  ingress-v2-canary.yaml  ingress-v2.yaml

2、创建资源

```shell
kubectl apply -f ./app-v1.yaml -f ./ingress-v1.yaml

```

3、测试

```shell
for i in {1..20};do curl my-app.com:23456;done
#不出意外结果应该是：Host: my-app-v1-7496f6b5bc-q9w4n, Version: v1.0.0

```

4、创建v2版本服务

```shell
kubectl apply -f ./app-v2.yaml

```

5、部署canary ingress，将流量的10%引入canary ingress，并到新的服务

```shell
kubectl apply -f ./ingress-v2-canary.yaml

```

6、测试结果

```shell
for i in {1..20};do curl my-app.com:23456;done
#不出意外应该有30%的流量访问到了v2版本的服务

```

7、当发现新版本没有问题时，就可以删除canary ingress

```shell
kubectl delete -f ./ingress-v2-canary.yaml

```

8、然后有两种方式可以全量服务至v2版本服务：一是将v2的测试集群删除，然后将v1版本的deployment的镜像版本通过滚动升级方式升至v2版本（生产环境推荐这种方式）；二是直接创建新的ingress来取代原来的ingress，然后部署（测试环境使用），本测试使用第二种方式

```shell
kubectl apply -f ./ingress-v2.yaml

```

   9、查看测试结果

```shell
for i in {1..20};do curl my-app.com:23456;done
#此时的结果应该都是v2版本的服务内容

```

10、测试完毕，clear测试环境

```shell
kubectl delete all -l app=my-app

```

11、以下为测试引用的资源清单

app-v1.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-v1
  labels:
    app: my-app
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  selector:
    app: my-app
    version: v1.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v1
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v1.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v1.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v1.0.0
        livenessProbe:
          httpGet:
            path: /live
            port: probe
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: probe
          periodSeconds: 5

```

app-v2.yaml

```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app-v2
  labels:
    app: my-app
spec:
  ports:
  - name: http
    port: 80
    targetPort: http
  selector:
    app: my-app
    version: v2.0.0
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app-v2
  labels:
    app: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
      version: v2.0.0
  template:
    metadata:
      labels:
        app: my-app
        version: v2.0.0
      annotations:
        prometheus.io/scrape: "true"
        prometheus.io/port: "9101"
    spec:
      containers:
      - name: my-app
        image: containersol/k8s-deployment-strategies
        ports:
        - name: http
          containerPort: 8080
        - name: probe
          containerPort: 8086
        env:
        - name: VERSION
          value: v2.0.0
        livenessProbe:
          httpGet:
            path: /live
            port: probe
          initialDelaySeconds: 5
          periodSeconds: 5
        readinessProbe:
          httpGet:
            path: /ready
            port: probe
          periodSeconds: 5

```

ingress-v1.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: "nginx"
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - backend:
          serviceName: my-app-v1
          servicePort: 80

```

ingress-v2-canary.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: my-app-canary
  labels:
    app: my-app
  annotations:
    kubernetes.io/ingress.class: "nginx"
    nginx.ingress.kubernetes.io/canary: "true"
    nginx.ingress.kubernetes.io/canary-weight: "1"
spec:
  rules:
  - host: my-app.com
    http:
      paths:
      - backend:
          serviceName: my-app-v2
          servicePort: 80

```

ingress-v2.yaml

```yaml
apiVersion: extensions/v1beta1
kind: Ingress
metadata: 
name: my-app
labels: 
app: my-app
annotations: 
kubernetes.io/ingress.class: "nginx"
spec: 
rules: 
- host: my-app.com
http: 
paths: 
- backend: 
serviceName: my-app-v2
servicePort: 80

```





