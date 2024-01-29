关于命令的使用区别我们解一哈：

```Shell
在linux下有些命令这样使用ls -a（参数前一横）；
有些命令这样使用cp --help(参数前两横)；
```

第一种：参数用一横的说明后面的参数是字符形式。

第二种：参数用两横的说明后面的参数是单词形式,比如:

```Shell
[root@col1422-10-45-47-142 demo]# grep --help
用法: grep [选项]... PATTERN [FILE]...
在每个 FILE 或是标准输入中查找 PATTERN。
默认的 PATTERN 是一个基本正则表达式(缩写为 BRE)。
例如: grep -i 'hello world' menu.h main.c

正则表达式选择与解释:
  -E, --extended-regexp     PATTERN 是一个可扩展的正则表达式(缩写为 ERE)
  -F, --fixed-strings       PATTERN 是一组由断行符分隔的定长字符串。
  -G, --basic-regexp        PATTERN 是一个基本正则表达式(缩写为 BRE)
  -P, --perl-regexp         PATTERN 是一个 Perl 正则表达式
  -e, --regexp=PATTERN      用 PATTERN 来进行匹配操作
  -f, --file=FILE           从 FILE 中取得 PATTERN
  -i, --ignore-case         忽略大小写
  -w, --word-regexp         强制 PATTERN 仅完全匹配字词
  -x, --line-regexp         强制 PATTERN 仅完全匹配一行
  -z, --null-data           一个 0 字节的数据行，但不是空行

Miscellaneous:
  -s, --no-messages         suppress error messages
  -v, --invert-match        select non-matching lines
  -V, --version             display version information and exit
      --help                display this help text and exit

输出控制:
  -m, --max-count=NUM       NUM 次匹配后停止
  -b, --byte-offset         输出的同时打印字节偏移
  -n, --line-number         输出的同时打印行号
      --line-buffered       每行输出清空
  -H, --with-filename       为每一匹配项打印文件名
  -h, --no-filename         输出时不显示文件名前缀
      --label=LABEL         将LABEL 作为标准输入文件名前缀
  -o, --only-matching       show only the part of a line matching PATTERN
  -q, --quiet, --silent     suppress all normal output
      --binary-files=TYPE   assume that binary files are TYPE;
                            TYPE is 'binary', 'text', or 'without-match'
  -a, --text                equivalent to --binary-files=text
  -I                        equivalent to --binary-files=without-match
  -d, --directories=ACTION  how to handle directories;
                            ACTION is 'read', 'recurse', or 'skip'
  -D, --devices=ACTION      how to handle devices, FIFOs and sockets;
                            ACTION is 'read' or 'skip'
  -r, --recursive           like --directories=recurse
  -R, --dereference-recursive
                            likewise, but follow all symlinks
      --include=FILE_PATTERN
                            search only files that match FILE_PATTERN
      --exclude=FILE_PATTERN
                            skip files and directories matching FILE_PATTERN
      --exclude-from=FILE   skip files matching any file pattern from FILE
      --exclude-dir=PATTERN directories that match PATTERN will be skipped.
  -L, --files-without-match print only names of FILEs containing no match
  -l, --files-with-matches  print only names of FILEs containing matches
  -c, --count               print only a count of matching lines per FILE
  -T, --initial-tab         make tabs line up (if needed)
  -Z, --null                print 0 byte after FILE name

文件控制:
  -B, --before-context=NUM  打印以文本起始的NUM 行
  -A, --after-context=NUM   打印以文本结尾的NUM 行
  -C, --context=NUM         打印输出文本NUM 行
  -NUM                      same as --context=NUM
      --group-separator=SEP use SEP as a group separator
      --no-group-separator  use empty string as a group separator
      --color[=WHEN],
      --colour[=WHEN]       use markers to highlight the matching strings;
                            WHEN is 'always', 'never', or 'auto'
  -U, --binary              do not strip CR characters at EOL (MSDOS/Windows)
  -u, --unix-byte-offsets   report offsets as if CRs were not there
                            (MSDOS/Windows)

‘egrep’即‘grep -E’。‘fgrep’即‘grep -F’。
直接使用‘egrep’或是‘fgrep’均已不可行了。
若FILE 为 -，将读取标准输入。不带FILE，读取当前目录，除非命令行中指定了-r 选项。
如果少于两个FILE 参数，就要默认使用-h 参数。
如果有任意行被匹配，那退出状态为 0，否则为 1；
如果有错误产生，且未指定 -q 参数，那退出状态为 2。

请将错误报告给: bug-grep@gnu.org
GNU Grep 主页: <http://www.gnu.org/software/grep/>
GNU 软件的通用帮助: <http://www.gnu.org/gethelp/>
```

# 文件列表

```Shell
ls -l: 列表展示
ls -a：查看文件，包含隐藏文件
ls -t：按照时间排序 -r逆序
```

# 文件查找

Find命令的一般形式为：

`find pathname -options [-print -exec -ok]`

让我们来看看该命令的参数：

`pathname`

pathname find命令所查找的目录路径。例如用.来表示当前目录，用/来表示系统根目录。

`选项`

find命令有很多选项，每一个选项前面跟随一个横杠 -。最常用的就是

- -name选项 按照文件名查找文件。
    
- -mtime -n +n 按照文件的更改时间来查找文件， -n表示文件更改时间距现在n天以内，+n表示文件更改时间距现在n天以前。Find命令还有-atime和-ctime选项，但它们都和-mtime选项相似，所以我们在这里只介绍-mtime选项。
    
- -type 查找某一类型的文件，诸如：
    
    - b - 块设备文件。
        
    - d - 目录。
        
    - c - 字符设备文件。
        
    - p - 管道文件。
        
    - l - 符号链接文件。
        
    - f - 普通文件。
        

`命令脚本`

- -print find命令将匹配的文件输出到标准输出。
    
- -exec find命令对匹配的文件执行该参数所给出的 shell命令。相应命令的形式为 'comm- and' {} \;，注意{}和\；之间的空格。
    
- -ok 和-exec的作用相同，只不过以一种更为安全的模式来执行该参数所给出的 shell命令， 在执行每一个命令之前，都会给出提示，让用户来确定是否执行。
    

## 按照文件名字查找

```Java
文件名选项是find命令最常用的选项，要么单独使用该选项，要么和其他选项一起使用。可以使用某种文件名模式来匹配文件，记住要用引号将文件名模式引起来。
不管当前路径是什么，如果想要在自己的根目录 $HOME中查找文件名符合*.txt的文件， 使用~作为'pathname参数，波浪号~代表了你的$HOME目录。
$ find ~ -name "*.txt" -print 
想要在当前目录及子目录中查找所有的‘ *.txt’文件，可以用：
$ find . -name "*.txt" -print 
想要的当前目录及子目录中查找文件名以一个大写字母开头的文件，可以用：
$ find . -name "[A-Z]*" -print 
想要在/etc目录中查找文件名以host开头的文件，可以用：
$ find /etc -name "host*" -print 
想要查找$HOME目录中的文件，可以用：
$ find ~ -name "*" -pr i或n tfind . -print 
要想让系统高负荷运行，就从根目录开始查找所有的文件。如果希望在系统管理员那里保留一个好印象的话，最好在这么做之前考虑清楚！
$ find / -name "*" -print 
如果想在当前目录查找文件名以两个小写字母开头，跟着是两个数字，最后是 *.txt的文件，下面的命令就能够返回名为ax37.txt的文件：
$ find . -name "[a-z][a-z][0--9][0--9].txt" -print 
```

## 按照更改时间查找文件

```Java
如果希望按照更改时间来查找文件，可以使用mtime选项。如果系统突然没有可用空间了， 很有可能某一个文件的长度在此期间增长迅速，这时就可以用 mtime选项来查找这样的文件。用减号-来限定更改时间在距今 n日以内的文件，而用加号 +来限定更改时间在距今n日以前的文件。
希望在系统根目录下查找更改时间在 5日以内的文件，可以用：
$ find / -mtime -5 -print 
为了在/var/adm目录下查找更改时间在3日以前的文件，可以用：
$ find /var/adm -mtime +3 -print 
```

## 使用exec或ok来执行shell命令

当匹配到一些文件以后，可能希望对其进行某些操作，这时就可以使用 -exec选项。一旦find命令匹配到了相应的文件，就可以用 -exec选项中的命令对其进行操作（在有些操作系统中只允许-exec选项执行诸如ls或ls -l这样的命令）。大多数用户使用这一选项是为了查找旧文件并删除它们。这里我强烈地建议你在真正执行 rm命令删除文件之前，最好先用 ls命令看一下，确认它们是所要删除的文件。exec选项后面跟随着所要执行的命令，然后是一对儿 {}，一个空格和一个\，最后是一个分号。为了使用exec选项，必须要同时使用print选项。

如果验证一下find命令，会发现该命令只输出从当前路径起的相对路径及文件名。为了用ls -l 命令列出所匹配到的文件，可以把ls -l 命令放在find命令的-exec选项中，例如：

```Java
root@crms-10-10-178-147[/root]# find . -name "*.txt" -print 
./.minikube/logs/lastStart.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip/_vendor/vendor.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/LICENSE.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/entry_points.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/top_level.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/LICENSE.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/entry_points.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/top_level.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/setuptools-58.3.0-py3-none-any/setuptools-58.3.0.dist-info/entry_points.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/setuptools-58.3.0-py3-none-any/setuptools-58.3.0.dist-info/top_level.txt
./quote.txt


root@crms-10-10-178-147[/root]# find . -name "*.txt" -print -exec ls -l {} \;
./.minikube/logs/lastStart.txt
-rw-r--r--. 1 root root 19406 Dec  8 11:37 ./.minikube/logs/lastStart.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip/_vendor/vendor.txt
-rw-r--r-- 1 root root 432 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip/_vendor/vendor.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/LICENSE.txt
-rw-r--r-- 1 root root 1090 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/LICENSE.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/entry_points.txt
-rw-r--r-- 1 root root 125 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/entry_points.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/top_level.txt
-rw-r--r-- 1 root root 4 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/pip-21.3.1-py3-none-any/pip-21.3.1.dist-info/top_level.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/LICENSE.txt
-rw-r--r-- 1 root root 1125 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/LICENSE.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/entry_points.txt
-rw-r--r-- 1 root root 108 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/entry_points.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/top_level.txt
-rw-r--r-- 1 root root 6 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/wheel-0.37.0-py2.py3-none-any/wheel-0.37.0.dist-info/top_level.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/setuptools-58.3.0-py3-none-any/setuptools-58.3.0.dist-info/entry_points.txt
-rw-r--r-- 1 root root 2636 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/setuptools-58.3.0-py3-none-any/setuptools-58.3.0.dist-info/entry_points.txt
./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/setuptools-58.3.0-py3-none-any/setuptools-58.3.0.dist-info/top_level.txt
-rw-r--r-- 1 root root 41 Dec 13 16:00 ./.local/share/virtualenv/wheel/3.6/image/1/CopyPipInstall/setuptools-58.3.0-py3-none-any/setuptools-58.3.0.dist-info/top_level.txt
./quote.txt
-rw-r--r-- 1 root root 55 Apr 13 15:59 ./quote.txt
```

find常用命令

```Java
我们已经介绍了find命令的基本选项，下面给出find命令的一些其他的例子。为了匹配$HOME目录下的所有文件，下面两种方法都可以使用：
$ find $$HOME -print $$ find ~ -print 
为了在当前目录中查找suid置位，文件属主具有读、写、执行权限，并且文件所属组的用户和其他用户具有读和执行的权限的文件，可以用：
$ find . -type f -perm 4755 -print 
为了查找系统中所有文件长度为 0的普通文件，并列出它们的完整路径，可以用：
$ find / -type f -size 0 -exec ls -l {} \; 
为了在/logs目录中查找更改时间在5日以前的文件并删除它们，可以用：记住，在shell中用任何方式删除文件之前，应当先查看相应的文件，一定要小心！
$ find logs -type f -mtime +5 -exec rm {} \; 
为了查找/var/logs目录中更改时间在7日以前的普通文件，并删除它们，可以用：
$ find /var/logs -type f -mtime +7 -exec rm {} \; 
为了查找系统中所有属于audit组的文件，可以用：
$find /-name -group audit -print 
我们的一个审计系统每天创建一个审计日志文件。日志文件名的最后含有数字，这样我们一眼就可以看出哪个文件是最新的，哪个是最旧的。        A d m i n . l o g 文件编上了序号：
admin.log.001、admin.log.002等等。下面的find命令将删除/logs目录中访问时间在 7日以前、含有数字后缀的admin.log文件。该命令只检查三位数字，所以相应日志文件的后缀不要超过
999。
$ find /logs -name 'admin.log[0-9][0-9 ]'[-0a-t9i]me +7 -exec rm {} \; 
为了查找当前文件系统中的所有目录并排序，可以用：
$ find . -type d -print -local -mount |sort 
为了查找系统中所有的rmt磁带设备，可以用：
$ find /dev/rmt -print 
```

# xargs

## 标准输入与管道

Unix 命令都带有参数，有些命令可以接受"标准输入"（stdin）作为参数，比如：

```Java
 $ cat /etc/passwd | grep root 
```

上面的代码使用了管道命令（|）。管道命令的作用，是将左侧命令（cat /etc/passwd）的标准输出转换为标准输入，提供给右侧命令（grep root）作为参数。因为grep命令可以接受标准输入作为参数，所以上面的代码等同于下面的代码。

```Java
 $ grep root /etc/passwd 
```

但是，大多数命令都不接受标准输入作为参数，只能直接在命令行输入参数，这导致无法用管道命令传递参数。举例来说，echo命令就不接受管道传参。

```Java
#这段不会有输出。因为管道右侧的echo不接受管道传来的标准输入作为参数。 
$ echo "hello world" | echo 
```

## xargs 命令的作用

而xargs命令的作用，是将标准输入转为命令行参数。

```Java
#将管道左侧的标准输入，转为命令行参数hello world，传给第二个echo命令。
$ echo "hello world" | xargs echo 
hello world 
```

xargs命令的格式如下：

`$ xargs [-options] [command]`

真正执行的命令，紧跟在xargs后面，接受xargs传来的参数。

xargs的作用在于，大多数命令（比如rm、mkdir、ls）与管道一起使用时，都需要xargs将标准输入转为命令行参数。

```Java
#下面的代码等同于mkdir one two three。
#如果不加xargs就会报错，提示mkdir缺少操作参数。
$ echo "one two three" | xargs mkdir 
```

## xargs单独使用

大多数时候，xargs命令都是跟管道一起使用的。但是，它也可以单独使用。

输入xargs按下回车以后，命令行就会等待用户输入，作为标准输入。你可以输入任意内容，然后按下Ctrl d，表示输入结束，这时echo命令就会把前面的输入打印出来。

```Java
$ xargs 
hello (Ctrl + d) 
hello 
```

```Java
$ xargs find -name
"*.txt"(Ctrl + d) 
./foo.txt
./hello.txt
```

上面的例子输入xargs find -name以后，命令行会等待用户输入所要搜索的文件。用户输入"*.txt"，表示搜索当前目录下的所有 TXT 文件，然后按下Ctrl d，表示输入结束。这时就相当执行`find -name *.txt`。

## xagrs选项

```Java
xagrs -h
```

# 文件查看

```Shell
less命令功能：less命令的用法与more命令类似,也可以用来浏览超过一页的文件.
所不同的是less命令除了可以按空格键向下显示文件外,还可以利用上下键来卷动文件.
当要结束浏览时,只要在less命令的提示符“: ”下按Q键即可.

  
查看文件头10行
head -n 10 example.txt
查看文件尾10行
tail -n 10 example.txt


默认显示最后 10 行
tail notes.log        

要跟踪名为 notes.log 的文件的增长情况，请输入以下命令：
此命令显示 notes.log 文件的最后 10 行。当将某些行添加至 notes.log 文件时，tail 命令会继续显示这些行。 显示一直继续，直到您按下（Ctrl-C）组合键停止显示。
tail -f notes.log

显示文件 notes.log 的内容，从第 20 行至文件末尾:
tail -n +20 notes.log

显示文件 notes.log 的最后 10 个字符:
tail -c 10 notes.log
vi：启动Vi编辑器
```

grep

```Shell
#使用grep -o统计文件中某个字符串出现的次数
cat /etc/passwd | grep -o "sbin" | wc -l
grep -o :-o, --only-matching       show only the part of a line matching PATTERN
wc -l ：用来统计行数


[root@col1422-10-45-47-142 demo]# cat /etc/passwd | grep -o "sbin" | wc -l
31
[root@col1422-10-45-47-142 demo]# cat /etc/passwd | grep -o "sbin"
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
sbin
```

# 文件编辑

## vim

  

```Ruby
#跳到第一行：按两次“g”，
#跳到最后一行：按“shift+g”，
#跳到第n行： 底线命令模式:n

#跳转到当前行的第一个字符：在当前行按“0”。
#跳转到当前行的最后一个字符：在当前行按“$”。

#查找 /pattern Enter
#清空内容  命令模式输入 :%d
```

## sed

```Java
[root@www ~]# sed [-nefr] [动作]
选项与参数：
-n ：使用安静(silent)模式。在一般 sed 的用法中，所有来自 STDIN 的数据一般都会被列出到终端上。但如果加上 -n 参数后，则只有经过sed 特殊处理的那一行(或者动作)才会被列出来。
-e ：直接在命令列模式上进行 sed 的动作编辑；
-f ：直接将 sed 的动作写在一个文件内， -f filename 则可以运行 filename 内的 sed 动作；
-r ：sed 的动作支持的是延伸型正规表示法的语法。(默认是基础正规表示法语法)
-i ：直接修改读取的文件内容，而不是输出到终端。
 
动作说明： [n1[,n2]]function
n1, n2 ：不见得会存在，一般代表『选择进行动作的行数』，举例来说，如果我的动作是需要在 10 到 20 行之间进行的，则『 10,20[动作行为] 』
 
function：
a ：新增， a 的后面可以接字串，而这些字串会在新的一行出现(目前的下一行)～
c ：取代， c 的后面可以接字串，这些字串可以取代 n1,n2 之间的行！
d ：删除，因为是删除啊，所以 d 后面通常不接任何咚咚；
i ：插入， i 的后面可以接字串，而这些字串会在新的一行出现(目前的上一行)；
p ：列印，亦即将某个选择的数据印出。通常 p 会与参数 sed -n 一起运行～
s ：取代，可以直接进行取代的工作哩！通常这个 s 的动作可以搭配正规表示法！例如 1,20s/old/new/g 就是啦！


#示例一：
sed 's/night/NIGHT/' quote.txt
The honeysuckle band played all NIGHT long for only $90
#示例二：
要从$90 中删除$ 符号（记住这是一个特殊符号，必须用\ 屏蔽其特殊含义），在replacement-pattern部分不写任何东西，保留空白，但仍需要用斜线括起来。
sed 's/\$//' quote.txt'
The honeysuckle band played all night long for only 90
#示例三：就地修改，不打印到控制台
sed -i 's/旧/新/g' kubefate.yaml > kubefate_163.yaml
#示例四：就地修改，不打印到控制台   
sed -i "/master_public_dns/s/SPARK_PUBLIC_DNS=to_replace_ip/SPARK_PUBLIC_DNS=$SPARK_MASTER/g" ./resources/docker-compose.yml
```

  

# 文件权限

```Ruby
chmod 777 file.java
//file.java的权限-rwxrwxrwx，r表示读、w表示写、x表示可执行
```

# 打包

```Ruby
解包：tar -xzvf FileName.tar
打包：tar -zcvf efficientnet-yolo3-pytorch.tar.gz ./efficientnet-yolo3-pytorch  将后面的demo文件夹压缩命名为demo.tar.gz
打包：tar -zcvf map_out.tar.gz ./map_out  将后面的demo文件夹压缩命名为demo.tar.gz
打包：tar -zcvf test.tar.gz /test1 /test2 压缩多个，将test1，test2压缩成一个
列出压缩文件列表：tar -tzvf test.tar.gz
```

# 删除大文件rsync

假如你要在linux下删除大量文件，比如100万、1000万，像/var/spool/clientmqueue/的mail邮件， 像/usr/local/nginx/proxy_temp的nginx缓存等，那么rm -rf *可能就不好使了。

rsync提供了一些跟删除相关的参数，rsync的核心的是替换原理

```Shell
rsync --help | grep delete
   --del                   an alias for --delete-during
     --delete                delete extraneous files from destination dirs
     --delete-before         receiver deletes before transfer, not during
     --delete-during         receiver deletes during the transfer
     --delete-delay          find deletions during, delete after
     --delete-after          receiver deletes after transfer, not during
     --delete-excluded       also delete excluded files from destination dirs
     --delete-missing-args   delete missing source args from destination
     --ignore-errors         delete even if there are I/O errors
     --max-delete=NUM        don't delete more than NUM files
```

其中--delete-before 指接收者在传输之前进行删除操作

当SRC和DEST文件性质不一致时将会报错

当SRC和DEST性质都为文件【f】时，意思是清空文件内容而不是删除文件

当SRC和DEST性质都为目录【d】时，意思是删除该目录下的所有文件，使其变为空目录

最重要的是，它的处理速度相当快，处理几个G的文件也就是秒级的事

```Shell
#创建空文件 
touch /data/blank.txt
#用rsync清空文件，比如nohup.out这样的实时更新的文件，动辄都是几十个G上百G的
rsync -a --delete-before -progress –stats /root/blank.txt /root/nohup.out



#也可以用来清空目录，如下，先建立一个空目录 
mkdir /data/blank
#用rsync删除目标目录 
rsync --delete-before -d /data/blank/ /var/spool/clientmqueue/
rsync --delete-before -d /work/wefe/blank/ /work/wefe/volume/
rsync --delete-before -d /data/wefe/blank/ /data/wefe/wefe_spark_cluster/
rsync --delete-before -d /root/blank/ /root/docker-compose/
rsync --delete-before -d /root/blank/ /root/cmake-3.17.0-Linux-x86_64/
rsync --delete-before -d /root/blank/ /root/efficientnet-yolo3-pytorch/
```

# **服务器间文件拷贝**

命令 scp -r 要拷贝的文件目录 root@目标服务器IP:/拷贝之后存放目录

scp -r docker-demo.tar root@192.168.243.129:/usr/local

然后根据提示输入服务器密码即可进行复制。

意思：把192.168.243.128 服务器上 docker-demo.tar 文件远程复制到 192.168.243.129 服务器上并存放在 /usr/local 目录下。
![[Pasted image 20240127163645.png]]