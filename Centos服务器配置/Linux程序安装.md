# Java

```Java
tar -xzvf  jdk-8u191-linux-x64.tar.gz
vim /etc/profile  
```

添加下面内容：

```Java
#set java environment
export JAVA_HOME=/data/projects/FATE-Cloud/cloud-manager/common/jdk1.8.0_202
export JRE_HOME=${JAVA_HOME}/jre
export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib:$CLASSPATH
export JAVA_PATH=${JAVA_HOME}/bin:${JRE_HOME}/bin
export PATH=$PATH:${JAVA_PATH}
# maven settings
export MAVEN_HOME=/data/projects/FATE-Cloud/cloud-manager/common/apache-maven-3.8.3
export MAVEN_HOME
export PATH=$PATH:${MAVEN_HOME}/bin
```

生效

```Java
source /etc/profile      --使环境变量生效
#验证
java -version              --查看jdk是否安装成功
```

# maven

```Java
tar -zxvf apache-maven-3.6.1-bin.tar.gz
```

编辑/etc/profile

```Java
# maven settings
export MAVEN_HOME=/data/maven/apache-maven-3.6.1
export MAVEN_HOME
export PATH=$PATH:${MAVEN_HOME}/bin
```

生效

```Java
source /etc/profile
#验证
mvn -v
```

# python

如果你的机器上已经存在Python2.6的话不用删除，因为有很多程序是依赖Python2.6。

**我们先安装Python3.6所依赖的包：**

```Java
yum -y install zlib-devel bzip2-devel openssl-devel ncurses-devel sqlite-devel readline-devel tk-devel gdbm-devel db4-devel libpcap-devel xz-devel 
yum install -y gcc
```

**下载Python3包** 你可以从官网下载：

[https://www.python.org/downloads/](https://links.jianshu.com/go?to=https%3A%2F%2Fwww.python.org%2Fdownloads%2F)

```Java
wget https://www.python.org/ftp/python/3.6.1/Python-3.6.1.tgz
```

**根目录下创建Python3目录**

```Java
mkdir -p /Python3 
```

解压python3包

```Java
tar -zxvf Python-3.6.1.tgz
```

**解压完进入目录，编译安装**

```Java
cd Python-3.6.1 
./configure --prefix=/usr/local/python3 
make && make install
```

**创建python3软连接**

```Java
ln -s /Python3/bin/python3 /usr/bin/python3 
```

**加入PATH**

```Java
echo "PATH=$PATH:$HOME/bin:/usr/local/python3/bin export PATH" 
export PATH = $PATH:$HOME/bin:/usr/local/python3/bin
source ~/.bash_profile 
#保存修改 
```

**我们可以验证自己成功**

```Java
python3 -V #成功会显示python版本 
pip3 -V #成功会显示pip版本
```

**若提示软连接已存在（慎用！，很可能不需要下面的操作）**

```Java
#删除原有的python3软连接
rm -rf /usr/bin/python3 
#创建python3软连接
ln -s /Python3/bin/python3 /usr/bin/python3 
#验证python3软连接是否正常创建，正常会返回安装的版本信息
python3 -V 

#删除原有的pip3软连接
rm -rf /usr/bin/pip3 
#创建pip3软连接
ln -s /Python3/bin/pip3 /usr/bin/pip3 
#验证pip3软连接是否正常创建，正常创建会返回pip版本与关联的python版本信息
pip3 -V
```

# virtualenv

我们在开发项目时，如果直接在本机上安装Python包进行项目开发，所有项目会共用同一个Python， 它有一些缺点： _（1）修改这个用户的库可能影响你的系统上的其它Python 软件。_ _（2）你将不可以运行这个包的多个版本（或者具有相同名字的其它包）。_ 特别是一旦你维护几个项目，这些情况就会出现。 如果确实出现，最好的解决办法是使用virtualenv。 这个工具允许你维护多个分离的Python环境，每个都具有它自己的库和包的命名空间。这种情况下，每个应用可能需要各自拥有一套“独立”的Python运行环境。virtualenv就是用来为一个应用创建一套“隔离”的Python运行环境。

**我们用pip安装virtualenv：**

```Java
pip3 install virtualenv
```

**进入我们安放项目的目录，新建一个应用，配置一套独立的Python环境：**

首先，新建一个应用目录，进入该目录。

**然后，创建一个独立Python环境**

如果virtualenv版本大于20，virtualenv默认--no-site-packages这个参数，就执行virtualenv venv：

```Python
#创建一个叫作venv的新环境
virtualenv --no-site-packages venv 
```

命令virtualenv就可以创建一个独立的Python运行环境，我们还加上了参数--no-site-packages，这样，已经安装到系统Python环境中的所有第三方包都不会复制过来，这样，我们就得到了一个不带任何第三方包的“干净”的Python运行环境。 **进入venv环境并且激活**venv环境

```Python
cd venv
source bin/activate
```

激活之后，会在命令行提示符前面看到环境名称，提醒你当前处于虚拟环境中# 后面你安装的任何库和执行的任何程序都在这个环境下运行

  

```Java
#在虚拟环境中安装包、创建.py文件，并且使用requests模块做测试，并创建一个小文件
pip install -r requirement.txt
pip install requests 
...
vim test.py
```

编写一个简单的程序

```Java
import requests
 
res = requests.get('https://www.baidu.com')
# 打印请求状态码
print(res.status_code)
```

运行程序
![[Pasted image 20240127163745.png]]

退出虚拟环境，并在此测试刚才的脚本是否可以运行（答案当然是否定的）

```Java
deactivate
```

在退出虚拟环境后，编写的程序无法运行，因为本机环境中并没有安装requests模块。
![[Pasted image 20240127163756.png]]

# pip管理

如果没有epel源下载阿里的epel源

```Java
wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo 
```

安装pip

```Java
yum -y install python-pip 
```

验证

```Java
# pip list

backports.ssl-match-hostname (3.5.0.1)

chardet (2.2.1)

configobj (4.7.2)

decorator (3.4.0)

iniparse (0.4)

ipaddress (1.0.16)

IPy (0.75)
```

升级

```Java
pip install --upgrade pip==18.0 --trusted-host pypi.douban.com
pip install --upgrade pip
```

# pip3管理

**pip安装的库的默认位置**

```Java
python -m site

root@crms-10-10-178-147[/]# python3 -m site
sys.path = [
    '/',
    '/usr/local/python3/lib/python36.zip',
    '/usr/local/python3/lib/python3.6',
    '/usr/local/python3/lib/python3.6/lib-dynload',
    '/usr/local/python3/lib/python3.6/site-packages',
]
USER_BASE: '/root/.local' (exists)
USER_SITE: '/root/.local/lib/python3.6/site-packages' (doesn't exist)
ENABLE_USER_SITE: True
```

**pip3升级**

```Java
python3 -m pip install --upgrade pip
```

**pip3下载很慢**

```Java
阿里云 https://mirrors.aliyun.com/pypi/simple/

中国科技大学 https://pypi.mirrors.ustc.edu.cn/simple/

豆瓣 http://pypi.douban.com/simple/

清华大学 https://pypi.tuna.tsinghua.edu.cn/simple/

中国科学技术大学 http://pypi.mirrors.ustc.edu.cn/simple/
```

# spark环境变量

  

配置环境变量

编辑 /etc/profile 文件

  

```Java
# 配置spark
export SPARK_HOME=/root/soft/spark-2.1.1-bin-hadoop2.7
export PATH=$SPARK_HOME/bin:$PATH
# 配置spark结束
```

  

刷新配置文件

```Java
source /etc/profile
```

打印环境变量配置

```Java
[root@zjj101 soft]# echo $SPARK_HOME
/root/soft/spark-2.1.1-bin-hadoop2.7
```

# 安装7z

```Java
一. 先安装wget
yum -y install wget

二. 下载7z的压缩包
wget https://sourceforge.net/projects/p7zip/files/p7zip/16.02/p7zip_16.02_src_all.tar.bz2

三. 安装bzip
yum install -y bzip2

四. 解压压缩包
tar -jxvf p7zip_16.02_src_all.tar.bz2

五. 进入解压的压缩包目录
cd p7zip_16.02

六. 安装gcc和gcc+
yum install gcc

yum install gcc-c++

七. 执行make  和 make install
最后出现下面的即为安装好：

./install.sh /usr/local/bin /usr/local/lib/p7zip /usr/local/man /usr/local/share/doc/p7zip
- installing /usr/local/bin/7za
- installing /usr/local/man/man1/7z.1
- installing /usr/local/man/man1/7za.1
- installing /usr/local/man/man1/7zr.1
- installing /usr/local/share/doc/p7zip/README
- installing /usr/local/share/doc/p7zip/ChangeLog
- installing HTML help in /usr/local/share/doc/p7zip/DOC

解压7z文件的命令就是：7za x blog.7z
```