# 查看进程

根据进程端口查看

```Shell
netstat -ano
netstat -ano|findstr 9000
```
![[Pasted image 20240127170906.png]]

杀死端口9000所对应的进程

```Shell
taskkill -PID 19796 -F
```

查看进程ID查看

```Shell
#tasklist查看进程id19796对应的服务名
tasklist|findstr 19796
```
![[Pasted image 20240127170911.png]]

# 任务栏图标空白

一、错误原因

在 Windows 10 系统中，为了加速图标的显示，当第一次对图标进行显示时，系统会对文件或程序的图标进行缓存。之后，当我们再次显示该图标时，系统会直接从缓存中读取数据，从而大大加快显示速度。

也正因为如此，当缓存文件出现问题时，就会引发系统图标显示不正常。既然找到了原因，解决办法也很简单，我们只需要将有问题的图标缓存文件删除掉，让系统重新建立图标缓存即可。

二、操作方法

首先，由于图标缓存文件是隐藏文件，我们需要在资源管理器中将设置改为“显示所有文件”。操作方法： 1）随便打开一个文件夹。 2）点击“查看”菜单，然后勾选“隐藏的项目”。 同时按下快捷键 Win+R，在打开的运行窗口中输入 %localappdata%，回车。 在打开的文件夹中，找到 Iconcache.db，将其删除。 在任务栏上右击鼠标，在弹出的菜单中点击“任务管理器”。 在任务管理器中找到“Windows资源管理器”，右击鼠标，选择“重新启动”即可重建图标缓存。

# 关闭用户帐户控制提示

以下是Win10关闭用户账户控制的具体方法，通过以下方法进行设置后，用户账户控制就能成功关闭了，再次安装或运行安装程序时，就不会再出现该提示了。方法/步骤

1、右击开始，选择运行(Win+R键);
![[Pasted image 20240127170918.png]]
2、输入gpedit.msc，回车;
![[Pasted image 20240127170925.png]]
3、计算机配置中找到Windows设置下的安全设置，将安全设置展开;

4、在安全设置中找到本地策略，打开找到安全选项;
![[Pasted image 20240127170930.png]]
5、在右侧策略列表中找到，用户账户控制：管理员批准模式中管理员的提升权限提示的行为;
![[Pasted image 20240127170936.png]]
6、双击管理员批准模式中管理员的提升权限提示的行为，弹出的窗口中选择不提示，直接提升，确定即可。
![[Pasted image 20240127170941.png]]

# 电脑按Fn加F1~F12才有用的解决方法

我是程序员，电脑是dell的，每次debug的时候就需要fn+f键，很不方便

解决办法：

直接按Fn+Esc就可以来回切换是否要按fn键来实现功能