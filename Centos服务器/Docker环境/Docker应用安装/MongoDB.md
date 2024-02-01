
docker search mongo 命令来查看可用版本：

```shell
$ docker search mongo
NAME                              DESCRIPTION                      STARS     OFFICIAL   AUTOMATED
mongo                             MongoDB document databases ...   1989      [OK]       
mongo-express                     Web-based MongoDB admin int...   22        [OK]       
mvertes/alpine-mongo              light MongoDB container          19                   [OK]
mongooseim/mongooseim-docker      MongooseIM server the lates...   9                    [OK]
torusware/speedus-mongo           Always updated official Mon...   9                    [OK]
jacksoncage/mongo                 Instant MongoDB sharded cluster  6                    [OK]
mongoclient/mongoclient           Official docker image for M...   4                    [OK]
jadsonlourenco/mongo-rocks        Percona Mongodb with Rocksd...   4                    [OK]
asteris/apache-php-mongo          Apache2.4 + PHP + Mongo + m...   2                    [OK]
19hz/mongo-container              Mongodb replicaset for coreos    1                    [OK]
nitra/mongo                       Mongo3 centos7                   1                    [OK]
ackee/mongo                       MongoDB with fixed Bluemix p...  1                    [OK]
kobotoolbox/mongo                 https://github.com/kobotoolb...  1                    [OK]
valtlfelipe/mongo                 Docker Image based on the la...  1                    [OK]
```

拉取镜像

```shell
$ docker pull mongo:4.0
```

安装完成后，我们可以使用以下命令来运行 mongo 容器：

```shell
$ docker run -itd --name mongo -p 37017:27017 mongo:4.0 --auth

-p 37017:27017 ：映射容器服务的 27017 端口到宿主机的 37017 端口。
  外部可以直接通过 宿主机 ip:37017 访问到 mongo 的服务。
--auth：需要密码才能访问容器服务。

#接着使用以下命令添加用户和设置密码，并且尝试连接。
$ docker exec -it mongo mongo admin 
# 创建一个名为 admin，密码为 123456 的用户。 
>  db.createUser({ user:'admin',pwd:'123456',roles:[ { role:'userAdminAnyDatabase', db: 'admin'}]}); 
# 尝试使用上面创建的用户信息进行连接。 
> db.auth('admin', '123456')
```

  

其实也可以不使用--auth

```shell
docker run -itd --name mongo -p 37017:27017 mongo:4.0
docker restart mongo
docker update mongo --restart=always
```