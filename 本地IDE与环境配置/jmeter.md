
在【我的电脑】右击，选择属性，选择【高级系统设置】，选择标签【高级】，点击环境变量  
1.在系统变量下方点击新建，变量名填写JAVA_HOME，变量值填写jdk安装路径（我的jdk安装目录：C:\Program Files\Java\jdk1.8.0_181)，如下图
![[Pasted image 20240120201018.png]]
2.再次点击新建（若有CLASSPATH，则选择编辑），变量名填写CLASSPATH，变量值填写 .;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar
![[Pasted image 20240120201027.png]]
3.在系统变量中找到Path，选中点击编辑，将%JAVA_HOME%\bin和%JAVA_HOME%\jre\bin填加进去；
![[Pasted image 20240120201040.png]]

jmeter下载地址：https://archive.apache.org/dist/jmeter/binaries/

