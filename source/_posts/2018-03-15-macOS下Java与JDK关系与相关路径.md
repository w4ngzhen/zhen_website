---

layout: post
title: macOS下Java与JDK关系与相关路径
date: 2018-03-15
tags: 
- macOS
- Java

---

macOS下的Java与JDK的路径曾经困扰过我一段时间，今天稍有些忘记，故记下笔记，整理一下。Java与JDK的关系不在本文笔记之内，Javaer常识。

<!-- more -->

**偏好设置中的Java**

在偏好设置中的Java是浏览器的插件，仅提供Java运行环境，其目录我们通过点击该Java图标，进入Java控制面板 —— Java —— 查看，可以显示对应的目录，本人的Java已经升级到了1.8_161，目录如下：
```
/Library/Internet Plug-Ins/JavaAppletPlugin.plugin/Contents/Home/bin/java
```
通过/Library/Internet\ Plug-Ins目录名称我们也可以知道这个Java是作为网络相关（如浏览器）的Applet插件的。再次强调，这个目录下的Java**仅仅**提供Java基础运行环境，进入bin中我们可以看到**并没有**javac，也可以说明这一点。

所以，我们完全可以删除Internet  Plug-Ins文件夹下的JavaAppletPlugin.plugin这个文件夹，并且同时为了删除偏好设置中的图标，需要删除/Library/PreferencesPanes/JavaControlPanel.prefPane这个文件夹，看文件夹名不难理解。（理论上是这样的，但是没有必要删除）

**通过oracle安装的jdk**

当我们安装了oracle的jdk之后，我们在命令行中输入which java，通常会显示：
```
/usr/bin/java
```
进入/usr/bin中，查看该目录下java的详细信息，可以看到如下的信息：
```shell
$ ls -l java
lrwxr-xr-x  1 root  wheel  74 11 11 15:08 java -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/java
```
可以看到这里的java是一个软链接，链接到上述输出目录中。我们进入到**Versions**目录中，这里我们只安装了一个版本的jdk，所以只会有如下的显示信息：
```shell
drwxr-xr-x  8 root  wheel  256  1 24 22:01 A
lrwxr-xr-x  1 root  wheel    1 11 11 15:05 Current -> A
```
可以看到，Current也只是一个软链接，链接到该目录下的A文件夹（这样的好处是装了多个版本java后，可以方便地调整Current指向，使得不同版本java变成系统默认的）

我们可以看到，实际上这里面也还不是真正的jdk目录，A文件夹是安装java后系统生成的。它里面的Commands文件夹下有个java可执行文件，然而，我们可以看到这个目录下结构明显不是正常的jdk的目录结构，我们会发下这个目录下有一个java_home，使用如下命令：
```shell
./java_home -V
# output
Matching Java Virtual Machines (1):
    1.8.0_121, x86_64:	"Java SE 8"	/Library/Java/JavaVirtualMachines/jdk1.8.0_121.jdk/Contents/Home
```
上面是我机器上的输出，这个输出才是真正的jdk目录。所以在我看来，oracle安装过程应该是如下流程的：
```shell
1.首先解压完整的JDK到
/Library/Java/JavaVirtualMachines/jdk{version}.jdk/Content/Home

2.在/System/Library/Frameworks/JavaVM.framework/Versions/目录中

  1)创建对应的A或者其他版本的文件夹（猜测是不同的JDK版本依次B、C、D之类的）

  2)创建Current文件夹（已存在就忽略）并软链接到A或其他版本的文件夹上
# 从A中的Commands文件夹内容本人认为，这个文件夹就是用来协调jdk相关的资源的，比如java与javac，而这里面的java、javac一类的命令会去调用第一步中对应版本的jdk目录中的java、javac
3.将/usr/bin下的java、javac、javadoc等软链接到第二步中对应的bin上
ls -l /usr/bin/java*
lrwxr-xr-x  1 root  wheel  74 11 11 15:08 /usr/bin/java -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/java
lrwxr-xr-x  1 root  wheel  75 11 11 15:08 /usr/bin/javac -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/javac
lrwxr-xr-x  1 root  wheel  77 11 11 15:08 /usr/bin/javadoc -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/javadoc
lrwxr-xr-x  1 root  wheel  75 11 11 15:08 /usr/bin/javah -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/javah
lrwxr-xr-x  1 root  wheel  75 11 11 15:08 /usr/bin/javap -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/javap
lrwxr-xr-x  1 root  wheel  82 11 11 15:08 /usr/bin/javapackager -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/javapackager
lrwxr-xr-x  1 root  wheel  76 11 11 15:08 /usr/bin/javaws -> /System/Library/Frameworks/JavaVM.framework/Versions/Current/Commands/javaws
```
根据上面的常识，我们也完全可以卸载oracle安装的Java，同时，我们可以将真正的Java目录提取出来，放置到我们需要的地方，通过设置环境变量的方式来设置JAVA_HOME。

