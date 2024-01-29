# keytool命令生成证书

打开要生成证书的目录，在视图框输入cmd回车，会弹出小黑框
![[Pasted image 20240129213123.png]]

输入keytool命令，其中-keystore指定自定义证书名称

```Java
keytool -genkeypair -alias "boot" -keyalg "RSA" -keystore "seek.keystore"
```

接着一步步输入信息，确认输入，秘钥就生成了

# 将证书放到springboot项目中
![[Pasted image 20240129213131.png]]

在applicaion.properties配置类中配置一下证书信息

```Properties
server.port=8006
server.ssl.key-store= classpath:seek.keystore
server.ssl.key-store-password=123456
server.ssl.keyStoreType=jks
server.ssl.keyAlias=boot
```

# 启动测试

1.浏览器访问路径https://localhost:8006/seek.html回车

![[Pasted image 20240129213158.png]]

点击高级，点击继续前往，请求成功