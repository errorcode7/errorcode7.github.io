# SPDX

SPDX(Software Package Data Exchange)是一种文件格式，用于记录有关分发给定计算机软件的软件许可证的信息。SPDX 是由 SPDX 工作组编写的，该工作组代表了 20 多个不同的组织，由 Linux 基金会所支持。

简单的说，就是简化授权说明。

## reuse
该工具用于在文件头添加SPDX格式的版权与license。

官方用例，init初始reuse，download下载license，addheader添加license到文件头，`addheader`会根据文件类型自动添加注释。reuse检测。 

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://download.fsfe.org/videos/reuse/screencasts/reuse-tool.gif">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">官方用例</div>
</center>
可用开源协议：<https://spdx.org/licenses/> ，选一个[GPL-3.0-or-later](https://spdx.org/licenses/GPL-3.0-or-later.html)。

```shell
#新建项目
git clone -b noncompliant https://github.com/fsfe/reuse-example.git

#初始化,生成文件`.reuse/dep5`
pip3 install reuse
cd reuse-example && reuse init 

#下载协议放到LICENSES/GPL-3.0-or-later.txt，reuse download --all 下载所有
reuse download GPL-3.0-or-later CC0-1.0

#添加所有文件，不想添加的文件加入.gitignore
reuse addheader --copyright="Jane Doe <jane@example.com>" --license="GPL-3.0-or-later" src/main.c Makefile README.md

reuse addheader --copyright="Jane Doe <jane@example.com>" --license="GPL-3.0-or-later" --force-dot-license img/cat.jpg img/dog.jpg

reuse addheader --copyright="Jane Doe <jane@example.com>" --license="CC0-1.0" .gitignore

#检测合规，会列举未授权的文件，最终要看到`Congratulations!`才行。
reuse lint
```

## 开源协议分类

<center>
    <img style="border-radius: 0.3125em;
    box-shadow: 0 2px 4px 0 rgba(34,36,38,.12),0 2px 10px 0 rgba(34,36,38,.08);" 
    src="https://picd.zhimg.com/80/253a7b1819e2af555ed0a7e0f11a0b59_720w.webp?source=1940ef5c">
    <br>
    <div style="color:orange; border-bottom: 1px solid #d9d9d9;
    display: inline-block;
    color: #999;
    padding: 2px;">开源协议分类</div>
</center>


参考：https://reuse.software/tutorial/

