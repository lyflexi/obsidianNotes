# 镜像版本选择
```shell
docker pull apache/rocketmq:4.9.2
```

# 为容器网络互联创建RocketMQ的docker网络
```shell
# 后续的name-server,broker,rocketmq-console都会使用该网络
docker network create rocketmq

# 创建好网络可以使用docker inspect命令查看网络信息
docker inspect rocketmq
```
# 部署name-server，修改runserver.sh

创建name-server的sh文件runserver.sh，否则文件挂载的时候就会把runserver.sh当作目录对待
```shell
mkdir -p /root/datamapping/rocketmq/namesrv/bin/
vi /root/datamapping/rocketmq/namesrv/bin/runserver.sh
```
打开提前修改生效，否则文件挂载的时候就会把runbroker.sh当作空内容看待以至于把容器内的runbroker.sh给冲掉，将71行和76行的Xms和Xmx等改小一点
```shell
#!/bin/sh

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
# The RAMDisk initializing size in MB on Darwin OS for gc-log
DIR_SIZE_IN_MB=600

choose_gc_log_directory()
{
    case "`uname`" in
        Darwin)
            if [ ! -d "/Volumes/RAMDisk" ]; then
                # create ram disk on Darwin systems as gc-log directory
                DEV=`hdiutil attach -nomount ram://$((2 * 1024 * DIR_SIZE_IN_MB))` > /dev/null
                diskutil eraseVolume HFS+ RAMDisk ${DEV} > /dev/null
                echo "Create RAMDisk /Volumes/RAMDisk for gc logging on Darwin OS."
            fi
            GC_LOG_DIR="/Volumes/RAMDisk"
        ;;
        *)
            # check if /dev/shm exists on other systems
            if [ -d "/dev/shm" ]; then
                GC_LOG_DIR="/dev/shm"
            else
                GC_LOG_DIR=${BASE_DIR}
            fi
        ;;
    esac
}

choose_gc_options()
{
    # Example of JAVA_MAJOR_VERSION value : '1', '9', '10', '11', ...
    # '1' means releases befor Java 9
    JAVA_MAJOR_VERSION=$("$JAVA" -version 2>&1 | sed -r -n 's/.* version "([0-9]*).*$/\1/p')
    if [ -z "$JAVA_MAJOR_VERSION" ] || [ "$JAVA_MAJOR_VERSION" -lt "9" ] ; then
      JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -Xmn128m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
      JAVA_OPT="${JAVA_OPT} -XX:+UseConcMarkSweepGC -XX:+UseCMSCompactAtFullCollection -XX:CMSInitiatingOccupancyFraction=70 -XX:+CMSParallelRemarkEnabled -XX:SoftRefLRUPolicyMSPerMB=0 -XX:+CMSClassUnloadingEnabled -XX:SurvivorRatio=8 -XX:-UseParNewGC"
      JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:${GC_LOG_DIR}/rmq_srv_gc_%p_%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps"
      JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
    else
      JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m -XX:MetaspaceSize=128m -XX:MaxMetaspaceSize=320m"
      JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0"
      JAVA_OPT="${JAVA_OPT} -Xlog:gc*:file=${GC_LOG_DIR}/rmq_srv_gc_%p_%t.log:time,tags:filecount=5,filesize=30M"
    fi
}

choose_gc_log_directory
choose_gc_options
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib:${JAVA_HOME}/lib/ext"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

$JAVA ${JAVA_OPT} $@

```

docker run
```shell
# 无须先pull镜像，docker run之前会自动下载镜像

docker run -d --name rmqnamesrv -p 9876:9876 \
--privileged=true \
--network rocketmq \
-v /root/datamapping/rocketmq/namesrv/logs:/root/logs \
-v /root/datamapping/rocketmq/namesrv/store:/root/store \
-v /root/datamapping/rocketmq/namesrv/bin/runserver.sh:/home/rocketmq/rocketmq-4.9.2/bin/runserver.sh \
-e "MAX_POSSIBLE_HEAP=100000000" \
apache/rocketmq:4.9.2 sh mqnamesrv autoCreateTopicEnable=true


docker update rmqnamesrv --restart=always

```
参数说明：

- **--name rmqnamesrv**：指定容器名称为rmqnamesrv，**注意这个名字，后续会使用**。
- **--network rocketmq**：为容器指定网络为rocketmq，同一网络下的容器能够通过容器名称互通。
- **--privileged=true**：如果使用-v映射了目录，则使用该参数获取文件访问权限

查看启动成功后的日志：
```shell
docker logs -f rmqnamesrv
```
  
显示：
```shell
[root@iZuf6e9pay3s4hpvpmmepbZ ~]# docker logs -f rmqnamesrv
OpenJDK 64-Bit Server VM warning: Using the DefNew young collector with the CMS collector is deprecated and will likely be removed in a future release
OpenJDK 64-Bit Server VM warning: UseCMSCompactAtFullCollection is deprecated and will likely be removed in a future release.
The Name Server boot success. serializeType=JSON

```
# 部署broker，修改runbroker.sh
首先在宿主机创建broker的配置文件目录
```shell
sudo mkdir -p /root/datamapping/rocketmq/broker/conf
```
创建broker的配置文件broker.conf
```shell
vi /root/datamapping/rocketmq/broker/conf/broker.conf
```
broker.conf内容如下（记得修改brokerIP1的值为宿主机的ip地址）
```properties
brokerClusterName = DefaultCluster
brokerName=broker-a
brokerId=0
deleteWhen=04
fileReservedTime=48
brokerRole=ASYNC_MASTER
flushDiskType=ASYNC_FLUSH
brokerIP1=192.168.18.100
```

创建broker的sh文件runbroker.sh，否则文件挂载的时候就会把runbroker.sh当作目录对待
```shell
sudo mkdir -p /root/datamapping/rocketmq/broker/bin
vi /root/datamapping/rocketmq/broker/bin/runbroker.sh
```
打开提前修改生效，否则文件挂载的时候就会把runbroker.sh当作空内容看待以至于把容器内的runbroker.sh给冲掉，这里主要提前修改67行内存大小
```shell
#!/bin/sh

# Licensed to the Apache Software Foundation (ASF) under one or more
# contributor license agreements.  See the NOTICE file distributed with
# this work for additional information regarding copyright ownership.
# The ASF licenses this file to You under the Apache License, Version 2.0
# (the "License"); you may not use this file except in compliance with
# the License.  You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

#===========================================================================================
# Java Environment Setting
#===========================================================================================
error_exit ()
{
    echo "ERROR: $1 !!"
    exit 1
}

[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=$HOME/jdk/java
[ ! -e "$JAVA_HOME/bin/java" ] && JAVA_HOME=/usr/java
[ ! -e "$JAVA_HOME/bin/java" ] && error_exit "Please set the JAVA_HOME variable in your environment, We need java(x64)!"

export JAVA_HOME
export JAVA="$JAVA_HOME/bin/java"
export BASE_DIR=$(dirname $0)/..
export CLASSPATH=.:${BASE_DIR}/conf:${CLASSPATH}

#===========================================================================================
# JVM Configuration
#===========================================================================================
# The RAMDisk initializing size in MB on Darwin OS for gc-log
DIR_SIZE_IN_MB=600

choose_gc_log_directory()
{
    case "`uname`" in
        Darwin)
            if [ ! -d "/Volumes/RAMDisk" ]; then
                # create ram disk on Darwin systems as gc-log directory
                DEV=`hdiutil attach -nomount ram://$((2 * 1024 * DIR_SIZE_IN_MB))` > /dev/null
                diskutil eraseVolume HFS+ RAMDisk ${DEV} > /dev/null
                echo "Create RAMDisk /Volumes/RAMDisk for gc logging on Darwin OS."
            fi
            GC_LOG_DIR="/Volumes/RAMDisk"
        ;;
        *)
            # check if /dev/shm exists on other systems
            if [ -d "/dev/shm" ]; then
                GC_LOG_DIR="/dev/shm"
            else
                GC_LOG_DIR=${BASE_DIR}
            fi
        ;;
    esac
}

choose_gc_log_directory

#JAVA_OPT="${JAVA_OPT} -server -Xms8g -Xmx8g"
JAVA_OPT="${JAVA_OPT} -server -Xms256m -Xmx256m"
JAVA_OPT="${JAVA_OPT} -XX:+UseG1GC -XX:G1HeapRegionSize=16m -XX:G1ReservePercent=25 -XX:InitiatingHeapOccupancyPercent=30 -XX:SoftRefLRUPolicyMSPerMB=0"
JAVA_OPT="${JAVA_OPT} -verbose:gc -Xloggc:${GC_LOG_DIR}/rmq_broker_gc_%p_%t.log -XX:+PrintGCDetails -XX:+PrintGCDateStamps -XX:+PrintGCApplicationStoppedTime -XX:+PrintAdaptiveSizePolicy"
JAVA_OPT="${JAVA_OPT} -XX:+UseGCLogFileRotation -XX:NumberOfGCLogFiles=5 -XX:GCLogFileSize=30m"
JAVA_OPT="${JAVA_OPT} -XX:-OmitStackTraceInFastThrow"
JAVA_OPT="${JAVA_OPT} -XX:+AlwaysPreTouch"
JAVA_OPT="${JAVA_OPT} -XX:MaxDirectMemorySize=15g"
JAVA_OPT="${JAVA_OPT} -XX:-UseLargePages -XX:-UseBiasedLocking"
JAVA_OPT="${JAVA_OPT} -Djava.ext.dirs=${JAVA_HOME}/jre/lib/ext:${BASE_DIR}/lib:${JAVA_HOME}/lib/ext"
#JAVA_OPT="${JAVA_OPT} -Xdebug -Xrunjdwp:transport=dt_socket,address=9555,server=y,suspend=n"
JAVA_OPT="${JAVA_OPT} ${JAVA_OPT_EXT}"
JAVA_OPT="${JAVA_OPT} -cp ${CLASSPATH}"

numactl --interleave=all pwd > /dev/null 2>&1
if [ $? -eq 0 ]
then
	if [ -z "$RMQ_NUMA_NODE" ] ; then
		numactl --interleave=all $JAVA ${JAVA_OPT} $@
	else
		numactl --cpunodebind=$RMQ_NUMA_NODE --membind=$RMQ_NUMA_NODE $JAVA ${JAVA_OPT} $@
	fi
else
	$JAVA ${JAVA_OPT} $@
fi

```

部署broker

```shell
# 无须先pull镜像，docker run之前会自动下载镜像
docker run -d --name rmqbroker -p 10911:10911 -p 10909:10909 \
--privileged=true \
--network rocketmq \
-v /root/datamapping/rocketmq/broker/logs:/root/logs \
-v /root/datamapping/rocketmq/broker/store:/root/store \
-v /root/datamapping/rocketmq/broker/conf/broker.conf:/home/rocketmq/rocketmq-4.9.2/conf/broker.conf \
-v /root/datamapping/rocketmq/broker/bin/runbroker.sh:/home/rocketmq/rocketmq-4.9.2/bin/runbroker.sh \
-e "NAMESRV_ADDR=rmqnamesrv:9876" -e "MAX_POSSIBLE_HEAP=200000000" \
apache/rocketmq:4.9.2 sh mqbroker autoCreateTopicEnable=true -c /home/rocketmq/rocketmq-4.9.2/conf/broker.conf


docker update rmqbroker --restart=always
```
参数说明：
- **--network rocketmq**：为容器指定网络为rocketmq，同一网络下的容器能够通过容器名称互通。
- **-e "NAMESRV_ADDR=rmqnamesrv:9876"**：此处的rmqnamesrv就是容器name-server的名称
docker命令参数中：-v 和 -c的区别
-v：是数据卷挂载。将宿主机的文件路径挂载到容器中
-c：指向的是容器中的路径。
查看启动日志：
```shell
docker logs -f rmqbroker
```
日志显示
```shell
[root@iZuf6e9pay3s4hpvpmmepbZ ~]# docker logs -f rmqbroker
OpenJDK 64-Bit Server VM warning: If the number of processors is expected to increase from one, then you should configure the number of parallel GC threads appropriately using -XX:ParallelGCThreads=N
The broker[broker-a, 47.103.44.163:10911] boot success. serializeType=JSON and name server is rmqnamesrv:9876
```
从日志可以看出，已经获取到配置文件中broker的ip地址。  
name-server的地址则是由**name-server容器名称:端口号**组成。

下面来验证一下容器内部是否能够通过容器名称进行网络互联。  
验证思路为从一个容器内部ping另一个容器，看是否能够ping通。
```shell
# 进入broker容器
docker exec -it rmqbroker /bin/bash

# ping name-server的容器名称
ping rmqnamesrv
```

ping输出
```shell
[rocketmq@99a61d7a7f1d bin]$ ping rmqnamesrv
PING rmqnamesrv (172.18.0.2) 56(84) bytes of data.
64 bytes from rmqnamesrv.rocketmq (172.18.0.2): icmp_seq=1 ttl=64 time=0.050 ms
64 bytes from rmqnamesrv.rocketmq (172.18.0.2): icmp_seq=2 ttl=64 time=0.082 ms
^C
--- rmqnamesrv ping statistics ---
2 packets transmitted, 2 received, 0% packet loss, time 1000ms
rtt min/avg/max/mdev = 0.050/0.066/0.082/0.016 ms
```

能够ping通，说明网络是互通的，现在rocketmq就已经部署完毕了。
# 部署console


为了后续方便，这里再部署一个RocketMQ的控制台。
```shell
docker pull styletang/rocketmq-console-ng:latest
```
console的部署就不说那么多了，直接上命令。
```shell
# 同样在启动的时候指定同一个network
# 注意修改rocketmq.namesrv.addr后面的地址为容器名称:端口号

#左边是映射的外部端口，右边是容器内端口
docker run -d --name rmqconsole -p 11111:8080 --network rocketmq \
 -e "JAVA_OPTS=-Drocketmq.namesrv.addr=rmqnamesrv:9876" styletang/rocketmq-console-ng


docker update rmqconsole --restart=always

```
访问控制台：http://192.168.18.100:8080/#/
![[Pasted image 20240119154436.png]]