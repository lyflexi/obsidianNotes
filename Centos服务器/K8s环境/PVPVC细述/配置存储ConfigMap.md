
ConfigMap是一个比较特殊的存储卷，它的主要作用是用来存储配置信息的。

  

# ConfigMap

ConfigMap的资源清单文件：
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: configMap
  namespace: dev
data: # <map[string]string>
  xxx
```

## ConfigMap基础
创建configmap.yaml文件，内容如下：
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: configmap
  namespace: dev
data:
  info: #存储文件名叫info,即key
    username:admin
    password:123456
```

创建ConfigMap：
```
kubectl create -f configmap.yaml
```
![[Pasted image 20240201100703.png]]

创建Pod，创建pod-configmap.yaml文件，内容如下：
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-configmap
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.17.1
      volumeMounts: #将存储文件名info 挂载到该pod容器的目录/configmap/config
        - mountPath: /configmap/config
          name: config
  volumes:
    - name: config
      configMap:
        name: configmap
```

创建Pod：
```
kubectl create -f pod-configmap.yaml
```
![[Pasted image 20240201100723.png]]

查看pod
```
kubectl get pod pod-configmap -n dev
```
![[Pasted image 20240201100740.png]]


进入容器，查看配置：

```
kubectl exec -it pod-configmap -n dev /bin/sh
cd /configmap/config
ls
more info
```

![[ConfigMap之进入容器查看.gif]]

ConfigMap中的key映射为一个文件，value映射为文件中的内容。如果更新了ConfigMap中的内容，容器中的值也会动态更新。

## ConfigMap高级
在ConfigMap基础中，我们已经可以实现创建ConfigMap了，但是如果实际工作中这样使用，就会显得很繁琐。

注意事项：
- ConfigMap 在设计上不是用来保存大量数据的。在 ConfigMap 中保存的数据不可超过 1 MiB。
- 如果需要保存超出此尺寸限制的数据，需要考虑挂载存储卷或者使用独立的数据库或者文件服务。
语法：
```
kubectl create configmap <map-name> <data-source>
```

  

### 从一个目录中创建ConfigMap

示例：
```
mkdir -pv configure-pod-container/configmap/
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
wget https://kubernetes.io/examples/configmap/ui.properties -O configure-pod-container/configmap/ui.properties
kubectl create configmap cm1 --from-file=configure-pod-container/configmap/
kubectl get cm cm1 -o yaml
```
![[Pasted image 20240201101038.png]]

### 从一个文件中创建ConfigMap

示例：

```
mkdir -pv configure-pod-container/configmap/
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
# 默认情况下的key的名称是文件的名称
kubectl create configmap cm2 --from-file=configure-pod-container/configmap/game.properties
```
![[Pasted image 20240201101102.png]]

### 从一个文件中创建ConfigMap，并自定义ConfigMap中key的名称

示例：
```
mkdir -pv configure-pod-container/configmap/
wget https://kubernetes.io/examples/configmap/game.properties -O configure-pod-container/configmap/game.properties
kubectl create configmap cm3 --from-file=cm3=configure-pod-container/configmap/game.properties
```
![[Pasted image 20240201101135.png]]
### 从环境变量文件创建ConfigMap

示例：

```
vim configure-pod-container/configmap/env-file.properties
```
语法规则:
```
#   env 文件中的每一行必须为 VAR = VAL 格式。
#   以＃开头的行(即注释)将被忽略。
#   空行将被忽略。
#   引号没有特殊处理(即它们将成为 ConfigMap 值的一部分)
enemies=aliens
lives=3
allowed="true"
```
创建
```
kubectl create cm cm4 --from-env-file=configure-pod-container/configmap/env-file.properties
```
![[Pasted image 20240201101214.png]]

注意：当`--from-env-file`从多个数据源创建ConfigMap的时候，仅仅最后一个env文件有效。

  

  

### 在命令行根据键值对创建ConfigMap

示例：

```
kubectl create configmap cm5 --from-literal=special.how=very --from-literal=special.type=charm
```
![[Pasted image 20240201101227.png]]
### 4.3.7 使用ConfigMap定义容器环境变量

- 示例：

```
kubectl create configmap cm6 --from-literal=special.how=very --from-literal=special.type=charm
vim test-pod.yaml
```
test-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      env:
        # 定义环境变量
        - name: SPECIAL_LEVEL_KEY
          valueFrom:
            configMapKeyRef:
              # ConfigMap的名称
              name: cm6
              # ConfigMap的key
              key: special.how
  restartPolicy: Never
```
创建test-pod.yaml
```
kubectl apply -f test-pod.yaml
```
![[Pasted image 20240201101236.png]]
### 4.3.8 将 ConfigMap 中的所有键值对配置为容器环境变量

示例：

```
kubectl create configmap cm7 --from-literal=special.how=very --from-literal=special.type=charm
vim test-pod.yaml
```
test-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "env" ]
      envFrom:
      - configMapRef:
          name: cm7
  restartPolicy: Never
```
创建test-pod.yaml
```
kubectl apply -f test-pod.yaml
```
![[Pasted image 20240201101316.png]]

### 使用存储在 ConfigMap 中的数据填充容器

示例：

```
kubectl create configmap cm8 --from-literal=special.how=very --from-literal=special.type=charm
vim test-pod.yaml
```
test-pod.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: test-pod
spec:
  containers:
    - name: test-container
      image: busybox
      command: [ "/bin/sh", "-c", "ls /etc/config/" ]
      volumeMounts:
      - name: config-volume
        mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        # configMap的名称
        name: cm8
  restartPolicy: Never
```
创建test-pod.yaml
```
kubectl apply -f test-pod.yaml
```
![[Pasted image 20240201101344.png]]
# Secret

在kubernetes中，还存在一种和ConfigMap非常类似的对象，称为Secret对象，它主要用来存储敏感信息，例如密码、密钥、证书等等。

准备数据，使用base64对数据进行编码：

```
# 准备username
echo -n "admin" | base64
```
![[Pasted image 20240201101434.png]]
  

```
echo -n "123456" | base64
```

![[Pasted image 20240201101445.png]]
创建Secret，创建secret.yaml文件，内容如下：

```
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: dev
type: Opaque
data:
  username: YWRtaW4=
  password: MTIzNDU2
```
创建Secret：
```
kubectl create -f secret.yaml
```

![[Pasted image 20240201101525.png]]

上面的方式是先手动将数据进行编码，其实也可以使用直接编写数据，将数据编码交给kubernetes。

```
apiVersion: v1
kind: Secret
metadata:
  name: secret
  namespace: dev
type: Opaque
stringData:
  username: admin
  password: 123456
```

如果同时使用data和stringData，那么data会被忽略。


查看Secret详情：
```
kubectl describe secret secret -n dev
```
![[Pasted image 20240201101551.png]]


创建Pod，创建pod-secret.yaml文件，内容如下：
```
apiVersion: v1
kind: Pod
metadata:
  name: pod-secret
  namespace: dev
spec:
  containers:
    - name: nginx
      image: nginx:1.17.1
      volumeMounts:
        - mountPath: /secret/config
          name: config
  volumes:
    - name: config
      secret:
        secretName: secret
```
创建Pod：
```
kubectl create -f pod-secret.yaml
```
![[Pasted image 20240201101611.png]]
查看Pod
```
kubectl get pod pod-secret -n dev
```
![[Pasted image 20240201101622.png]]

进入容器，查看secret信息，发现已经自动解码了：
```
kubectl exec -it pod-secret -n dev /bin/sh
ls /secret/config
more /secret/config/username
more /secret/config/password
```
![[Secret之查看容器.gif]]

  

Secret的实际用途
- imagePullSecret：Pod拉取私有镜像仓库的时使用的账户密码，会传递给kubelet，然后kubelet就可以拉取有密码的仓库里面的镜像。
- 创建一个ImagePullSecret：

```
kubectl create secret docker-registry docker-harbor-registrykey --docker-server=192.168.18.119:85 \
          --docker-username=admin --docker-password=Harbor12345 \
          --docker-email=1900919313@qq.com
```

查看是否创建成功：
```
kubectl get secret docker-harbor-registrykey
```

新建redis.yaml文件，内容如下：
```
apiVersion: v1
kind: Pod
metadata:
  name: redis
spec:
  containers:
    - name: redis
      image: 192.168.18.119:85/yuncloud/redis # 这是Harbor的镜像私有仓库地址
  imagePullSecrets:
    - name: docker-harbor-registrykey
```

创建Pod：
```
kubectl apply -f redis.yaml
```

  

# ConfigMap&&Secret使用SubPath解决目录覆盖问题

ConfigMap和Secret在进行目录挂载的时候会覆盖目录，我们可以使用SubPath解决这个问题。

示例：

```
# 创建一个Pod
kubectl run nginx --image=nginx:1.17.1
# 将nginx.conf导出到本地
kubectl exec -it nginx -- cat /etc/nginx/nginx.conf > nginx.conf
# 创建ConfigMap
kubectl create cm nginx-conf --from-file=nginx.conf
```

```
kubectl delete pod nginx
vim nginx.yaml
```
nginx.yaml
```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.17.1
      command: [ "/bin/sh", "-c", "sleep 3600" ]
      volumeMounts:
      - name: nginx-conf
        mountPath: /etc/nginx
  volumes:
    - name: nginx-conf
      configMap:
        # configMap的名称
        name: nginx-conf
  restartPolicy: Never
```

```
kubectl apply -f nginx.yaml
```

```
kubectl exec -it nginx -- ls /etc/nginx
```
![[Pasted image 20240201101854.png]]
```
vim nginx.yaml
```

```
apiVersion: v1
kind: Pod
metadata:
  name: nginx
spec:
  containers:
    - name: nginx
      image: nginx:1.17.1
      command: [ "/bin/sh", "-c", "sleep 3600" ]
      volumeMounts:
      - name: nginx-conf
        mountPath: /etc/nginx/nginx.conf
        subPath: nginx.conf # subPath：要覆盖文件的相对路径
  volumes:
    - name: nginx-conf
      configMap:
        # configMap的名称
        name: nginx-conf
        items:
         - key: nginx.conf # key：ConfigMap中的key的名称
           path: nginx.conf # 此处的path相当于 mv nginx.conf nginx.conf
  restartPolicy: Never
```

```
kubectl apply -f nginx.yaml
```

```
kubectl exec -it nginx -- ls /etc/nginx
```
![[Pasted image 20240201101909.png]]
# ConfigMap&&Secret的热更新

- 注意事项：
- ①如果ConfigMap和Secret是以subPath的形式挂载的，那么Pod是不会感知到ConfigMap和Secret的更新的。
- ②如果Pod的变量来自ConfigMap和Secret中定义的内容，那么ConfigMap和Secret更新后，也不会更新Pod中的变量。