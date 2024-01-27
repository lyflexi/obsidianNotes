前往[nodejs官网](https://nodejs.org/)下载压缩包，，一般选择长期稳定版安装（下图中框出来的版本）
![[Pasted image 20240127170616.png]]

由于我们选择通过安装包的方式进行安装，所以点击上图中的箭头指向的Other Downloads，进入下面的界面
![[Pasted image 20240127170626.png]]

点击上图中框出来的地方，下载安装包（根据自己电脑配置选择32位或64位的压缩包，我的电脑是64位的，所以选择64-bit）

解压（我将其解压到：D:\env\node-v16.13.1）
![[Pasted image 20240127170634.png]]

配置环境变量

将解压后的根目录`D:\env\node-v16.13.1`和`D:\env\node-v16.13.1\node_global`目录分别写入系统变量Path
![[Pasted image 20240127170640.png]]

进行到这一步就已经安装成功了，可以去cmd中测试是否安装成功：

```Bash
查看nodejs版本
node -v 
查看npm版本
npm -v 
```

正常情况下应该输出各自的版本，如下图：
![[Pasted image 20240127170713.png]]

设置npm全局模块目录和缓存目录

nodejs的npm全局模块和缓存默认安装目录在C盘的用户目录里面，占用C盘空间。可以将其手动更改到自己设置的目录，我选择将其统一放在nodejs目录里：在nodejs的解压目录中新建node_global（npm全局模块目录）和node_cache（缓存）目录
![[Pasted image 20240127170720.png]]

在命令行中执行以下两句命令

```Java
npm config set prefix "E:\environment\node-v18.17.1-win-x64\node_global" 
npm config set cache "E:\environment\node-v18.17.1-win-x64\node_cache" 
```

这两个命令执行完后cmd中不会有回复，

可以输入npm config ls -l查看所有配置，能够看到自己的修改。

也可以打开电脑C盘里自己用户的文件夹，会新生成一个.npmrc的文件，用记事本打开查看可以看到自己刚刚新生成的配置。
![[Pasted image 20240127170728.png]]
打开该文件：

![](https://x3r1317gt9.feishu.cn/space/api/box/stream/download/asynccode/?code=MzUwNzBlN2E2NTM5YTdlMjNmZjE2YTg5ZTg4ZDgzYTBfZm1UUmtieFVLM05vVFZEUUxVNWhueFN0YXUzd2s3ZmdfVG9rZW46RzBSVGJqcmgwb2xlVHF4d3ZKQ2NueHFabklkXzE3MDYzNDYzNjg6MTcwNjM0OTk2OF9WNA)

配置镜像源

npm默认的仓库地址是在国外网站，速度较慢，建议大家设置到淘宝镜像。但是切换镜像是比较麻烦的。

推荐一款切换镜像的工具：nrm，输入下面命令安装：

```Java
npm install nrm -g 
```

（命令说明：-g代表全局。如果不加-g参数，安装模块会默认装在当前打开cmd的目录的node_modules文件夹中。加了-g后会装到前面配置的node_global文件夹里的node_modules文件夹中）
![[Pasted image 20240127170749.png]]

安装好后通过以下命令查看npm的仓库列表：

```Shell
nrm ls 
#报错9:14
C:\Users\hasee>nrm ls
internal/modules/cjs/loader.js:1102
      throw new ERR_REQUIRE_ESM(filename, parentPath, packageJsonPath);
      ^

Error [ERR_REQUIRE_ESM]: Must use import to load ES Module: E:\environment\node-v14.17.1-win-x64\node_global\node_modules\nrm\node_modules\open\index.js
require() of ES modules is not supported.
require() of E:\environment\node-v14.17.1-win-x64\node_global\node_modules\nrm\node_modules\open\index.js from E:\environment\node-v14.17.1-win-x64\node_global\node_modules\nrm\cli.js is an ES module file as it is a .js file whose nearest parent package.json contains "type": "module" which defines all .js files in that package scope as ES modules.
Instead rename index.js to end in .cjs, change the requiring code to use import(), or remove "type": "module" from E:\environment\node-v14.17.1-win-x64\node_global\node_modules\nrm\node_modules\open\package.json.

    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1102:13)
    at Module.load (internal/modules/cjs/loader.js:950:32)
    at Function.Module._load (internal/modules/cjs/loader.js:790:14)
    at Module.require (internal/modules/cjs/loader.js:974:19)
    at require (internal/modules/cjs/helpers.js:92:18)
    at Object.<anonymous> (E:\environment\node-v14.17.1-win-x64\node_global\node_modules\nrm\cli.js:9:14)
    at Module._compile (internal/modules/cjs/loader.js:1085:14)
    at Object.Module._extensions..js (internal/modules/cjs/loader.js:1114:10)
    at Module.load (internal/modules/cjs/loader.js:950:32)
    at Function.Module._load (internal/modules/cjs/loader.js:790:14) {
  code: 'ERR_REQUIRE_ESM'
}
#报错解决：现在open v9.0.0 是ESModule版本的包，应该使用open的CommonJs规范的包 
npm install -g nrm open@8.4.2 --save
nrm ls 
```
![[Pasted image 20240127170803.png]]

本机测试淘宝镜像最快。输入命令：

```Java
nrm test 
```
![[Pasted image 20240127170808.png]]

其中标*号的是目前使用的镜像。最好设置为淘宝镜像源，不建议使用cnpm（可能会出现奇怪的问题）

```Java
nrm use taobao 
```
![[Pasted image 20240127170814.png]]

配置好后可以去第3节中提到的.npmrc文件夹中查看：
![[Pasted image 20240127170819.png]]

以上，就是nodejs的全部安装过程了。