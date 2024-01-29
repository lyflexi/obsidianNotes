![[Pasted image 20240129100210.png]]
依赖管理，以任一个pom.xml为例，都有`<parent>`标签里的`<artifactId>spring-boot-starter-parent</artifactId>`

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.atguigu</groupId>
    <artifactId>boot-01-helloworld</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>


    <parent>
        <groupId>org.springframework.boot</groupId>
        <!-- 版本仲裁机制 -->
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>2.3.4.RELEASE</version>
    </parent>

    <dependencies>
        <dependency>
            ...
        </dependency>
    ...
    </dependencies>
</project>
```

点进`<artifactId>spring-boot-starter-parent</artifactId>`

```XML
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance">
  <modelVersion>4.0.0</modelVersion>
  <parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-dependencies</artifactId>
    <version>2.3.4.RELEASE</version>
  </parent>
  <artifactId>spring-boot-starter-parent</artifactId>
  <packaging>pom</packaging>
  <name>spring-boot-starter-parent</name>
  <description>Parent pom providing dependency and plugin management for applications built with Maven</description>
  <properties>
    <java.version>1.8</java.version>
    <resource.delimiter>@</resource.delimiter>
    <maven.compiler.source>${java.version}</maven.compiler.source>
    <maven.compiler.target>${java.version}</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <project.reporting.outputEncoding>UTF-8</project.reporting.outputEncoding>
  </properties>
</project>
```

再点进`<artifactId>spring-boot-dependencies</artifactId>`，下面几乎声明了所有开发中常用的依赖的版本号,自动版本仲裁机制

```XML
<properties>
  <activemq.version>5.15.13</activemq.version>
  <antlr2.version>2.7.7</antlr2.version>
  <appengine-sdk.version>1.9.82</appengine-sdk.version>
  <artemis.version>2.12.0</artemis.version>
  <aspectj.version>1.9.6</aspectj.version>
  <assertj.version>3.16.1</assertj.version>
  ...
  <mysql.version>8.0.17</mysql.version>
  ...
</properties>
```

所以以后我们开发，无需关注版本号，自动版本仲裁，引入依赖默认都可以不写版本

那么我们引入依赖就可以不用写版本，就会默认使用底层的对应依赖版本号

```XML
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
</dependency>
```

也可以修改默认版本号，模仿底层的版本写法

```XML
1、查看spring-boot-dependencies里面规定当前依赖的版本用的 key。
2、在当前项目里面重写properties配置
<properties>
  <mysql.version>5.1.43</mysql.version>
</properties>
<dependency>
  <groupId>mysql</groupId>
  <artifactId>mysql-connector-java</artifactId>
</dependency>
```

但是如果以后引入非版本仲裁的jar【还是我们说的第三方的jar包】，要写版本号
```xml
<!-- https://mvnrepository.com/artifact/com.alibaba/druid -->
<dependency>
	<groupId>com.alibaba</groupId>
	<artifactId>druid</artifactId>
	<version>1.2.16</version>
</dependency>
```
