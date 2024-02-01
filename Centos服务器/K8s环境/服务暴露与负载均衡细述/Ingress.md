我们已经知道，**Service对集群之外暴露服务的主要方式有两种：NodePort和LoadBalancer**，但是这两种方式，都有一定的缺点：

- NodePort类型的Service进行服务暴露的时候，每个Service都需要为其打开一个端口。当Service数量比较多的时候，**端口管理变得困难**。
- LoadBalancer类型的Service的缺点是每个Service都需要一个LB，浪费，麻烦，并且需要kubernetes之外的设备的支持。
- 最重要的是，NodePort类型的服务暴露是基于四层代理转发的，无法根据HTTP的header或者path进行路由转发。

基于这种现状，kubernetes提供了Ingress资源对象，Ingress只需要一个NodePort或者一个LB就可以满足暴露多个Service的需求，工作机制大致如下图所示：
![[Pasted image 20240131210752.png]]

实际上，Ingress相当于一个七层的负载均衡器，是kubernetes对反向代理的一个抽象，它的工作原理类似于Nginx，可以理解为Ingress里面建立了诸多映射规则，**Ingress Controller通过监听这些配置规则并转化为Nginx的反向代理配置**，然后对外提供服务。

- Ingress：kubernetes中的一个对象，作用是定义请求如何转发到Service的规则。
- Ingress Controller：具体实现反向代理及负载均衡的程序，对Ingress定义的规则进行解析，根据配置的规则来实现请求转发，实现的方式有很多，比如Nginx，Contour，Haproxy等。

Ingress（以Nginx）的工作原理如下：

- 用户编写Ingress规则，说明那个域名对应kubernetes集群中的那个Service。
- Ingress控制器动态感知Ingress服务规则的变化，然后生成一段对应的Nginx的反向代理配置。
- Ingress控制器会将生成的Nginx配置写入到一个运行着的Nginx服务中，并动态更新。
- 到此为止，其实真正在工作的就是一个Nginx了，内部配置了用户定义的请求规则。
![[Pasted image 20240131210917.png]]
# 环境准备

## 搭建Ingress环境

安装ingress-controller，创建文件夹，**并进入到此文件夹中：**
```
mkdir ingress-controller
cd ingress-controller
```

 
获取ingress-nginx，本次使用的是0.30版本，网络不行，可以下载本人提供的[📎mandatory.yaml](https://www.yuque.com/attachments/yuque/0/2022/yaml/750797/1658306983955-6ac196bb-8c41-4a86-880f-effb44fa2465.yaml)[📎service-nodeport.yaml](https://www.yuque.com/attachments/yuque/0/2022/yaml/750797/1658306983969-a18df1fb-c370-445d-a6af-0dbeba2f3513.yaml)：

```
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/mandatory.yaml
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/nginx-0.30.0/deploy/static/provider/baremetal/service-nodeport.yaml
```


**修改：同时切回自己的流量，浩鲸WiFi真是无语。。。**

```
#修改mandatory .yaml文件中的仓库
#修改quay.io/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
#为quay-mirror.qiniu.com/kubernetes-ingress-controller/nginx-ingress-controller:0.30.0
```

创建Ingress-nginx：

```
kubectl apply -f ./
```


查看ingress-nginx：pod和service，注意都在ingress-nginx命名空间下：
```
kubectl get pod -n ingress-nginx
kubectl get svc -n ingress-nginx
```
![[Pasted image 20240131211015.png]]
![[Pasted image 20240131211022.png]]

## 准备Service和Pod

为了后面的实验比较方便，创建如下图所示的模型：
![[Pasted image 20240131211033.png]]

创建tomcat-nginx.yaml文件，内容如下：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx-pod
  template:
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
      - name: nginx
        image: nginx:1.17.1
        ports:
        - containerPort: 80

---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: tomcat-deployment
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      app: tomcat-pod
  template:
    metadata:
      labels:
        app: tomcat-pod
    spec:
      containers:
      - name: tomcat
        image: tomcat:8.5-jre10-slim
        ports:
        - containerPort: 8080

---

apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 80

---

apiVersion: v1
kind: Service
metadata:
  name: tomcat-service
  namespace: dev
spec:
  selector:
    app: tomcat-pod
  clusterIP: None
  type: ClusterIP
  ports:
  - port: 8080
    targetPort: 8080
```

创建Service和Pod：yaml里面已经指定了命名空间在dev，所以不需要-n dev再去指定了
```
kubectl create -f tomcat-nginx.yaml
```


查看Service和Pod：
```
kubectl get svc,pod -n dev
```
![[Pasted image 20240131211057.png]]
# Http代理

创建ingress-http.yaml文件，内容如下：
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-http
  namespace: dev
spec:
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.xudaxian.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

图解:
![[Pasted image 20240131211108.png]]
创建：

```
kubectl create -f ingress-http.yaml
```
![[Pasted image 20240131211117.png]]
查看：
```
kubectl get ingress ingress-http -n dev
```
![[Pasted image 20240131211124.png]]
查看详情：
```
kubectl describe ingress ingress-http -n dev
```
![[Pasted image 20240131211129.png]]

在你的Windows的hosts文件中添加如下的规则（**192.168.209.100为Master节点的IP地址**）：

```
192.168.18.100 nginx.xudaxian.com
192.168.18.100 tomcat.xudaxian.com
```
![[Pasted image 20240131211145.png]]

查看ingress-nginx的端口（本次测试http的端口是30378，https的端口是31125）：

```
kubectl get svc -n ingress-nginx
```
![[Pasted image 20240131211158.png]]
本机通过浏览器输入下面的地址访问：

```
http://nginx.xudaxian.com:30378
http://tomcat.xudaxian.com:30378
```
![[Pasted image 20240131211213.png]]
![[Pasted image 20240131211218.png]]

# Https代理

生成证书：

```
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 -keyout tls.key -out tls.crt -subj "/C=CN/ST=BJ/L=BJ/O=nginx/CN=xudaxian.com"
```
![[Pasted image 20240131211228.png]]

创建密钥：
```
kubectl create secret tls tls-secret --key tls.key --cert tls.crt
```
![[Pasted image 20240131211235.png]]

创建ingress-https.yaml文件，内容如下：
```
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: ingress-https
  namespace: dev
spec:
  tls:
    - hosts:
      - nginx.xudaxian.com
      - tomcat.xudaxian.com
      secretName: tls-secret # 指定秘钥
  rules:
  - host: nginx.xudaxian.com
    http:
      paths:
      - path: /
        backend:
          serviceName: nginx-service
          servicePort: 80
  - host: tomcat.xudaxian.com
    http:
      paths:
      - path: /
        backend:
          serviceName: tomcat-service
          servicePort: 8080
```

  

创建：
```
kubectl create -f ingress-https.yaml
```
![[Pasted image 20240131211246.png]]

查看：
```
kubectl get ingress ingress-https -n dev
```
![[Pasted image 20240131211251.png]]

查看详情：
```
kubectl describe ingress ingress-https -n dev
```
![[Pasted image 20240131211301.png]]

查看Service：
```
kubectl get svc -n ingress-nginx
```
![[Pasted image 20240131211307.png]]
- 在本机的hosts文件中添加如下的规则（192.168.209.100为Master节点的IP地址）：略。
- 本机通过浏览器输入下面的地址访问：

```
https://nginx.xudaxian.com:31125
https://tomcat.xudaxian.com:31125
```
