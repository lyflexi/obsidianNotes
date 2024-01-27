# 前置操作

关闭selinux

```Shell
#查看selinux是否开启：
getenforce
#永久关闭selinux，需要重启：
sed -i 's/enforcing/disabled/' /etc/selinux/config
#临时关闭selinux，重启之后，无效：
setenforce 0
```

关闭防火墙

```Bash
#查看防火墙是否开启：
firewall-cmd --state
#临时关闭防火墙：
systemctl stop firewalld.service
#永久关闭防火墙：
systemctl disable firewalld.service
#开启防火墙：
systemctl start firewalld.service
```

# mysql5

目录映射，使得Linux虚拟机的/root/datamapping/mysql/目录文件，自动同步与容器内mysql同步

```Bash
sudo docker pull mysql:5.7

# --name指定容器名字 
# -v目录挂载 前面的目录是Linux物理服务器的本地目录，后面的是mysql容器内部的目录
# -p指定端口映射(前面的端口是Linux物理服务器的端口)  
# -e设置mysql参数 -d后台运行
sudo docker run -p 3306:3306 --name mysql \
-v /root/datamapping/mysql/log:/var/log/mysql \
-v /root/datamapping/mysql/data:/var/lib/mysql \
-v /root/datamapping/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7


sudo docker run -p 3147:3306 --name serving \
-v /root/datamapping/mysql/log:/var/log/mysql \
-v /root/datamapping/data:/var/lib/mysql \
-v /root/datamapping/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7

sudo docker run -p 3306:3306 --name 613tools-model \
-v /root/datamapping/mysql/log:/var/log/mysql \
-v /root/datamapping/mysql/data:/var/lib/mysql \
-v /root/datamapping/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7

sudo docker run -p 3306:3306 --name mysql \
-v /root/datamapping/mysql/log:/var/log/mysql \
-v /root/datamapping/mysql/data:/var/lib/mysql \
-v /root/datamapping/mysql/conf:/etc/mysql \
-e MYSQL_ROOT_PASSWORD=root \
-d mysql:5.7
```

创建MySQL配置文件

```Bash
[root@localhost vagrant]# docker ps
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                               NAMES
6a685a33103f        mysql:5.7           "docker-entrypoint.s…"   32 seconds ago      Up 30 seconds       0.0.0.0:3306->3306/tcp, 33060/tcp   mysql

docker exec -it mysql bin/bash
exit;


因为有目录映射，所以我们可以直接在镜像外执行
vi /root/datamapping/mysql/conf/my.conf 
#mysql5.7
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
[mysqld]
init_connect='SET collation_connection = utf8_unicode_ci'
init_connect='SET NAMES utf8'
character-set-server=utf8
collation-server=utf8_unicode_ci
skip-character-set-client-handshake
skip-name-resolve
```

开机自启

```Bash
docker restart mysql
docker update mysql --restart=always
```