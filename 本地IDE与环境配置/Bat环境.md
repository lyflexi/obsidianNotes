## bat遍历文件内容

```Java
@echo off
setlocal enabledelayedexpansion

rem 遍历所有行
for /f "tokens=*" %%a in ('type pom.xml') do ( echo %%a )

rem 遍历所有行，默认以空格分隔，只取第一列
for /f %%a in ('type pom.xml') do (        echo %%a )

rem 遍历所有行，以.分隔，只取第一列，如果要取前几列，"token=1,2,3 delims=."
for /f "delims=." %%a in ('type pom.xml') do ( echo %%a )

rem 遍历所有行，以空格分隔，只取第一列
for /f %%a in ('findstr "version" pom.xml') do ( echo %%a )

rem 遍历所有行，以空格分隔，只取第一列，按通配符匹配
for /f %%a in ('findstr "<version>.*</version>" pom.xml') do ( echo %%a )
echo.
pause 


set element=
for /f "tokens=*" %%a in ('findstr "rt.jar]" "new 4.txt"') do (
call:getClassName "%%a" className || pause > nul
)
pause 
:getClassName
set e=%~1
for /f "tokens=2" %%b in ('echo %e%') do (
echo %%b
)
exit /b 0
```

BAT批处理有着具有非常强大的字符串处理能力，其功能虽没有C、[Python](https://so.csdn.net/so/search?from=pc_blog_highlight&q=Python)等高级编程语言丰富，但是常见的字符串截取、替换、连接、查找等功能也是应有尽有，本文逐一详细讲解。

## 字符串截取

百学不如一练，直接上字符串截取案例代码，如下： **vstr1.bat**

```Java
@echo off & setlocal

rem strlen=31
set str=This is a string function demo.

rem 倒数第5位开始，取4位：demo
echo %str:~-5,4%

rem 倒数第5位开始，取剩余所有字符：demo.
echo %str:~-5%

rem 倒数第100位开始，返回从0位开始的字符串：This is a string function demo.
echo %str:~-100%

rem 倒数第0位开始，取4位：This
echo %str:~0,4%

rem 倒数第0位开始，取所有字符：This is a string function demo.
echo %str:~0%

rem 倒数第0位开始，取100位超出长度，返回：This is a string function demo.
echo %str:~0,100%

rem 截取字符串赋值给变量
set str1=%str:~0,4%
echo str1=%str1%

rem 显示系统时间，去掉后面的毫秒显示
echo %time:~0,8%
```

## 字符串替换

替换字符串，即将某一字符串中的特定字符或字符串替换成指定的字符串，DEMO如下： **vstr2.bat**

```Java
@echo off & setlocal

set str1="This is a string replace demo!"
echo 替换前：str1=%str1%
echo 空格替换成#：str1=%str1: =#%
echo=

set str2="武汉，加油!"
echo 替换前：str2=%str2%
echo 武汉替换成中国：str2=%str2:武汉=中国%

    
输出结果如下:
替换前：str1="This is a string replace demo!"
空格替换成#：str1="This#is#a#string#replace#demo!"

替换前：str2="武汉，加油!"
武汉替换成中国：str2="中国，加油!"
```

## 字符串连接

较常见编程语言用 + 或 strcat函数 进行字符串拼接而言，bat脚本的字符串连接则更简单，但是功能也相对较弱，DEMO如下：

```Java
@echo off & setlocal

set aa=武汉
set bb=加油
rem 武汉, 加油
echo %aa%, %bb%

rem 武汉, 加油 赋值给aa变量
set "aa=%aa%, %bb%"
echo aa=%aa%
pause
```

## 字符串查找

字符串查找有两个命令：find和findstr，简单理解findstr是find的加强版（除 /C 只显示匹配到的行数外，其它都可实现），并且支持正则表达式。两者具体用法可以查看使用说明：find /? 或者 findstr /?，简单上手可以参考下述DEMO。

**vfind_data.txt**

```Java
Hello, Marcus!
hello, marcus!
Please say hello
```

**rem vfind.bat**

```Java

@echo off & setlocal
rem vfind_data.txt中查找包含Hello字符串的行，区分大小写
rem 只找到Hello, Marcus!
find "Hello" vfind_data.txt

rem vfind_data.txt中查找包含Hello字符串的行，不区分大小写
rem 三行都会找到
find /i "hello" vfind_data.txt

rem vfind_data.txt中查找不包含please字符串的行，不区分大小写
rem 找到hello 开头的两行
find /v /i "please" vfind_data.txt

rem 字符串作为输入，查找该字符串中是否包含“hello”
rem 输出Hello, marcus!
echo Hello, marcus! | find /i "hello"
```

**rem vfindstr.bat**

```Java

@echo off & setlocal
rem 查找文件vfind_data.txt中包含Hello字符串的行，区分大小写
findstr "Hello" vfind_data.txt

rem 查找hello开头的行，不区分大小写；数字比较请排除双引号、空格干扰
findstr /i "^hello" vfind_data.txt

rem 查找hello结尾的行，不区分大小写；数字比较请排除双引号、空格干扰
rem 文件最后一行若不是空白行，则最后一行hello$ 匹配不到，字符串查找时hello$也匹配不到
findstr /i "hello$" vfind_data.txt

echo Hello, marcus! | findstr /i "hello"

rem 找到输出found，没找到输出not found
echo Hello, marcus! | findstr /i "hello" > nul && (echo found) || (echo not found)
```