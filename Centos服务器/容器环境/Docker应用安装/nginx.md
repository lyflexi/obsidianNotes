

```Bash
docker pull nginx:1.10
# 随便启动一个nginx实例，只是为了复制出配置，放到docker里作为镜像的统一配置
docker run -p 80:80 --name nginx -d nginx:1.10

cd /mydata/nginx
docker container cp nginx:/etc/nginx .
然后在外部 /mydata/nginx/nginx 有了一堆文件
mv /mydata/nginx/nginx /mydata/nginx/conf
# 停掉nginx
docker stop nginx
docker rm nginx

# 创建新的nginx
docker run -p 80:80 --name nginx \
-v /mydata/nginx/html:/usr/share/nginx/html \
-v /mydata/nginx/logs:/var/log/nginx \
-v /mydata/nginx/conf:/etc/nginx \
-d nginx:1.10

# 注意一下这个路径映射到了/usr/share/nginx/html，我们在nginx配置文件中是写/usr/share/nginx/html，不是写/mydata/nginx/html

docker update nginx --restart=always
```

  

测试

```Bash
cd /mydata/nginx/html/
vim index.html
随便写写
测试 http://192.168.56.10:80
```