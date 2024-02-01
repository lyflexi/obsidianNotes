k8s 由众多组件组成，组件间通过 API 互相通信，概括起来分为Master和若干Node：
- master：集群的控制平面，负责集群的决策(管理)
- Node：集群的数据平面，负责为容器提供运行环境(干活)
Master：
- ApiServer: 资源操作的唯一入口，接收用户输入的命令，提供认证、授权、API注册和发现等机制，**kubectl命令入口**
- Scheduler : 负责集群资源调度，按照预定的调度策略将Pod调度到相应的node节点上
- ControllerManager : 负责维护集群的状态，比如程序部署安排、故障检测、自动扩展、滚动更新等
- Etcd ：负责存储集群中各种资源对象的信息，键值数据库，存放Pod对象信息
Node：
- Kubelet : 负责维护容器的生命周期，即通过控制Docker来创建、更新、销毁容器
- KubeProxy : 负责提供集群内部的服务发现和负载均衡
以上由部署工具kubeadm负责Master节点的初始化init，以及Node节点的join
![[Pasted image 20240131195831.png]]

# k8s镜像拉取策略

k8s的配置文件中经常看到有imagePullPolicy属性，这个属性是描述镜像的拉取策略
- Always 总是拉取镜像
- IfNotPresent 本地有则使用本地镜像,不拉取
- Never 只使用本地镜像，从不拉取，即使本地没有

如果省略imagePullPolicy， 默认策略为always

# Namespace命令

Namespace是kubernetes系统中的一种非常重要资源，它的主要作用是用来实现多套环境的资源隔离或者多租户的资源隔离。

默认情况下，kubernetes集群中的所有的Pod都是可以相互访问的。

但是在实际中，可能不想让两个Pod之间进行互相的访问，那此时就可以将两个Pod划分到不同的namespace下。

kubernetes通过将集群内部的资源分配到不同的Namespace中，可以形成逻辑上的"组"，以方便不同的组的资源进行隔离使用和管理。

  
可以通过kubernetes的授权机制，将不同的namespace交给不同租户进行管理，这样就实现了多租户的资源隔离。此时还能结合kubernetes的资源配额机制，限定不同租户能占用的资源，例如CPU使用量、内存使用量等等，来实现租户可用资源的管理。

  
kubernetes在集群启动之后，会默认创建几个namespace

```shell
[root@master ~]# kubectl  get namespace
NAME              STATUS   AGE
default           Active   45h     #  所有未指定Namespace的对象都会被分配在default命名空间
kube-node-lease   Active   45h     #  集群节点之间的心跳维护，v1.13开始引入
kube-public       Active   45h     #  此命名空间下的资源可以被所有人访问（包括未认证用户）
kube-system       Active   45h     #  所有由Kubernetes系统创建的资源都处于这个命名空间
```

下面来看namespace资源的具体操作：
## 基础命令方式
创建namespace：
```shell
# 创建namespace
[root@master ~]# kubectl create ns dev
namespace/dev created
```

查看namespace：
```shell
# 1 查看所有的ns  命令：kubectl get ns
[root@master ~]# kubectl get ns
NAME              STATUS   AGE
default           Active   45h
kube-node-lease   Active   45h
kube-public       Active   45h     
kube-system       Active   45h     

# 2 查看指定的ns   命令：kubectl get ns ns名称
[root@master ~]# kubectl get ns default
NAME      STATUS   AGE
default   Active   45h

# 3 指定输出格式  命令：kubectl get ns ns名称  -o 格式参数
# kubernetes支持的格式有很多，比较常见的是wide、json、yaml
[root@master ~]# kubectl get ns default -o yaml
apiVersion: v1
kind: Namespace
metadata:
  creationTimestamp: "2020-04-05T04:44:16Z"
  name: default
  resourceVersion: "151"
  selfLink: /api/v1/namespaces/default
  uid: 7405f73a-e486-43d4-9db6-145f1409f090
spec:
  finalizers:
  - kubernetes
status:
  phase: Active
  
# 4 查看ns详情  命令：kubectl describe ns ns名称
[root@master ~]# kubectl describe ns default
Name:         default
Labels:       <none>
Annotations:  <none>
Status:       Active  # Active 命名空间正在使用中  Terminating 正在删除命名空间

# ResourceQuota 针对namespace做的资源限制
# LimitRange针对namespace中的每个组件做的资源限制
No resource quota.
No LimitRange resource.

#查看namespace下的所有资源：
kubectl get all -n dev

```

删除namespace
```shell
[root@master ~]# kubectl delete ns dev
namespace "dev" deleted
```
## yaml创建方式
配置yaml方式创建namespace，准备一个yaml文件：ns-dev.yaml
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: dev
```

然后就可以执行对应的创建和删除命令了：

```shell
#创建：
kubectl  create  -f  ns-dev.yaml
#删除：
kubectl  delete  -f  ns-dev.yaml
```

# Pod命令

Pod是kubernetes集群进行管理的最小单元，程序要运行必须部署在容器中，而容器必须存在于Pod中。

Pod可以认为是容器的封装，一个Pod中可以存在一个或者多个容器。
![[Pasted image 20240131200606.png]]

kubernetes在集群启动之后，集群中的各个组件也都是以Pod方式运行的。可以通过下面命令查看：

```shell
[root@master ~]# kubectl get pod -n kube-system
NAMESPACE     NAME                             READY   STATUS    RESTARTS   AGE
kube-system   coredns-6955765f44-68g6v         1/1     Running   0          2d1h
kube-system   coredns-6955765f44-cs5r8         1/1     Running   0          2d1h
kube-system   etcd-master                      1/1     Running   0          2d1h
kube-system   kube-apiserver-master            1/1     Running   0          2d1h
kube-system   kube-controller-manager-master   1/1     Running   0          2d1h
kube-system   kube-flannel-ds-amd64-47r25      1/1     Running   0          2d1h
kube-system   kube-flannel-ds-amd64-ls5lh      1/1     Running   0          2d1h
kube-system   kube-proxy-685tk                 1/1     Running   0          2d1h
kube-system   kube-proxy-87spt                 1/1     Running   0          2d1h
kube-system   kube-scheduler-master            1/1     Running   0          2d1h
```
## 基础命令方式
创建并运行pod，kubernetes没有提供单独运行Pod的命令，都是通过Pod控制器来实现的

```shell
# 命令格式： kubectl run (pod控制器名称) [参数] 
# --image  指定Pod的镜像
# --port   指定端口
# --namespace  指定namespace
[root@master ~]# kubectl run nginx --image=nginx:1.17.1 --port=80 --namespace dev 
deployment.apps/nginx created
```

查看podIP
```shell
# 获取podIP
[root@master ~]# kubectl get pods -n dev -o wide
NAME                     READY   STATUS    RESTARTS   AGE    IP             NODE    ... 
nginx-5ff7956ff6-fg2db   1/1     Running   0          190s   10.244.1.23   node1   ...
```

访问POD：
```shell
[root@master ~]# curl http://10.244.1.23:80
<!DOCTYPE html>
<html>
<head>
        <title>Welcome to nginx!</title>
</head>
<body>
        <p><em>Thank you for using nginx.</em></p>
</body>
</html>
```



查看Pod详细信息：describe
```shell
# 查看Pod基本信息
[root@master ~]# kubectl get pods -n dev
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5ff7956ff6-fg2db   1/1     Running   0          43s

# 查看Pod的详细信息
[root@master ~]# kubectl describe pod nginx-5ff7956ff6-fg2db -n dev
Name:         nginx-5ff7956ff6-fg2db
Namespace:    dev
Priority:     0
Node:         node1/192.168.109.101
Start Time:   Wed, 08 Apr 2020 09:29:24 +0800
Labels:       pod-template-hash=5ff7956ff6
              run=nginx
Annotations:  <none>
Status:       Running
IP:           10.244.1.23
IPs:
  IP:           10.244.1.23
Controlled By:  ReplicaSet/nginx-5ff7956ff6
Containers:
  nginx:
    Container ID:   docker://4c62b8c0648d2512380f4ffa5da2c99d16e05634979973449c98e9b829f6253c
    Image:          nginx:1.17.1
    Image ID:       docker-pullable://nginx@sha256:485b610fefec7ff6c463ced9623314a04ed67e3945b9c08d7e53a47f6d108dc7
    Port:           80/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Wed, 08 Apr 2020 09:30:01 +0800
    Ready:          True
    Restart Count:  0
    Environment:    <none>
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from default-token-hwvvw (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  default-token-hwvvw:
    Type:        Secret (a volume populated by a Secret)
    SecretName:  default-token-hwvvw
    Optional:    false
QoS Class:       BestEffort
Node-Selectors:  <none>
Tolerations:     node.kubernetes.io/not-ready:NoExecute for 300s
                 node.kubernetes.io/unreachable:NoExecute for 300s
Events:
  Type    Reason     Age        From               Message
  ----    ------     ----       ----               -------
  Normal  Scheduled  <unknown>  default-scheduler  Successfully assigned dev/nginx-5ff7956ff6-fg2db to node1
  Normal  Pulling    4m11s      kubelet, node1     Pulling image "nginx:1.17.1"
  Normal  Pulled     3m36s      kubelet, node1     Successfully pulled image "nginx:1.17.1"
  Normal  Created    3m36s      kubelet, node1     Created container nginx
  Normal  Started    3m36s      kubelet, node1     Started container nginx
```

查看Pod信息，格式化输出
```Ruby
kubectl get pod aitraining-master-164-5b979f4d-5hflq -n ai-train -o yaml
```

进入docker容器 ：
```Plain
docker exec -ti  <your-container-name>   /bin/sh
```

进入Kubernetes的pod：

```Plain
kubectl exec -ti <your-pod-name>  -n <your-namespace>  -- /bin/sh
```

删除Pod：
```shell
[root@master ~]# kubectl delete pod nginx-5ff7956ff6-fg2db -n dev
pod "nginx-5ff7956ff6-fg2db" deleted

# 此时，显示删除Pod成功，但是再查询，发现又新产生了一个 
[root@master ~]# kubectl get pods -n dev
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5ff7956ff6-jj4ng   1/1     Running   0          21s
```

这是因为当前Pod是由Pod控制器创建的，控制器会监控Pod状况，一旦发现Pod死亡，会立即重建

此时要想删除Pod，必须删除Pod控制器
```shell
# 先来查询一下当前namespace下的Pod控制器
[root@master ~]# kubectl get deploy -n  dev
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   1/1     1            1           9m7s

#接下来，删除此PodPod控制器
[root@master ~]# kubectl delete deploy nginx -n dev
deployment.apps "nginx" deleted
# 稍等片刻，再查询Pod，发现Pod被删除了
[root@master ~]# kubectl get pods -n dev
No resources found in dev namespace.
```

异常情况，执行完删除命令以后看到pod一直处于terminating的状态。则可以执行强制删除pod：
```shell
kubectl delete pod [pod name] --force --grace-period=0 -n [namespace]
注意：必须加-n参数指明namespace，否则可能报错pod not found。
```

批量删除Evicted pod，方法一（shell脚本）：
```Java
#!/bin/bash
for each in $(kubectl get pods -n NameSpace|grep Evicted|awk '{print $1}');
do
  kubectl delete pods $each -n NameSpace
done
```

批量删除Evicted pod，方法二（简单命令）：

```Java
kubectl get pods -n NameSpace | grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n NameSpace
kubectl get pods -n kube-fate | grep Evicted | awk '{print $1}' | xargs kubectl delete pod -n kube-fate
```

## yaml创建方式

创建一个pod-nginx.yaml，内容如下：
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
spec:
  containers:
  - image: nginx:1.17.1
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

```shell
#创建：
kubectl  create  -f  pod-nginx.yaml
#删除：
kubectl  delete  -f  pod-nginx.yaml
```

# Label命令

Label是kubernetes系统中的一个重要概念。它的作用就是在资源上添加标识，用来对它们进行区分和选择。
- 一个Label会以key/value键值对的形式附加到各种对象上，如Node、Pod、Service等等
- 一个资源对象可以定义任意数量的Label ，同一个Label也可以被添加到任意数量的资源对象上去
- Label通常在资源对象定义时确定，当然也可以在对象创建后动态添加或者删除

可以通过Label实现资源的多维度分组，以便灵活、方便地进行资源分配、调度、配置、部署等管理工作。

一些常用的Label 示例如下：

```shell
版本标签："version":"release", "version":"stable"......
环境标签："environment":"dev"，"environment":"test"，"environment":"pro"
架构标签："tier":"frontend"，"tier":"backend"
```

标签定义完毕之后，还要考虑到标签的选择，这就要使用到Label Selector用于查询和筛选拥有某些标签的资源对象

当前有两种Label Selector：

基于等式的Label Selector：

```shell
name = slave: 选择所有包含Label中key="name"且value="slave"的对象
env != production: 选择所有包括Label中的key="env"且value不等于"production"的对象
```

基于集合的Label Selector：

```shell
name in (master, slave): 选择所有包含Label中的key="name"且value="master"或"slave"的对象
name not in (frontend): 选择所有包含Label中的key="name"且value不等于"frontend"的对象
```

标签的选择条件可以使用多个，此时将多个Label Selector进行组合，使用逗号","进行分隔即可。例如：

```shell
name=slave，env!=production
name not in (frontend)，env!=production
```
## 基础命令方式
给pod资源打标签

```shell
[root@master ~]# kubectl label pod nginx-pod version=1.0 -n dev
pod/nginx-pod labeled
```

更新pod资源的标签
```shell
[root@master ~]# kubectl label pod nginx-pod version=2.0 -n dev --overwrite
pod/nginx-pod labeled
```

查看标签

```shell
[root@master ~]# kubectl get pod nginx-pod  -n dev --show-labels
NAME        READY   STATUS    RESTARTS   AGE   LABELS
nginx-pod   1/1     Running   0          10m   version=2.0
```

筛选标签
```shell
[root@master ~]# kubectl get pod -n dev -l version=2.0  --show-labels
NAME        READY   STATUS    RESTARTS   AGE   LABELS
nginx-pod   1/1     Running   0          17m   version=2.0
[root@master ~]# kubectl get pod -n dev -l version!=2.0 --show-labels
No resources found in dev namespace.

[root@master ~]# kubectl get all -l environment=production,tier=frontend
```

删除标签
```shell
[root@master ~]# kubectl label pod nginx-pod version- -n dev labelName-
pod/nginx-pod labeled
```

## yaml创建方式
配置文件方式
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: dev
  labels:
    version: "3.0" 
    env: "test"
spec:
  containers:
  - image: nginx:1.17.1
    name: pod
    ports:
    - name: nginx-port
      containerPort: 80
      protocol: TCP
```

然后就可以执行对应的更新命令了：

```shell
kubectl  apply  -f  pod-nginx.yaml
```

# Deployment命令

在kubernetes中，Pod是最小的控制单元，但是kubernetes很少直接控制Pod，一般都是通过Pod控制器来完成的。

Pod控制器用于pod的管理，确保pod资源符合预期的状态，当pod的资源出现故障时，会尝试进行重启或重建pod。

在kubernetes中Pod控制器的种类有很多，本章节只介绍一种：Deployment。
![[Pasted image 20240131201620.png]]

deployment控制器和pod之间是通过label产生关联的
## 基础命令方式
创建deployment:
```yaml
# 命令格式: kubectl run deployment名称  [参数] 
# --image  指定pod的镜像
# --port   指定端口
# --replicas  指定创建pod数量
# --namespace  指定namespace
[root@master ~]# kubectl run nginx --image=nginx:1.17.1 --port=80 --replicas=3 -n dev
deployment.apps/nginx created
```

查看创建的Pod:
```yaml
[root@master ~]# kubectl get pods -n dev
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5ff7956ff6-6k8cb   1/1     Running   0          19s
nginx-5ff7956ff6-jxfjt   1/1     Running   0          19s
nginx-5ff7956ff6-v6jqw   1/1     Running   0          19s
```

查看deployment的信息:
```yaml
[root@master ~]# kubectl get deploy -n dev
NAME    READY   UP-TO-DATE   AVAILABLE   AGE
nginx   3/3     3            3           2m42s

# UP-TO-DATE：成功升级的副本数量
# AVAILABLE：可用副本的数量

[root@master ~]# kubectl get deploy -n dev -o wide
NAME    READY UP-TO-DATE  AVAILABLE   AGE     CONTAINERS   IMAGES              SELECTOR
nginx   3/3     3         3           2m51s   nginx        nginx:1.17.1        run=nginx
```

查看deployment的详细信息:
```yaml
[root@master ~]# kubectl describe deploy nginx -n dev
Name:                   nginx
Namespace:              dev
CreationTimestamp:      Wed, 08 Apr 2020 11:14:14 +0800
Labels:                 run=nginx
Annotations:            deployment.kubernetes.io/revision: 1
Selector:               run=nginx
Replicas:               3 desired | 3 updated | 3 total | 3 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  run=nginx
  Containers:
   nginx:
    Image:        nginx:1.17.1
    Port:         80/TCP
    Host Port:    0/TCP
    Environment:  <none>
    Mounts:       <none>
  Volumes:        <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   nginx-5ff7956ff6 (3/3 replicas created)
Events:
  Type    Reason             Age    From                   Message
  ----    ------             ----   ----                   -------
  Normal  ScalingReplicaSet  5m43s  deployment-controller  Scaled up replicaset nginx-5ff7956ff6 to 3
  
```

删除deployment：
```yaml
[root@master ~]# kubectl delete deploy nginx -n dev
deployment.apps "nginx" deleted
```
## yaml创建方式
创建一个deploy-nginx.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:#####################################################pod模板
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:1.17.1
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

然后就可以执行对应的创建和删除命令了：

```shell
#创建：
kubectl  create  -f  deploy-nginx.yaml
#删除：
kubectl  delete  -f  deploy-nginx.yaml
```

deploy重启:

```shell
kubectl rollout restart deploy <deployment-name> -n <namespace>
```

# Service命令

通过上节课的学习，已经能够利用Deployment来创建一组Pod来提供具有高可用性的服务。

虽然每个Pod都会分配一个单独的Pod IP，然而却存在如下两问题：
- Pod IP 会随着Pod删除自动重建而产生变化
- Pod IP 仅仅是集群内可见的虚拟IP，外部无法访问


这样对于访问这个服务带来了难度。因此，kubernetes设计了Service来解决这个问题。

Service可以看作是一组同类Pod对外的访问接口。借助Service，应用可以方便地实现服务发现和负载均衡。

service跟deployment一样，也是通过标签选择器去实现与一组pods相关联的。
![[Pasted image 20240131201822.png]]

首先还是创建pod或者deployment：

有如下配置文件：

创建一个deploy-nginx.yaml，内容如下：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx
  namespace: dev
spec:
  replicas: 3
  selector:
    matchLabels:
      run: nginx
  template:#####################################################pod模板
    metadata:
      labels:
        run: nginx
    spec:
      containers:
      - image: nginx:1.17.1
        name: nginx
        ports:
        - containerPort: 80
          protocol: TCP
```

创建3个pod：

```yaml
kubectl  create  -f  deploy-nginx.yaml
```
## 创建ClusterIP服务
操作一：创建集群内部可访问的Service

```yaml
# 暴露Service
[root@master ~]# kubectl expose deploy nginx --name=svc-nginx1 --type=ClusterIP --port=80 --target-port=80 -n dev
service/svc-nginx1 exposed

#--type=ClusterIP --port=80 --target-port=80 -n dev
service设定pord=80,转发到后面端口为80的pod
```

查看service
```yaml
[root@master ~]# kubectl get svc svc-nginx1 -n dev -o wide
NAME         TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE     SELECTOR
svc-nginx1   ClusterIP   10.109.179.231   <none>        80/TCP    3m51s   run=nginx
```

这里产生了一个CLUSTER-IP，这就是service的IP，在Service的生命周期中，这个地址是不会变动的

可以通过这个IP访问当前service对应的POD
```yaml
[root@master ~]# curl 10.109.179.231:80
<!DOCTYPE html>
<html>
<head>
<title>Welcome to nginx!</title>
</head>
<body>
<h1>Welcome to nginx!</h1>
.......
</body>
</html>
```
## 创建NodePort服务
操作二：创建集群外部也可访问的Service

如果需要创建外部也可以访问的Service，需要修改type为NodePort

```yaml
[root@master ~]# kubectl expose deploy nginx --name=svc-nginx2 --type=NodePort --port=80 --target-port=80 -n dev
service/svc-nginx2 exposed
```

此时查看，会发现出现了NodePort类型的Service，而且有一对Port（80:31928/TCP）

```yaml
[root@master ~]# kubectl get svc  svc-nginx2  -n dev -o wide
NAME          TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)        AGE    SELECTOR
svc-nginx2    NodePort    10.100.94.0      <none>        80:31928/TCP   9s     run=nginx
```

接下来就可以通过集群外的主机访问 节点IP:31928访问服务了

例如在的电脑主机上通过浏览器访问下面的地址：远程服务器IP：你创建的service 集群外部可以访问的暴露的端口
![[Pasted image 20240131202015.png]]
![[Pasted image 20240131202020.png]]

删除Service
```shell
[root@master ~]# kubectl delete svc svc-nginx-1 -n dev                                   
service "svc-nginx-1" deleted
```

配置文件方式创建一个svc-nginx.yaml，内容如下：
```yaml
apiVersion: v1
kind: Service
metadata:
  name: svc-nginx
  namespace: dev
spec:
  clusterIP: 10.109.179.231
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    run: nginx
  type: ClusterIP
```

然后就可以执行对应的创建和删除命令了：

```shell
#创建：
kubectl  create  -f  svc-nginx.yaml
#删除：
kubectl  delete  -f  svc-nginx.yaml
```