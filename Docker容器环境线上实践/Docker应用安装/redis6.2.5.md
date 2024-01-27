注意，使用redis要关闭本地VPN

```Shell
docker pull redis:6.2.5

#创建配置文件
# 1、准备redis配置文件内容,如果直接挂载的话docker会以为挂载的是一个目录，所以我们先创建一个文件然后再挂载，在虚拟机中。
mkdir -p /root/datamapping/redis/conf
vi /root/datamapping/redis/conf/redis.conf


#配置示例
appendonly yes
port 6379
bind 0.0.0.0

#docker启动redis
docker run -d -p 6379:6379 --restart=always \
-v /root/datamapping/redis/conf/redis.conf:/etc/redis/redis.conf \
-v  /root/datamapping/redis/data:/data \
 --name redis redis:6.2.5 \
 redis-server /etc/redis/redis.conf
```


# 设置密码（可选）
```Bash
docker pull redis:3.2.10
docker run -p 6379:6379 --name redis \
-v /root/datamapping/redis/data:/data \
-v /root/datamapping/redis/conf/redis.conf:/etc/redis/redis.conf \
-d redis:3.2.10 redis-server /etc/redis/redis.conf

# 直接进去redis客户端。
docker exec -it redis redis-cli
exit
```

默认是不持久化的。在配置文件中输入appendonly yes，就可以aof持久化了。修改完docker restart redis，docker -it redis redis-cli

```Bash
vim /root/datamapping/redis/conf/redis.conf
# 插入下面内容，aof持久化
appendonly yes
# 插入下面内容，设置密码
requirepass 123456 
保存

docker restart redis
```


设置redis容器自动启动

```Shell
docker update redis --restart=always
```