查看本地镜像和检索拉取Zookeeper 镜像
```shell
# 查看本地镜像
docker images
# 检索ZooKeeper 镜像
docker search zookeeper
# 我使用的版本
docker pull zookeeper:3.7.0
```
创建ZooKeeper 挂载目录（数据挂载目录、配置挂载目录和日志挂载目录）
```shell
mkdir -p /root/datamapping/zookeeper/data # 数据挂载目录
mkdir -p /root/datamapping/zookeeper/conf # 配置挂载目录
mkdir -p /root/datamapping/zookeeper/logs # 日志挂载目录
```
启动ZooKeeper容器
```shell
docker run -d --name zookeeper --privileged=true -p 2181:2181  -v /root/datamapping/zookeeper/data:/data -v /root/datamapping/zookeeper/conf:/conf -v /root/datamapping/zookeeper/logs:/datalog zookeeper:3.7.0
```
-e TZ="Asia/Shanghai" # 指定上海时区 
-d # 表示在一直在后台运行容器
-p 2181:2181 # 对端口进行映射，将本地2181端口映射到容器内部的2181端口
--name # 设置创建的容器名称
-v # 将本地目录(文件)挂载到容器指定目录；
--restart always # 始终重新启动zookeeper，也可以创建成功后再设置自动启动
```shell
docker update zookeeper --restart=always
```
接下来，添加ZooKeeper配置文件，在挂载配置文件目录(/root/datamapping/zookeeper/conf)下，新增zoo.cfg 配置文件
```
vi /root/datamapping/zookeeper/conf/zoo.cfg
```
配置内容如下：
```shell
# 保存zookeeper中的数据
dataDir=/data  
# 客户端连接端口，通常不做修改
clientPort=2181 
dataLogDir=/datalog
# 通信心跳时间
tickTime=2000  
# LF(leader - follower)初始通信时限
initLimit=5    
# LF 同步通信时限
syncLimit=2    
autopurge.snapRetainCount=3
autopurge.purgeInterval=0
maxClientCnxns=60
standaloneEnabled=true
admin.enableServer=true
server.1=localhost:2888:3888;2181

```
重启zookeeper，进入容器内部，验证容器状态
```shell
docker restart zookeeper
# 进入zookeeper 容器内部
docker exec -it zookeeper /bin/bash
# 检查容器状态
docker exec -it zookeeper /bin/bash ./bin/zkServer.sh status
# 进入控制台
docker exec -it zookeeper zkCli.sh

```

