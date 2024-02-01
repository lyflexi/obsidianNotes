主要介绍各种Pod控制器的详细使用。

# 1 Pod控制器的介绍

- 在kubernetes中，按照Pod的创建方式可以将其分为两类：

- 自主式Pod：kubernetes直接创建出来的Pod，这种Pod删除后就没有了，也不会重建。
- 控制器创建Pod：通过Pod控制器创建的Pod，这种Pod删除之后还会自动重建。**创建出来的pod名字是以deployment名字开头后面跟随机数**

- Pod控制器：Pod控制器是管理Pod的中间层，使用了Pod控制器之后，我们只需要告诉Pod控制器，想要多少个什么样的Pod就可以了，它就会创建出满足条件的Pod并确保每一个Pod处于用户期望的状态，如果Pod在运行中出现故障，控制器会基于指定的策略重启或重建Pod。
- 在kubernetes中，有很多类型的Pod控制器，每种都有自己的适合的场景，常见的有下面这些：

- ReplicationController：比较原始的Pod控制器，已经被废弃，由ReplicaSet替代。
- **ReplicaSet**：保证指定数量的Pod运行，并支持Pod数量变更，镜像版本变更。
- **Deployment**：通过控制ReplicaSet来控制Pod，并支持滚动升级、版本回退。
- Horizontal Pod Autoscaler：可以根据集群负载自动调整Pod的数量，实现削峰填谷。
- DaemonSet：在集群中的指定Node上都运行一个副本，一般用于守护进程类的任务。
- Job：它创建出来的Pod只要完成任务就立即退出，用于执行一次性任务。
- CronJob：它创建的Pod会周期性的执行，用于执行周期性的任务。
- StatefulSet：管理有状态的应用。

# 2 ReplicaSet（RS）

  

## 2.1 概述

  

- ReplicaSet的主要作用是保证一定数量的Pod能够正常运行，它会持续监听这些Pod的运行状态，一旦Pod发生故障，就会重启或重建。
- 同时它还支持对Pod数量的扩缩容和版本镜像的升级。
![[Pasted image 20240131203712.png]]

  

- ReplicaSet的资源清单文件：

  

```
apiVersion: apps/v1 # 版本号 
kind: ReplicaSet # 类型 
metadata: # 元数据 
  name: # rs名称
  namespace: # 所属命名空间 
  labels: #标签 
    controller: rs 
spec: # 详情描述 
  replicas: 3 # 副本数量 
  selector: # 选择器，通过它指定该控制器管理哪些po
    matchLabels: # Labels匹配规则 
      app: nginx-pod 
    matchExpressions: # Expressions匹配规则 
      - {key: app, operator: In, values: [nginx-pod]} 
template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本 
  metadata: 
    labels: 
      app: nginx-pod 
  spec: 
    containers: 
      - name: nginx 
        image: nginx:1.17.1 
        ports: 
        - containerPort: 80
```

  

- 在这里，需要新了解的配置项就是spec下面几个选项：

- replicas：指定副本数量，其实就是当然rs创建出来的Pod的数量，默认为1.
- selector：选择器，它的作用是建立Pod控制器和Pod之间的关联关系，采用了Label Selector机制（在Pod模块上定义Label，在控制器上定义选择器，就可以表明当前控制器能管理哪些Pod了）。
- template：模板，就是当前控制器创建Pod所使用的模板，里面其实就是前面学过的Pod的定义。

  

## 2.2 创建ReplicaSet

  

- 创建pc-replicaset.yaml文件，内容如下：

  

```
apiVersion: apps/v1 # 版本号
kind: ReplicaSet # 类型
metadata: # 元数据
  name: pc-replicaset # rs名称
  namespace: dev # 命名类型
spec: # 详细描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器可以管理哪些Pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
  template: # 模块 当副本数据不足的时候，会根据下面的模板创建Pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx # 容器名称
          image: nginx:1.17.1 # 容器需要的镜像地址
          ports:
            - containerPort: 80 # 容器所监听的端口
```

  

- 创建rs：

  

```
kubectl create -f pc-replicaset.yaml
```

  

![[Pasted image 20240131203722.png]]

  

- 查看rs：

  

```
kubectl get rs pc-replicaset -n dev -o wide
```

  
![[Pasted image 20240131203727.png]]

  

- 查看当前控制器创建出来的Pod（控制器创建出来的Pod的名称是在控制器名称后面拼接了-xxx随机码）：

  

```
kubectl get pod -n dev
```
![[Pasted image 20240131203734.png]]

## 2.3 扩缩容

  

- 编辑rs的副本数量，修改spec:replicas:6即可。

  

```
kubectl edit rs pc-replicaset -n dev
```
![[扩缩容之编辑rs的副本数量.gif]]

  

- 使用scale命令实现扩缩容，后面加上--replicas=n直接指定目标数量即可。

  

```
kubectl scale rs pc-replicaset --replicas=2 -n dev
```
![[Pasted image 20240131203814.png]]

  

## 2.4 镜像升级

  

- 编辑rs的容器镜像，修改spec:containers:image为nginx:1.17.2即可。

  

```
kubectl edit rs pc-replicaset -n dev
```
![[镜像升级之编辑rs的容器镜像.gif]]

- 使用set命令实现镜像升级。

  

```
# 语法
kubectl set image rs rs名称 容器名称=镜像版本 -n 命名空间
```

  

```
kubectl set image rs pc-replicaset nginx=nginx:1.17.1 -n dev
```
![[Pasted image 20240131203839.png]]
## 2.5 删除ReplicaSet

  

- 使用kubectl delete rs 命令会删除ReplicaSet和其管理的Pod。

  

```
# 在kubernetes删除ReplicaSet前，会将ReplicaSet的replicas调整为0，等到所有的Pod被删除后，再执行ReplicaSet对象的删除
kubectl delete rs pc-replicaset -n dev
```
![[Pasted image 20240131203846.png]]

- 如果希望仅仅删除ReplicaSet对象（保留Pod），只需要在使用kubectl delete rs命令的时候添加--cascade=false选项（不推荐）：

  

```
kubectl delete rs pc-replicaset -n dev --cascade=false
```

  

- 使用yaml直接删除（推荐）：

  

```
kubectl delete -f pc-replicaset.yaml
```

  

# 3 Deployment（Deploy）

  

## 3.1 概述

  

- 为了更好的解决服务编排的问题，kubernetes在v1.2版本开始，引入了Deployment控制器。值得一提的是，Deployment控制器并不直接管理Pod，而是通过管理ReplicaSet来间接管理Pod，即：Deployment管理ReplicaSet，ReplicaSet管理Pod。所以Deployment的功能比ReplicaSet强大。
![[Pasted image 20240131203853.png]]

- Deployment的主要功能如下：

- 支持ReplicaSet的所有功能。
- 支持发布的停止、继续。
- 支持版本滚动更新和版本回退。

- Deployment的资源清单：

  

```
apiVersion: apps/v1 # 版本号 
kind: Deployment # 类型 
metadata: # 元数据 
  name: # rs名称 
  namespace: # 所属命名空间 
  labels: #标签 
    controller: deploy 
spec: # 详情描述 
  replicas: 3 # 副本数量 
  revisionHistoryLimit: 3 # 保留历史版本，默认为10 
  paused: false # 暂停部署，默认是false 
  progressDeadlineSeconds: 600 # 部署超时时间（s），默认是600 
  strategy: # 策略 
    type: RollingUpdate # 滚动更新策略 
    rollingUpdate: # 滚动更新 
      maxSurge: 30% # 最大额外可以存在的副本数，可以为百分比，也可以为整数 maxUnavailable: 30% # 最大不可用状态的    Pod 的最大值，可以为百分比，也可以为整数 
  selector: # 选择器，通过它指定该控制器管理哪些pod 
    matchLabels: # Labels匹配规则 
      app: nginx-pod 
    matchExpressions: # Expressions匹配规则 
      - {key: app, operator: In, values: [nginx-pod]} 
  template: # 模板，当副本数量不足时，会根据下面的模板创建pod副本 
    metadata: 
      labels: 
        app: nginx-pod 
    spec: 
      containers: 
      - name: nginx 
        image: nginx:1.17.1 
        ports: 
        - containerPort: 80
```

  

## 3.2 创建Deployment

  

- 创建pc-deployment.yaml文件，内容如下：

  

```
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型
metadata: # 元数据
  name: pc-deployment # deployment的名称
  namespace: dev # 命名类型
spec: # 详细描述
  replicas: 3 # 副本数量
  selector: # 选择器，通过它指定该控制器可以管理哪些Pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
  template: # 模块 当副本数据不足的时候，会根据下面的模板创建Pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx # 容器名称
          image: nginx:1.17.1 # 容器需要的镜像地址
          ports:
            - containerPort: 80 # 容器所监听的端口
```

  

- 创建Deployment：

  

```
kubectl create -f pc-deployment.yaml
```
![[Pasted image 20240131203903.png]]
- 查看Deployment：

  

```
# UP-TO-DATE 最新版本的Pod数量
# AVAILABLE 当前可用的Pod数量
kubectl get deploy pc-deployment -n dev
```
![[Pasted image 20240131203908.png]]

- 查看ReplicaSet：

  

```
kubectl get rs -n dev
```
![[Pasted image 20240131203916.png]]
- 查看Pod：

  

```
kubectl get pod -n dev
```

![[Pasted image 20240131203923.png]]
## 3.3 扩缩容

  

- 使用scale命令实现扩缩容：

  

```
kubectl scale deploy pc-deployment --replicas=5 -n dev
```
![[Pasted image 20240131203929.png]]

- 编辑Deployment的副本数量，修改spec:replicas:4即可。

  

```
kubectl edit deployment pc-deployment -n dev
```

![[扩缩容之编辑Deployment副本数量.gif]]
## 3.4 镜像更新

  

### 3.4.1 概述

  

- Deployment支持两种镜像更新的策略：`重建更新`和`滚动更新（默认）`，可以通过`strategy`选项进行配置。

  

```
strategy: 指定新的Pod替代旧的Pod的策略，支持两个属性
  type: 指定策略类型，支持两种策略
    Recreate：在创建出新的Pod之前会先杀掉所有已经存在的Pod
    RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本的Pod
  rollingUpdate：当type为RollingUpdate的时候生效，用于为rollingUpdate设置参数，支持两个属性：
    maxUnavailable：用来指定在升级过程中不可用的Pod的最大数量，默认为25%。
    maxSurge： 用来指定在升级过程中可以超过期望的Pod的最大数量，默认为25%。
```

  

### 3.4.2 重建更新

  

- 编辑pc-deployment.yaml文件，在spec节点下添加更新策略

  

```
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型
metadata: # 元数据
  name: pc-deployment # deployment的名称
  namespace: dev # 命名类型
spec: # 详细描述
  replicas: 3 # 副本数量
  strategy: # 镜像更新策略
    type: Recreate # Recreate：在创建出新的Pod之前会先杀掉所有已经存在的Pod
  selector: # 选择器，通过它指定该控制器可以管理哪些Pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
  template: # 模块 当副本数据不足的时候，会根据下面的模板创建Pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx # 容器名称
          image: nginx:1.17.1 # 容器需要的镜像地址
          ports:
            - containerPort: 80 # 容器所监听的端口
```

  

- 更新Deployment：

  

```
kubectl apply -f pc-deployment.yaml
```

  

- 镜像升级：

  

```
kubectl set image deployment pc-deployment nginx=nginx:1.17.2 -n dev
```

![[Pasted image 20240131204005.png]]

- 查看升级过程：

  

```
kubectl get pod -n dev -w
```
![[Pasted image 20240131204011.png]]

### 3.4.3 滚动更新

  

- 编辑pc-deployment.yaml文件，在spec节点下添加更新策略：

  

```
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型
metadata: # 元数据
  name: pc-deployment # deployment的名称
  namespace: dev # 命名类型
spec: # 详细描述
  replicas: 3 # 副本数量
  strategy: # 镜像更新策略
    type: RollingUpdate # RollingUpdate：滚动更新，就是杀死一部分，就启动一部分，在更新过程中，存在两个版本的Pod
    rollingUpdate:
      maxUnavailable: 25%
      maxSurge: 25%
  selector: # 选择器，通过它指定该控制器可以管理哪些Pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
  template: # 模块 当副本数据不足的时候，会根据下面的模板创建Pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx # 容器名称
          image: nginx:1.17.1 # 容器需要的镜像地址
          ports:
            - containerPort: 80 # 容器所监听的端口
```

  

- 更新Deployment：

  

```
kubectl apply -f pc-deployment.yaml
```

  

- 镜像升级：

  

```
kubectl set image deployment pc-deployment nginx=nginx:1.17.3 -n dev
```

  
![[Pasted image 20240131204020.png]]

- 查看升级过程：

  

```
kubectl get pod -n dev -w
```

![[Pasted image 20240131204025.png]]

- 滚动更新的过程：
![[Pasted image 20240131204031.png]]

- 镜像更新中rs的变化：

  

```
# 查看rs，发现原来的rs依旧存在，只是Pod的数量变为0，而后又产生了一个rs，目标Pod的数量变为3
# 其实这就是deployment能够进行版本回退的奥妙所在
kubectl get rs -n dev
```
![[Pasted image 20240131204040.png]]
![[Pasted image 20240131204050.png]]

  

## 3.5 版本回退

  

- Deployment支持版本升级rollout过程中的暂停、继续功能以及版本回退等诸多功能，下面具体来看：

  

```
# 版本升级相关功能
kubetl rollout 参数 deploy xx  # 支持下面的选择
# status 显示当前升级的状态
# history 显示升级历史记录
# pause 暂停版本升级过程
# resume 继续已经暂停的版本升级过程
# restart 重启版本升级过程
# undo 回滚到上一级版本 （可以使用--to-revision回滚到指定的版本）
```

  

- 查看当前升级版本的状态：

  

```
kubectl rollout status deployment pc-deployment -n dev
```

![[Pasted image 20240131204056.png]]

- 查看升级历史记录：

  

```
kubectl rollout history deployment pc-deployment -n dev
```
![[Pasted image 20240131204105.png]]

- 版本回退：

  

```
# 可以使用-to-revision=1回退到1版本，如果省略这个选项，就是回退到上个版本，即2版本
kubectl rollout undo deployment pc-deployment --to-revision=1 -n dev
```
![[Pasted image 20240131204111.png]]

deployment之所以能够实现版本的回退，就是通过记录下历史的ReplicaSet来实现的，

一旦想回滚到那个版本，只需要将当前版本的Pod数量降为0，然后将回退版本的Pod提升为目标数量即可。

  

## 3.6 金丝雀发布

**Deployment支持更新过程中的控制，如暂停更新操作（pause）或继续更新操作（resume）。**

- 例如有一批新的Pod资源创建完成后立即暂停更新过程，此时，仅存在一部分新版本的应用，主体部分还是旧的版本。然后，再筛选一小部分的用户请求到新版本的Pod应用，继续观察能够稳定的按照期望的方式运行，如果没有问题之后再继续完成余下的Pod资源的滚动更新，否则立即回滚操作。
- 更新Deployment的版本，并配置暂停Deployment：

```
kubectl set image deployment pc-deployment nginx=nginx:1.17.4 -n dev && kubectl rollout pause deployment pc-deployment -n dev
```

![[Pasted image 20240131204117.png]]

观察更新状态：
```
kubectl rollout status deployment pc-deployment -n dev
```

  ![[Pasted image 20240131204124.png]]

监控更新的过程，可以看到已经新增了一个资源，但是并没有按照预期的状态去删除一个旧的资源，因为使用了pause暂停命令：
```
kubectl get rs -n dev -o wide
```
![[Pasted image 20240131204130.png]]
  

查看Pod：
```
kubectl get pod -n dev
```

![[Pasted image 20240131204141.png]]

确保更新的Pod没问题之后，继续更新：
```
kubectl rollout resume deployment pc-deployment -n dev
```
![[Pasted image 20240131204147.png]]

查看最后的更新情况：
```
kubectl get rs -n dev -o wide
kubectl get pod -n dev
```

![[Pasted image 20240131204154.png]]

## 3.7 删除Deployment

  
删除Deployment，其下的ReplicaSet和Pod也会一起被删除：

```
kubectl delete -f pc-deployment.yaml
```
![[Pasted image 20240131204237.png]]
# 4 Horizontal Pod Autoscaler（HPA）

  

## 4.1 概述

  

- 我们已经可以通过手动执行`kubectl scale`命令实现Pod的扩缩容，但是这显然不符合kubernetes的定位目标–自动化和智能化。kubernetes期望可以通过监测Pod的使用情况，实现Pod数量的自动调整，于是就产生了HPA这种控制器。
- HPA可以获取每个Pod的利用率，然后和HPA中定义的指标进行对比，同时计算出需要伸缩的具体值，最后实现Pod的数量的调整。其实HPA和之前的Deployment一样，也属于一种kubernetes资源对象，它通过追踪分析目标Pod的负载变化情况，来确定是否需要针对性的调整目标Pod的副本数。

![[Pasted image 20240131204242.png]]
## 4.2 安装metrics-server（v0.3.6）

  

- metrics-server可以用来收集集群中的资源使用情况。
- 获取metrics-server，需要注意使用的版本（网路不行，请点这里[📎v0.3.6.tar.gz](https://www.yuque.com/attachments/yuque/0/2021/gz/750797/1625710040218-25fc0a60-3142-4461-8021-014cd69894aa.gz)）：

```
wget https://github.com/kubernetes-sigs/metrics-server/archive/v0.3.6.tar.gz
tar -zxvf v0.3.6.tar.gz
```

  

进入metrics-server-0.3.6/deploy/1.8+/目录：
```
cd metrics-server-0.3.6/deploy/1.8+/
```

修改metrics-server-deployment.yaml文件：
```
vim metrics-server-deployment.yaml
```
按图中添加下面选项
```
hostNetwork: true
image: registry.cn-hangzhou.aliyuncs.com/google_containers/metrics-server-amd64:v0.3.6 
args:
  - --kubelet-insecure-tls 
  - --kubelet-preferred-address-types=InternalIP,Hostname,InternalDNS,ExternalDNS,ExternalIP
```
![[Pasted image 20240131204249.png]]

  

安装metrics-server：
```
kubectl apply -f ./
```
![[Pasted image 20240131204338.png]]
查看metrics-server生成的Pod：

```
kubectl get pod -n kube-system
```
![[Pasted image 20240131204348.png]]

  查看资源使用情况：

```
kubectl top node
kubectl top pod -n kube-system
```
![[Pasted image 20240131204403.png]]

  

## 4.3 安装metrics-server（v0.4.1）

获取metrics-server：
```
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.1/components.yaml
```

修改components.yaml（修改之后的components.yaml文件[📎components.yaml](https://www.yuque.com/attachments/yuque/0/2021/yaml/750797/1625710040206-21b1015d-4e0a-4d90-a4e1-46a33876b059.yaml)）：
```
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    k8s-app: metrics-server
  name: metrics-server
  namespace: kube-system
spec:
  selector:
    matchLabels:
      k8s-app: metrics-server
  strategy:
    rollingUpdate:
      maxUnavailable: 0
  template:
    metadata:
      labels:
        k8s-app: metrics-server
    spec:
      containers:
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        # 修改部分
        - --kubelet-insecure-tls
        # 修改部分
        image: registry.cn-shanghai.aliyuncs.com/xuweiwei-kubernetes/metrics-server:v0.4.1
```
![[Pasted image 20240131204433.png]]

安装metrics-server：
```
kubectl apply -f components.yaml
```

  

## 4.4 准备Deployment和Service


创建nginx.yaml文件，内容如下：
```
apiVersion: apps/v1 # 版本号
kind: Deployment # 类型
metadata: # 元数据
  name: nginx # deployment的名称
  namespace: dev # 命名类型
spec: # 详细描述
  selector: # 选择器，通过它指定该控制器可以管理哪些Pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
  template: # 模块 当副本数据不足的时候，会根据下面的模板创建Pod副本
    metadata:
      labels:
        app: nginx-pod
    spec:
      containers:
        - name: nginx # 容器名称
          image: nginx:1.17.1 # 容器需要的镜像地址
          ports:
            - containerPort: 80 # 容器所监听的端口
          resources: # 资源限制
            requests:
              cpu: "100m" # 100m表示100millicpu，即0.1个CPU
```

创建Deployment：

```
kubectl create -f nginx.yaml
```

![[Pasted image 20240131204500.png]]


查看Deployment和Pod：

```
kubectl get pod,deploy -n dev
```
![[Pasted image 20240131204516.png]]
  
创建Service：
```
kubectl expose deployment nginx --name=nginx --type=NodePort --port=80 --target-port=80 -n dev
```
![[Pasted image 20240131204531.png]]

查看Service：
```
kubectl get svc -n dev
```
![[Pasted image 20240131204543.png]]
## 4.5 部署HPA

  创建pc-hpa.yaml文件，内容如下：

```
apiVersion: autoscaling/v1 # 版本号
kind: HorizontalPodAutoscaler # 类型
metadata: # 元数据
  name: pc-hpa # deployment的名称
  namespace: dev # 命名类型
spec:
  minReplicas: 1 # 最小Pod数量
  maxReplicas: 10 # 最大Pod数量
  targetCPUUtilizationPercentage: 3 # CPU使用率指标
  scaleTargetRef:  # 指定要控制的Nginx的信息
    apiVersion: apps/v1
    kind: Deployment
    name: nginx
```

  
创建hpa：
```
kubectl create -f pc-hpa.yaml
```
![[Pasted image 20240131204604.png]]

  
查看hpa：
```
kubectl get hpa -n dev
```
![[Pasted image 20240131204613.png]]
## 4.6 测试

使用压测工具如Jmeter对service的地址[http://192.168.18.100:30395](http://192.168.18.100:30395)进行压测，然后通过控制台查看hpa和pod的变化。
hpa的变化：
```
kubectl get hpa -n dev -w
```
![[Pasted image 20240131204628.png]]

Deployment的变化：

```
kubectl get deployment -n dev -w
```
![[Pasted image 20240131204636.png]] 

Pod的变化：
```
kubectl get pod -n dev -w
```
![[Pasted image 20240131204645.png]]

# 5 DaemonSet（DS）

## 5.1 概述

  

- DaemonSet类型的控制器可以保证集群中的每一台（或指定）节点上都运行一个副本，一般适用于日志收集、节点监控等场景。也就是说，如果一个Pod提供的功能是节点级别的（每个节点都需要且只需要一个），那么这类Pod就适合使用DaemonSet类型的控制器创建。
![[Pasted image 20240131204654.png]]
DaemonSet控制器的特点：
- 每向集群中添加一个节点的时候，指定的Pod副本也将添加到该节点上。
- 当节点从集群中移除的时候，Pod也会被垃圾回收。

DaemonSet的资源清单：
```
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型
metadata: # 元数据
  name: # 名称
  namespace: #命名空间
  labels: #标签
    controller: daemonset
spec: # 详情描述
  revisionHistoryLimit: 3 # 保留历史版本
  updateStrategy: # 更新策略
    type: RollingUpdate # 滚动更新策略
    rollingUpdate: # 滚动更新
      maxUnavailable: 1 # 最大不可用状态的Pod的最大值，可用为百分比，也可以为整数
  selector: # 选择器，通过它指定该控制器管理那些Pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
    matchExpressions: # Expressions匹配规则
      - key: app
        operator: In
        values:
          - nginx-pod
  template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
     metadata:
       labels:
         app: nginx-pod
     spec:
       containers:
         - name: nginx
           image: nginx:1.17.1
           ports:
             - containerPort: 80
```

  

## 5.2 创建DaemonSet

创建pc-daemonset.yaml文件，内容如下：

```
apiVersion: apps/v1 # 版本号
kind: DaemonSet # 类型
metadata: # 元数据
  name: pc-damonset # 名称
  namespace: dev #命名空间
spec: # 详情描述
  selector: # 选择器，通过它指定该控制器管理那些Pod
    matchLabels: # Labels匹配规则
      app: nginx-pod
  template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
     metadata:
       labels:
         app: nginx-pod
     spec:
       containers:
         - name: nginx
           image: nginx:1.17.1
           ports:
             - containerPort: 80
```

  

创建DaemonSet：
```
kubectl create -f pc-daemonset.yaml
```
![[Pasted image 20240131204718.png]]

## 5.3 查看DaemonSet

查看DaemonSet：
```
kubectl get ds -n dev -o wide
```
![[Pasted image 20240131204730.png]]
## 5.4 删除DaemonSet

  

删除DaemonSet：

```
kubectl delete ds pc-damonset -n dev
```
![[Pasted image 20240131204739.png]]

# 6 Job

  

## 6.1 概述

  Job主要用于负责批量处理短暂的一次性任务。Job的特点：
- 当Job创建的Pod执行成功结束时，Job将记录成功结束的Pod数量。
- 当成功结束的Pod达到指定的数量时，Job将完成执行。

Job可以保证指定数量的Pod执行完成。
![[Pasted image 20240131204744.png]]
Job的资源清单：
```
apiVersion: batch/v1 # 版本号
kind: Job # 类型
metadata: # 元数据
  name:  # 名称
  namespace:  #命名空间
  labels: # 标签
    controller: job
spec: # 详情描述
  completions: 1 # 指定Job需要成功运行Pod的总次数，默认为1
  parallelism: 1 # 指定Job在任一时刻应该并发运行Pod的数量，默认为1
  activeDeadlineSeconds: 30 # 指定Job可以运行的时间期限，超过时间还没结束，系统将会尝试进行终止
  backoffLimit: 6 # 指定Job失败后进行重试的次数，默认为6
  manualSelector: true # 是否可以使用selector选择器选择Pod，默认为false
  selector: # 选择器，通过它指定该控制器管理那些Pod
    matchLabels: # Labels匹配规则
      app: counter-pod
    matchExpressions: # Expressions匹配规则
      - key: app
        operator: In
        values:
          - counter-pod
  template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
     metadata:
       labels:
         app: counter-pod
     spec:
       restartPolicy: Never # 重启策略只能设置为Never或OnFailure
       containers:
         - name: counter
           image: busybox:1.30
           command: ["/bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 20;done"]
```

  

关于模板中的重启策略的说明：

- 如果设置为OnFailure，则Job会在Pod出现故障的时候重启容器，而不是创建Pod，failed次数不变。
- 如果设置为Never，则Job会在Pod出现故障的时候创建新的Pod，并且故障Pod不会消失，也不会重启，failed次数+1。
- 如果指定为Always的话，就意味着一直重启，意味着Pod任务会重复执行，这和Job的定义冲突，所以不能设置为Always。

  

## 6.2 创建Job

  

- 创建pc-job.yaml文件，内容如下：

  

```
apiVersion: batch/v1 # 版本号
kind: Job # 类型
metadata: # 元数据
  name: pc-job # 名称
  namespace: dev #命名空间
spec: # 详情描述
  manualSelector: true # 是否可以使用selector选择器选择Pod，默认为false
  selector: # 选择器，通过它指定该控制器管理那些Pod
    matchLabels: # Labels匹配规则
      app: counter-pod
  template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
    metadata:
      labels:
        app: counter-pod
    spec:
      restartPolicy: Never # 重启策略只能设置为Never或OnFailure
      containers:
        - name: counter
          image: busybox:1.30
          command: [ "/bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 3;done" ]
```

  

创建Job：
```
kubectl create -f pc-job.yaml
```

![[Pasted image 20240131204809.png]]

## 6.3 查看Job

查看Job：

```
kubectl get job -n dev -w
```
![[Pasted image 20240131204822.png]]

查看Pod：
```
kubectl get pod -n dev -w
```
![[Pasted image 20240131204834.png]]

## 6.4 删除Job

删除Job：

```
kubectl delete -f pc-job.yaml
```
![[Pasted image 20240131204843.png]]

# 7 CronJob（CJ）

  

## 7.1 概述

  

- CronJob控制器以Job控制器为其管控对象，并借助它管理Pod资源对象，Job控制器定义的作业任务在其控制器资源创建之后便会立即执行，但CronJob可以以类似Linux操作系统的周期性任务作业计划的方式控制器运行时间点及重复运行的方式，换言之，CronJob可以在特定的时间点反复去执行Job任务。
![[Pasted image 20240131204848.png]]

CronJob的资源清单：

```
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型
metadata: # 元数据
  name:  # 名称
  namespace:  #命名空间
  labels:
    controller: cronjob
spec: # 详情描述
  schedule: # cron格式的作业调度运行时间点，用于控制任务任务时间执行
  concurrencyPolicy: # 并发执行策略
  failedJobsHistoryLimit: # 为失败的任务执行保留的历史记录数，默认为1
  successfulJobsHistoryLimit: # 为成功的任务执行保留的历史记录数，默认为3
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象，下面其实就是job的定义
    metadata: {}
    spec:
      completions: 1 # 指定Job需要成功运行Pod的总次数，默认为1
      parallelism: 1 # 指定Job在任一时刻应该并发运行Pod的数量，默认为1
      activeDeadlineSeconds: 30 # 指定Job可以运行的时间期限，超过时间还没结束，系统将会尝试进行终止
      backoffLimit: 6 # 指定Job失败后进行重试的次数，默认为6
      template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
        spec:
          restartPolicy: Never # 重启策略只能设置为Never或OnFailure
          containers:
            - name: counter
              image: busybox:1.30
              command: [ "/bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 20;done" ]
```

  

schedule：cron表达式，用于指定任务的执行时间。

- */1  *  *  *  *：表示分钟  小时  日  月份  星期。
- 分钟的值从0到59。
- 小时的值从0到23。
- 日的值从1到31。
- 月的值从1到12。
- 星期的值从0到6，0表示星期日。
- 多个时间可以用逗号隔开，范围可以用连字符给出：* 可以作为通配符，/表示每...

concurrencyPolicy：并发执行策略

- Allow：运行Job并发运行（默认）。
- Forbid：禁止并发运行，如果上一次运行尚未完成，则跳过下一次运行。
- Replace：替换，取消当前正在运行的作业并使用新作业替换它。

  

## 7.2 创建CronJob

创建pc-cronjob.yaml文件，内容如下：

```
apiVersion: batch/v1beta1 # 版本号
kind: CronJob # 类型
metadata: # 元数据
  name: pc-cronjob # 名称
  namespace: dev  #命名空间
spec: # 详情描述
  schedule: "*/1 * * * * " # cron格式的作业调度运行时间点，用于控制任务任务时间执行
  jobTemplate: # job控制器模板，用于为cronjob控制器生成job对象，下面其实就是job的定义
    metadata: {}
    spec:
      template: # 模板，当副本数量不足时，会根据下面的模板创建Pod模板
        spec:
          restartPolicy: Never # 重启策略只能设置为Never或OnFailure
          containers:
            - name: counter
              image: busybox:1.30
              command: [ "/bin/sh","-c","for i in 9 8 7 6 5 4 3 2 1;do echo $i;sleep 2;done" ]
```

  

创建CronJob：

```
kubectl create -f pc-cronjob.yaml
```
![[Pasted image 20240131204908.png]]

## 7.3 查看CronJob

  

查看CronJob：

```
kubectl get cronjob -n dev -w
```
![[Pasted image 20240131204917.png]]

  

查看Job：
```
kubectl get job -n dev -w
```
![[Pasted image 20240131204928.png]]

查看Pod：
```
kubectl get pod -n dev -w
```
![[Pasted image 20240131204934.png]]
## 7.4 删除CronJob
删除CronJob：

```
kubectl delete -f pc-cronjob.yaml
```
![[Pasted image 20240131204947.png]]
# 8 StatefulSet（有状态）

## 8.1 概述

- 无状态应用：

- 认为Pod都是一样的。
- 没有顺序要求。
- 不用考虑在哪个Node节点上运行。
- 随意进行伸缩和扩展。

- 有状态应用：

- 有顺序的要求。
- 认为每个Pod都是不一样的。
- 需要考虑在哪个Node节点上运行。
- 需要按照顺序进行伸缩和扩展。
- 让每个Pod都是独立的，保持Pod启动顺序和唯一性。

- StatefulSet是Kubernetes提供的管理有状态应用的负载管理控制器。
- StatefulSet部署需要HeadLinessService（无头服务）。

为什么需要HeadLinessService（无头服务）？

- 在用Deployment时，每一个Pod名称是没有顺序的，是随机字符串，因此是Pod名称是无序的，但是在StatefulSet中要求必须是有序 ，每一个Pod不能被随意取代，Pod重建后pod名称还是一样的。
- 而Pod IP是变化的，所以是以Pod名称来识别。Pod名称是Pod唯一性的标识符，必须持久稳定有效。这时候要用到无头服务，它可以给每个Pod一个唯一的名称 。

- StatefulSet常用来部署RabbitMQ集群、Zookeeper集群、MySQL集群、Eureka集群等。

## 8.2 创建StatefulSet

- 创建pc-stateful.yaml文件，内容如下：

```
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  ports:
    - port: 80 # Service的端口
      targetPort: 80 # Pod的端口
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pc-statefulset
  namespace: dev
spec:
  replicas: 3
  serviceName: service-headliness
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
```

- 创建StatefulSet：

```
kubectl create -f pc-stateful.yaml
```
![[Pasted image 20240131204957.png]]
## 8.3 查看StatefulSet

查看StatefulSet：
```
kubectl get statefulset pc-statefulset -n dev -o wide
```

![[Pasted image 20240131205016.png]]

查看Pod：
```
kubectl get pod -n dev -o wide
```
![[Pasted image 20240131205027.png]]
## 8.4 删除StatefulSet

删除StatefulSet：
```
kubectl delete -f pc-stateful.yaml
```
![[Pasted image 20240131205036.png]]
## 8.5 Deployment和StatefulSet的区别

- Deployment和StatefulSet的区别：Deployment没有唯一标识而StatefulSet有唯一标识。
- StatefulSet的唯一标识是根据主机名+一定规则生成的。
- StatefulSet的唯一标识是`主机名.无头Service名称.命名空间.svc.cluster.local`。

  

## 8.6 StatefulSet的金丝雀发布

StatefulSet支持两种更新策略：OnDelete和RollingUpdate（默认），其中OnDelete表示删除之后才更新，RollingUpdate表示滚动更新。
```
updateStrategy:
  rollingUpdate: # 如果更新的策略是OnDelete，那么rollingUpdate就失效
    partition: 2 # 表示从第2个分区开始更新，默认是0
  type: RollingUpdate /OnDelete # 滚动更新
```

  

示例：pc-statefulset.yaml
```
apiVersion: v1
kind: Service
metadata:
  name: service-headliness
  namespace: dev
spec:
  selector:
    app: nginx-pod
  clusterIP: None # 将clusterIP设置为None，即可创建headliness Service
  type: ClusterIP
  ports:
    - port: 80 # Service的端口
      targetPort: 80 # Pod的端口
---

apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: pc-statefulset
  namespace: dev
spec:
  replicas: 3
  serviceName: service-headliness
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
			
  updateStrategy:
    rollingUpdate:
      partition: 0
    type: RollingUpdate  			
```