# IDEA内置setting.xml配置

```XML
<?xml version="1.0" encoding="UTF-8"?>

<settings xmlns="http://maven.apache.org/SETTINGS/1.0.0"
  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/SETTINGS/1.0.0 http://maven.apache.org/xsd/settings-1.0.0.xsd">

  <localRepository>E:\environment\maven-respository</localRepository>


  <mirrors>
    <mirror>
      <id>repo1</id>
      <name>aliyun maven</name>
      <!-- <url>http://10.45.40.209/nexus/content/groups/public</url> -->
      <!-- <url>http://10.45.47.116/nexus/content/groups/public/</url> -->
      <!-- <url>http://10.45.47.168:8081/nexus/content/groups/public/</url> -->   <!-- 广研-->
      <url>http://maven.aliyun.com/nexus/content/groups/public/</url> <!-- 阿里云 -->
      <!-- <url>http://gz.iwhalecloud.com:6060/nexus/content/groups/public/</url> -->
      <mirrorOf>central</mirrorOf>
    </mirror>
    <mirror>
      <id>repo2</id>
      <name>guangyan maven</name>
      <!-- <url>http://10.45.40.209/nexus/content/groups/public</url> -->
      <!-- <url>http://10.45.47.116/nexus/content/groups/public/</url> -->
      <url>http://10.45.47.168:8081/nexus/content/groups/public/</url>   <!-- 广研-->
      <!-- <url>http://maven.aliyun.com/nexus/content/groups/public/</url> --><!-- 阿里云 -->
      <!-- <url>http://gz.iwhalecloud.com:6060/nexus/content/groups/public/</url> -->
      <mirrorOf>central</mirrorOf>
    </mirror>
    <!-- 中央仓库1 -->
    <mirror>
      <id>repo1</id>
      <mirrorOf>central</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://repo1.maven.org/maven2/</url>
    </mirror>

    <!-- 中央仓库2 -->
    <mirror>
      <id>repo2</id>
      <mirrorOf>central</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://repo2.maven.org/maven2/</url>
    </mirror>
  </mirrors>

  <profiles>
    <profile>
      <id>jdk-1.8</id>
      <activation>
        <activeByDefault>true</activeByDefault>
        <jdk>1.8</jdk>
      </activation>
      <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
        <maven.compiler.compilerVersion>1.8</maven.compiler.compilerVersion>
      </properties>
    </profile>
  </profiles>
</settings>
```

# Maven打包方式

我们在用maven构建java项目时，最常用的打包命令有

```PowerShell
mvn package
mvn install
mvn deploy
```

这三个命令都可完成打jar包或war（当然也可以是其它形式的包）的功能，但这三个命令还是有区别的。下面通过分别执行这三个命令的输出结果，来分析各自所执行的maven的生命周期。

## mvn clean package
![[Pasted image 20240127170431.png]]

![[Pasted image 20240127170440.png]]

  

## mvn clean install
![[Pasted image 20240127170448.png]]
![[Pasted image 20240127170454.png]]

  

  

## mvn clean deploy

（忽略最后的BUILD FAILURE）
![[Pasted image 20240127170501.png]]
![[Pasted image 20240127170509.png]]

  

仔细查看上面的输出结果截图，可以发现

mvn clean package依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)等７个阶段。

mvn clean install依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install等8个阶段。

mvn clean deploy依次执行了clean、resources、compile、testResources、testCompile、test、jar(打包)、install、deploy等９个阶段。

  

## package、install、deploy总结

通过三个命令的输出我们可以看出三者的区别在于包函的maven生命的阶段和执行目标(goal)不同

maven生命周期（lifecycle）由各个阶段组成，每个阶段由maven的插件plugin来执行完成。

```Java
    <build>
        <plugins>
            ...
             <plugin>
                 <groupId>org.apache.maven.plugins</groupId>
                 <artifactId>maven-jar-plugin</artifactId>
                 <version>2.6</version>
                 <configuration>
                    <archive>
                         <manifest>
                             <addClasspath>true</addClasspath>
                             <classpathPrefix>lib/</classpathPrefix>
                            <mainClass>com.xxx.xxxService</mainClass>
                       </manifest>
                    </archive>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-dependency-plugin</artifactId>
                <version>2.10</version>
                <executions>
                    <execution>
                        <id>copy-dependencies</id>
                        <phase>package</phase>
                        <goals>
                            <goal>copy-dependencies</goal>
                        </goals>
                        <configuration>
                            <outputDirectory>${project.build.directory}/lib</outputDirectory>
                        </configuration>
                    </execution>
                </executions>
            </plugin>

        </plugins>
    </build>
```

生命周期（lifecycle）主要包括clean、resources、complie、install、package、testResources、testCompile、deploy等，

其中带test开头的都是用业编译测试代码或运行单元测试用例的。

  

  

由上面的分析可知主要区别如下，

**package命令完成了项目编译、单元测试、打包功能，但没有把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库**

**install命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库，但没有布署到远程maven私服仓库**

**deploy命令完成了项目编译、单元测试、打包功能，同时把打好的可执行jar包（war包或其它形式的包）布署到本地maven仓库和远程maven私服仓库**

  

## 常用命令

```Shell
maven clean install
```

  

# scope分类说明

## compile

**compile：**默认scope为compile，表示为当前依赖参与项目的编译、测试和运行阶段，属于强依赖。打包之时，会达到包里去。

test：该依赖仅仅参与测试相关的内容，包括测试用例的编译和执行，比如定性的Junit。

runtime：依赖仅参与运行周期中的使用。一般这种类库都是接口与实现相分离的类库，比如JDBC类库，在编译之时仅依赖相关的接口，在具体的运行之时，才需要具体的mysql、oracle等等数据的驱动程序。

此类的驱动都是为runtime的类库。

## provided

**provided：**该依赖在打包过程中，不需要打进去，这个由外置的环境来提供，比如tomcat或者基础类库等等。

事实上，该依赖可以参与编译、测试和运行等周期，与compile等同。区别在于打包阶段进行了exclude操作。以springboot打war包为例：
![[Pasted image 20240127170529.png]]
![[Pasted image 20240127170534.png]]

## system

**system：**使用上与provided相同，不同之处在于该依赖不从maven仓库中提取，而是从本地文件系统中提取，其会参照systemPath的属性进行提取依赖。

## import

**import：**这个是maven2.0.9版本后出的属性，import只能在dependencyManagement的中使用，能解决maven单继承问题，import依赖关系实际上并不参与限制依赖关系的传递性。

# 异常报错

## Cannot download sources Sources not found

下载源码出现：Decompiled .class file ，右下角出现Cannot download sources Sources not found for: xxx

解决办法：在对应项目pom.xml所在目录下执行以下命令：

```Shell
mvn dependency:resolve -Dclassifier=sources
```