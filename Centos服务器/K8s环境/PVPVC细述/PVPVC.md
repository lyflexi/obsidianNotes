在前面已经提到，容器的生命周期可能很短，会被频繁的创建和销毁。那么容器在销毁的时候，保存在容器中的数据也会被清除。这种结果对用户来说，在某些情况下是不乐意看到的。为了持久化保存容器中的数据，kubernetes引入了Volume的概念。
# 1 概述
Volume是Pod中能够被多个容器访问的共享目录，它**被定义在Pod上**，然后被一个Pod里面的多个容器挂载到具体的文件目录下，kubernetes通过Volume实现**同一个Pod中不同容器之间的数据共享以及数据的持久化存储**。Volume的生命周期不和Pod中的单个容器的生命周期有关，当容器终止或者重启的时候，Volume中的数据也不会丢失。

kubernetes的Volume支持多种类型，比较常见的有下面的几个：
- 简单存储：EmptyDir、HostPath、NFS。
- 高级存储：PV、PVC。
- 配置存储：ConfigMap、Secret。


# EmptyDir

EmptyDir是最基础的Volume类型，一个EmptyDir就是Host上的一个空目录。

EmptyDir是在Pod被分配到Node时创建的，它的初始内容为空，并且**无须指定宿主机上对应的目录文件**，因为kubernetes会自动分配一个目录，**当Pod销毁时，EmptyDir中的数据也会被永久删除。**

EmptyDir的用途如下：
- 临时空间，例如用于某些应用程序运行时所需的临时目录，且无须永久保留。
- 一个容器需要从另一个容器中获取数据的目录（多容器共享目录）。

接下来，通过一个容器之间的共享案例来使用描述一个EmptyDir。

在一个Pod中准备两个容器nginx和busybox，然后声明一个volume分别挂载到两个容器的目录中，然后nginx容器负责向volume中写日志，busybox中通过命令将日志内容读到控制台。
![[Pasted image 20240131211728.png]]
创建Pod，创建volume-emptydir.yaml文件，内容如下：
```
apiVersion: v1
kind: Pod
metadata:
  name: volume-emptydir
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.17.1
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
      volumeMounts: # 将logs-volume挂载到nginx容器中对应的目录，该目录为/var/log/nginx
        - name: logs-volume
          mountPath: /var/log/nginx
    - name: busybox
      image: busybox:1.30
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件
      volumeMounts: # 将logs-volume挂载到busybox容器中的对应目录，该目录为/logs
        - name: logs-volume
          mountPath: /logs
  volumes: # 声明volume，name为logs-volume，类型为emptyDir
    - name: logs-volume
      emptyDir: {}
```

将名字为logs-volume的存储卷，分别挂载到ngnix和busybox下，并且设置挂载目录：
![[Pasted image 20240131212000.png]]

知识点：访问ngnix会生成两个文件，一个是access.log，一个是error.log

创建Pod：

```
kubectl create -f volume-emptydir.yaml
```
![[Pasted image 20240131212009.png]]

查看Pod：
```
kubectl get pod volume-emptydir -n dev -o wide
```
![[Pasted image 20240131212040.png]]

访问Pod中的Nginx：

```
curl 10.244.2.2
```
![[Pasted image 20240131212057.png]]


查看指定容器的标准输出：

```
kubectl logs -f volume-emptydir -n dev -c busybox
```
![[Pasted image 20240131212118.png]]
# HostPath

我们已经知道EmptyDir中的数据不会被持久化，它会随着Pod的结束而销毁，如果想要简单的将数据持久化到主机中，可以选择HostPath。

HostPath就是将**Node主机中的一个实际目录挂载到Pod中**，以供容器使用，这样的设计就可以保证Pod销毁了，但是数据依旧可以保存在Node主机上。
![[Pasted image 20240131212127.png]]
创建Pod，创建volume-hostpath.yaml文件，内容如下：
```
apiVersion: v1
kind: Pod
metadata:
  name: volume-hostpath
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.17.1
      imagePullPolicy: IfNotPresent
      ports:
        - containerPort: 80
      volumeMounts: # 将logs-volume挂载到nginx容器中对应的目录，该目录为/var/log/nginx
        - name: logs-volume
          mountPath: /var/log/nginx
    - name: busybox
      image: busybox:1.30
      imagePullPolicy: IfNotPresent
      command: ["/bin/sh","-c","tail -f /logs/access.log"] # 初始命令，动态读取指定文件
      volumeMounts: # 将logs-volume挂载到busybox容器中的对应目录，该目录为/logs
        - name: logs-volume
          mountPath: /logs
  volumes: # 声明volume，name为logs-volume，类型为hostPath
    - name: logs-volume
      hostPath:
        path: /root/logs
        type: DirectoryOrCreate # 目录存在就使用，不存在就先创建再使用
```

  

type的值的说明：

- DirectoryOrCreate：目录存在就使用，不存在就先创建后使用。
- Directory：目录必须存在。
- FileOrCreate：文件存在就使用，不存在就先创建后使用。
- File：文件必须存在。
- Socket：unix套接字必须存在。
- CharDevice：字符设备必须存在。
- BlockDevice：块设备必须存在。

创建Pod：
```
kubectl create -f volume-hostpath.yaml
```
![[Pasted image 20240131212147.png]]

查看Pod：
```
kubectl get pod volume-hostpath -n dev -o wide
```
![[Pasted image 20240131212200.png]]


访问Pod中的Nginx：

```
curl 10.244.2.3
```
![[Pasted image 20240131212211.png]]

需要到Pod所在的节点（k8s-node2）查看hostPath映射的目录中的文件：

```
ls /root/logs
```
![[Pasted image 20240131212229.png]]

同样的道理，如果在此目录中创建文件，到容器中也是可以看到的。

 
# PV和PVC

前面我们已经学习了使用NFS提供存储，此时就要求用户会搭建NFS系统，并且会在yaml配置nfs。由于kubernetes支持的存储系统有很多，要求客户全部掌握，显然不现实。为了能够屏蔽底层存储实现的细节，方便用户使用，kubernetes引入了PV和PVC两种资源对象。
- PV（Persistent Volume）是持久化卷的意思，是对底层的共享存储的一种抽象。一般情况下**PV由kubernetes管理员**进行创建和配置，它和底层具体的共享存储技术有关，并通过插件完成和共享存储的对接。
- PVC（Persistent Volume Claim）是持久化卷声明的意思，是用户对于存储需求的一种声明。换言之，PVC其实就是用户向kubernetes系统发出的一种资源需求申请。
![[Pasted image 20240131212309.png]]
  使用了PV和PVC之后，工作可以得到进一步的提升：
- 存储：存储工程师维护。
- PV：kubernetes管理员维护。
- PVC：kubernetes用户维护。
## 准备工作（准备NFS环境）

创建目录：
```
mkdir -pv /root/data/{pv1,pv2,pv3}
```
授权：
```
chmod 777 -R /root/data
```
修改/etc/exports文件：，将3个NFS路径暴露出去
```
vim /etc/exports

/root/data/pv1     192.168.18.0/24(rw,no_root_squash) 
/root/data/pv2     192.168.18.0/24(rw,no_root_squash) 
/root/data/pv3     192.168.18.0/24(rw,no_root_squash)
```


重启nfs服务：
```
systemctl restart nfs
```

  

## PV

PV的资源清单文件

pv是集群级别的资源，跨namespace的，所以不用配置namespace


PV是存储资源的抽象，下面是PV的资源清单文件：


```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型，和底层正则的存储对应
    path:
    server:
  capacity: # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes: # 访问模式
    -
  storageClassName: # 存储类别
  persistentVolumeReclaimPolicy: # 回收策略
```
pv的关键配置参数说明：

存储类型：底层实际存储的类型，kubernetes支持多种存储类型，每种存储类型的配置有所不同。
存储能力（capacity）：目前只支持存储空间的设置（storage=1Gi），不过未来可能会加入IOPS、吞吐量等指标的配置。
访问模式（accessModes）：用来描述用户应用对存储资源的访问权限，访问权限包括下面几种方式：
- ReadWriteOnce（RWO）：读写权限，但是只能被单个节点挂载。
- ReadOnlyMany（ROX）：只读权限，可以被多个节点挂载。
- ReadWriteMany（RWX）：读写权限，可以被多个节点挂载。
回收策略（ persistentVolumeReclaimPolicy）：当PV不再被使用之后，对其的处理方式，目前支持三种策略：
- Retain（保留）：保留数据，需要管理员手动清理数据。
- Recycle（回收）：清除PV中的数据，效果相当于`rm -rf /volume/*`。**pvc释放后变为unbound状态，可以继续被其他pvc绑定**
- Delete（删除）：由 与PV相连的后端存储完成volume的删除操作，常见于云服务器厂商的存储服务。

存储类别（storageClassName）：PV可以通过storageClassName参数指定一个存储类别。
- 具有特定类型的PV只能和请求了该类别的PVC进行绑定。
- 未设定类别的PV只能和不请求任何类别的PVC进行绑定。

状态（status）：一个PV的生命周期，可能会处于4种不同的阶段。
- Available（可用）：表示可用状态，还未被任何PVC绑定。
- Bound（已绑定）：表示PV已经被PVC绑定。
- Released（已释放）：表示PVC被删除(PVC与PV解绑了)，但是PV资源还没有被集群重新释放。
- Failed（失败）：表示该PV的自动回收失败。

  
创建pv.yaml文件，内容如下：
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv1
spec:
  nfs: # 存储类型吗，和底层正则的存储对应
    path: /root/data/pv1
    server: 192.168.18.100
  capacity: # 存储能力，目前只支持存储空间的设置
    storage: 1Gi
  accessModes: # 访问模式
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain # 回收策略

---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv2
spec:
  nfs: # 存储类型吗，和底层正则的存储对应
    path: /root/data/pv2
    server: 192.168.18.100
  capacity: # 存储能力，目前只支持存储空间的设置
    storage: 2Gi
  accessModes: # 访问模式
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain # 回收策略
  
---
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv3
spec:
  nfs: # 存储类型吗，和底层正则的存储对应
    path: /root/data/pv3
    server: 192.168.18.100
  capacity: # 存储能力，目前只支持存储空间的设置
    storage: 3Gi
  accessModes: # 访问模式
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain # 回收策略
```

  

创建PV
```
kubectl create -f pv.yaml
```
![[Pasted image 20240201095651.png]]
查看PV：
```
kubectl get pv -o wide
```
![[Pasted image 20240201095656.png]]
CLAIM是指明，该PV被哪个PVC所绑定

## PVC

但PVC是用户空间,是有namespace的

PVC是资源的申请，用来声明对存储空间、访问模式、存储类别需求信息，下面是PVC的资源清单文件：


```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc
  namespace: dev
spec:
  accessModes: # 访客模式
    - 
  selector: # 采用标签对PV选择
  storageClassName: # 存储类别
  resources: # 请求空间
    requests:
      storage: 5Gi
```

  

PVC的关键配置参数说明：

访客模式（accessModes）：用于描述用户应用对存储资源的访问权限。
- 选择条件（selector）：通过Label Selector的设置，可使PVC对于系统中已存在的PV进行筛选。
- 存储类别（storageClassName）：PVC在定义时可以设定需要的后端存储的类别，只有设置了该class的pv才能被系统选出。
- 资源请求（resources）：描述对存储资源的请求。

  
创建pvc.yaml文件，内容如下：

PVC的访客模式accessModes一定要和PV的访客模式accessModes一致，否则绑定不上
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc1
  namespace: dev
spec:
  accessModes: # 访客模式
    - ReadWriteMany
  resources: # 请求空间
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc2
  namespace: dev
spec:
  accessModes: # 访客模式
    - ReadWriteMany
  resources: # 请求空间
    requests:
      storage: 1Gi

---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc3
  namespace: dev
spec:
  accessModes: # 访客模式
    - ReadWriteMany
  resources: # 请求空间
    requests:
      storage: 5Gi
```
### 创建PVC
创建PVC：

```
kubectl create -f pvc.yaml
```
![[Pasted image 20240201095753.png]]

  

查看PVC：
```
kubectl get pvc -n dev -o wide
```
![[Pasted image 20240201095807.png]]
  

查看PV：
```
kubectl get pv -o wide
```
![[Pasted image 20240201095822.png]]
### 创建Pod使用PVC
创建Pod使用PVC，创建pvc-pod.yaml文件，内容如下：
```
apiVersion: v1
kind: Pod
metadata:
  name: pod1
  namespace: dev
spec:
  containers:
  - name: busybox
    image: busybox:1.30
    command: ["/bin/sh","-c","while true;do echo pod1 >> /root/out.txt; sleep 10; done;"]
    volumeMounts:
    - name: volume
      mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc1
        readOnly: false

---
apiVersion: v1
kind: Pod
metadata:
  name: pod2
  namespace: dev
spec:
  containers:
    - name: busybox
      image: busybox:1.30
      command: ["/bin/sh","-c","while true;do echo pod1 >> /root/out.txt; sleep 10; done;"]
      volumeMounts:
        - name: volume
          mountPath: /root/
  volumes:
    - name: volume
      persistentVolumeClaim:
        claimName: pvc2
        readOnly: false
```

  

创建Pod：
```
kubectl create -f pvc-pod.yaml
```
![[Pasted image 20240201095855.png]]
  

部署pod之后查看Pod

```
kubectl get pod -n dev -o wide
```
![[Pasted image 20240201095950.png]]

部署pod之后查看PVC
```
kubectl get pvc -n dev -o wide
```
![[Pasted image 20240201100003.png]]

部署pod之后查看PV
```
kubectl get pv -n dev -o wide
```

![[Pasted image 20240201100008.png]]

查看nfs中的文件存储：

```
ls /root/data/pv1/out.txt
ls /root/data/pv2/out.txt
```

![[Pasted image 20240201100028.png]]

## 生命周期
![[Pasted image 20240201100056.png]]
  

PVC和PV是一一对应的，PV和PVC之间的相互作用遵循如下的生命周期。
- 资源供应：管理员手动创建底层存储和PV。
- 资源绑定：用户创建PVC，kubernetes负责根据PVC声明去寻找PV，并绑定在用户定义好PVC之后，系统将根据PVC对存储资源的请求在以存在的PV中选择一个满足条件的。
	- 一旦找到，就将该PV和用户定义的PVC进行绑定，用户的应用就可以使用这个PVC了。
	- 如果找不到，PVC就会无限期的处于Pending状态，直到系统管理员创建一个符合其要求的PV。
	- PV一旦绑定到某个PVC上，就会被这个PVC独占，不能再和其他的PVC进行绑定了。
- 资源使用：用户可以在Pod中像volume一样使用PVC，Pod使用Volume的定义，将PVC挂载到容器内的某个路径进行使用。
- 资源释放：当存储资源使用完毕后，用户可以删除PVC，和该PVC绑定的PV将会标记为“已释放”，但是还不能立刻和其他的PVC进行绑定。**通过之前PVC写入的数据可能还留在存储设备上，只有在清除之后该PV才能再次使用。**
- 资源回收：管理员根据PV设置的回收策略进行资源的回收，用于设置与之绑定的PVC释放资源之后如何处理遗留数据的问题。只有PV的存储空间完成回收，才能供新的PVC绑定和使用。
## PV 和 PVC 的绑定规则细述
  
创建PVC后一直绑定不了PV的可能原因

- **①PVC的空间申请大小比PV的空间要大。**
- **②PVC的storageClassName和PV的storageClassName不一致。**
- **③PVC的accessModes和PV的accessModes不一致。**
- **④VolumeMode：主要定义 volume 是文件系统（FileSystem）类型还是块（Block）类型，PV 与 PVC 的 VolumeMode 标签必须相匹配。**

**PV 状态介绍**

|   |   |
|---|---|
|PV 状态|描述|
|Avaliable|创建好的 PV 在没有和 PVC 绑定的时候处于 Available 状态。|
|Bound|当一个 PVC 与 PV 绑定之后，PVC 就会进入 Bound 的状态。|
|Released|一个回收策略为 Retain 的 PV，当其绑定的 PVC 被删除，该 PV 会由 Bound 状态转变为 Released 状态。  <br>**注意：**Released 状态的 PV 需要手动删除 YAML 配置文件中的 claimRef 字段才能与 PVC 成功绑定。|
![[Pasted image 20240201100359.png]]

**PVC 状态介绍**：

|   |   |
|---|---|
|PVC 状态|描述|
|Pending|没有满足条件的 PV 能与 PVC 绑定时，PVC 将处于 Pending 状态。|
|Bound|当一个 PV 与 PVC 绑定之后，PVC 会进入 Bound 的状态。|

**绑定规则**

当 PVC 绑定 PV 时，需考虑以下参数来筛选当前集群内是否存在满足条件的 PV。

|   |   |
|---|---|
|参数|描述|
|VolumeMode|主要定义 volume 是文件系统（FileSystem）类型还是块（Block）类型，PV 与 PVC 的 VolumeMode 标签必须相匹配。|
|Storageclass|PV 与 PVC 的 storageclass 类名必须相同（或同时为空）。|
|AccessMode|主要定义 volume 的访问模式，PV 与 PVC 的 AccessMode 必须相同。|
|Size|主要定义 volume 的存储容量，PVC 中声明的容量必须小于等于 PV，如果存在多个满足条件的 PV，则选择最小的 PV 与 PVC 绑定。|

说明：

PVC 创建后，系统会根据上述参数筛选满足条件的 PV 进行绑定。如果当前集群内的 PV 资源不足，系统会动态创建一个满足绑定条件的 PV 与 PVC 进行绑定。

**StorageClass 的选择和 PV/PVC 的绑定关系**

容器服务 TKE 的平台操作中，StorageClass 的选择与 PV/PVC 之间的绑定关系见下图：
![[Pasted image 20240201100511.png]]
  