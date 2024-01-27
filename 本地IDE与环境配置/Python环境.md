# pip源

```Bash
#临时使用：python使用清华、阿里云、中科大源镜像安装包，例如：
pip3 install -i https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn "模块名比如tensorflow==2.1.0"
#永久配置镜像源，会帮当前用户创建代理文件，位置：Writing to C:\Users\hasee\AppData\Roaming\pip\pip.ini
pip config set global.index-url https://pypi.mirrors.ustc.edu.cn/simple --trusted-host pypi.mirrors.ustc.edu.cn
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple --trusted-host pypi.tuna.tsinghua.edu.cn
pip config set global.index-url https://mirrors.aliyun.com/pypi/simple --trusted-host mirrors.aliyun.com
#查看镜像源配置：
pip config get global.index-url
#恢复默认镜像配置
pip config unset global.index-url
```

永久配置镜像源，等价于修改代理文件，位置：Writing to C:\Users\hasee\AppData\Roaming\pip\pip.ini

```Bash
[global]
index-url = https://pypi.mirrors.ustc.edu.cn/simple
```

# conda环境

```Bash
#永久配置镜像源，等价于修改镜像源配置文件C:\Users\hasee\.condarc
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/msys2/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --set show_channel_urls yes
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/msys2/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/bioconda/
conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/menpo/
conda config --set show_channel_urls yes
#查看镜像源配置：
conda config --show
#恢复默认源
conda config --remove-key channels
#######################################################################################################虚拟环境
#创建虚拟环境
conda create -n env_name（环境名称） python=3.7(对应的python版本号）
conda create -n DeltaPSI python=3.8
#激活虚拟环境
conda activate env_name（环境名称）
#退出虚拟环境,即返回默认的python环境
deactivate env_name（环境名称）
#删除虚拟环境
conda remove -n env_name(环境名称) --all
#查看已创建的虚拟环境
conda env list  或 conda info -e  或  conda info --env
#克隆删除环境（若想修改某个虚拟环境的名字,anaconda中没有重命名的命令，使用克隆删除的方法）
（1）进入旧环境
conda activate old_name
（2）克隆旧环境
conda create -n new_name --clone old_name
（3）退出旧环境
conda deactivate
（4）删除旧环境
conda remove -n old_name --all
#环境回滚
conda list --revisions
conda install --rev 0
#######################################################################################################安装依赖
#conda install 命令可能报错：CondaSSLError: OpenSSL appears to be unavailable on this machine.，解决办法如下：
复制 D:\path_to_your_conda3\Library\bin 目录下的 libcrypto-1_1-x64.dll 及 libssl-1_1-x64.dll 到 D:\path_to_your_conda3\DLLs 目录下即可　　　　
#查看指定环境的已安装的包,不加-n则安装在当前活跃环境
conda list -n py27　　　
#查找package信息
conda search selenium
#指定环境安装package，不加-n则安装在当前活跃环境
conda install -n py27 selenium  
#指定环境更新package，不加-n则更新在当前活跃环境
conda update -n py27 selenium 
#删除package，不加-n则删除在当前活跃环境
conda remove -n py27 selenium
#清理（应该是pkgs文件下的）安装包缓存
conda clean --all
```

永久配置镜像源，等价于修改镜像源配置文件`C:\Users\hasee\.condarc`

```Bash
channels:
  - https://mirrors.ustc.edu.cn/anaconda/cloud/menpo/
  - https://mirrors.ustc.edu.cn/anaconda/cloud/bioconda/
  - https://mirrors.ustc.edu.cn/anaconda/cloud/msys2/
  - https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/
  - https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
  - https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/conda-forge/
  - https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/msys2/
  - defaults
show_channel_urls: true
ssl_verify: false
```

# Jupyter Notebook批量下载

因为Jupyter Notebook目前来说只是支持ipynp的下载，并且一次仅仅只能下载一个，影响了我们的效率,这组代码可以帮助大家对一个文件夹进行总下载，很方便、很快捷，希望可以帮助到大家。

1、在文件夹目录下新建一个Python界面然后将这段代码输入进去，并且运行一次进行保存：

这个图片是我没有进行任何操作的图片
![[Pasted image 20240127170018.png]]

```Java
import os
import tarfile

def recursive_files(dir_name='.', ignore=None):
    for dir_name,subdirs,files in os.walk(dir_name):
        if ignore and os.path.basename(dir_name) in ignore: 
            continue

        for file_name in files:
            if ignore and file_name in ignore:
                continue

            yield os.path.join(dir_name, file_name)

def make_tar_file(dir_name='.', tar_file_name='tarfile.tar', ignore=None):
    tar = tarfile.open(tar_file_name, 'w')

    for file_name in recursive_files(dir_name, ignore):
        tar.add(file_name)

    tar.close()


dir_name = '.'
tar_file_name = 'archive.tar'
ignore = {'.ipynb_checkpoints', '__pycache__', tar_file_name}
make_tar_file(dir_name, tar_file_name, ignore)
```

2、这张图片是我执行第一步操作后的

可以看到出现了archive.tar，然后进行第三步操作
![[Pasted image 20240127170026.png]]

  

3.选择archive.taz并且点击下载，就成功了。

如图片所示
![[Pasted image 20240127170033.png]]

# Jupyter Notebook转markdown，插件安装

把Markdown转成JupyterNotebook，一般情况下很少用到。偶尔要用时，又一时想不起链接在哪。所以这里记录一下。

  

之所以附带上《动手学深度学习》这本书（[http://zh.d2l.ai/index.html](http://zh.d2l.ai/index.html)），是因为里面的教程都是markdown写的，可以做为例子练练手，很好地转换成JupyterNotebook格式，为此，李沐还专门改写了一个对中文支持更好的版本notedown，

[https://github.com/mli/notedown](https://github.com/mli/notedown)

这个版本的原版本是

[https://github.com/aaren/notedown](https://github.com/aaren/notedown)

  

下面简单介绍一下使用方法（假设你已经安装好了JupyterNotebook)。

用Jupyter记事本读写GitHub源文件，根据[http://zh.gluon.ai/chapter_appendix/jupyter.html](http://zh.gluon.ai/chapter_appendix/jupyter.html)描述，下面安装notedown插件，运行Jupyter记事本并加载插件。

```Java
pip install https://github.com/mli/notedown/tarball/master
jupyter notebook --NotebookApp.contents_manager_class='notedown.NotedownContentsManager'
```

如果想每次运行Jupyter记事本时默认开启notedown插件，可以参考下面的步骤。

首先，执行下面的命令生成Jupyter记事本配置文件（如果已经生成，可以跳过）：

```Java
jupyter notebook --generate-config
```

然后，将下面这一行加入到Jupyter记事本配置文件（一般在用户主目录下的隐藏文件夹.jupyter中的jupyter_notebook_config.py）的末尾：

```Java
c.NotebookApp.contents_manager_class = 'notedown.NotedownContentsManager'
```

之后，只需要运行jupyter notebook命令即可默认开启notedown插件。

## markdownTOJupyterNotebook

在命令窗口下，输入以下简单命令即可，

```Java
notedown input.md > output.ipynb
```

## JupyterNotebookTOmarkdown

当然，把Jupyter转成markdown也很简单，参考github上aaren的说明原文贴在下面

Convert a notebook into markdown, stripping all outputs:

```Java
notedown input.ipynb --to markdown --strip > output.md
```

Convert a notebook into markdown, with output JSON intact:

```Java
notedown input.ipynb --to markdown > output_with_outputs.md
```

Strip the output cells from markdown:

```Java
notedown with_output_cells.md --to markdown --strip > no_output_cells.md
```