# 准备好挂载目录

## conf
创建conf目录与配置文件
```shell
mkdir -p /root/datamapping/mysql8/conf
vim /root/datamapping/mysql8/conf/hm.cnf
```
hm.cnf配置文件内容如下：
```shell
[client]
default-character-set=utf8mb4
[mysql]
default-character-set=utf8mb4
[mysqld]
init_connect='SET NAMES utf8mb4'
character-set-server=utf8mb4
collation-server=utf8mb4_unicode_ci
#忽略客户端信息并使用默认服务器字符集
skip-character-set-client-handshake
#禁止DNS解析
skip-name-resolve
#限制LOAD DATA, SELECT ... OUTFILE, and LOAD_FILE()传到哪个指定目录
secure_file_priv=/var/lib/mysql

```
## init可选（初始化sql）
创建init目录与配置文件
```shell
mkdir -p /root/datamapping/mysql8/init
vim /root/datamapping/mysql8/init/hmall.sql
```
hmall.sql文件内容如下。。。
```shell
也可以先不导入，等MySQL创建好之后再手动导入
```

# 开始安装
```shell
docker pull mysql:8.0.27
```
然后创建一个通用网络：
```Bash
docker network create hm-net
```
使用下面的命令来安装MySQL
```bash
docker run -d \
  --name mysql8 \
  -p 3308:3306 \
  -e TZ=Asia/Shanghai \
  -e MYSQL_ROOT_PASSWORD=root \
  -v /root/datamapping/mysql8/data:/var/lib/mysql \
  -v /root/datamapping/mysql8/conf:/etc/mysql/conf.d \
  -v /root/datamapping/mysql8/init:/docker-entrypoint-initdb.d \
  --network hm-net\
  mysql:8.0.27

#开机自启
docker update mysql8 --restart=always
```

# 配置远程访问
MySQL8默认没有开启远程访问，
进入mysql8容器
```shell
docker exec -it  mysql /bin/bash
mysql -u root -p
密码为root
```
先修改下mysql root用户密码（必须）
```shell
use mysql;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'root';
flush privileges;
```
开启远程访问并修改默认密码和加密方式
```shell
alter user 'root'@'%' identified with mysql_native_password by 'root';
flush privileges;
```
退出即可
