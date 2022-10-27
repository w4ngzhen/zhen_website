---
title: 【个人笔记】基于VirtualBox7的Debian11基础环境搭建
date: 2022-10-27
tags:
 - debian11
 - virtualbox7
---

本文主要是对在最新的VirtualBox7上搭建Debian11的笔记记录，方便后续个人回顾，同时搭配对配置的浅析。

<!-- more -->

# sudoers配置

非root用户想要使用sudo命令，需要两个条件：

1. 系统安装了sudo命令；
2. 将该用户添加到了`/etc/sudoers`中；

对于第一点检查是否安装sudo命令，可以使用`su -`命令切换为root用户，尝试使用sudo命令，同时检查`/etc/sudoers`这个文件是否存在。如果root使用sudo命令都提示`command not found`，且`/etc/sudoers`这个文件也不存在，大概率系统就没有sudo命令，进而几乎所有非root用户都无法使用sudo命令。此时，我们就只能使用root用户来安装sudo命令了（`apt-get install sudo`），安装好以后会自动创建`/etc/sudoers`文件。

>PS：如果出现尝试安装sudo的时候，也提示无法找到sudo这个包，大概率你没有更新APT，所以先使用命令`apt-get update`更新下仓库信息。这个时候如果apt-get更新很慢，大概率你没有配置合适的镜像，导致网络很慢。此时，在root用户下，按照下面“APT源配置”方式去配置下APT源，配置好以后再重新尝试安装sudo命令。

当安装好了sudo以及`/etc/sudoers`也创建好以后，就可以在root用户下去修改`/etc/sudoers`，把非root用户添加到sudoers里面。

```
# /etc/sudoers
... ...
root ALL=(ALL:ALL) ALL
用户名 ALL=(ALL:ALL) ALL # 权限含义自行了解
...
```

将非root用户添加到sudoers以后，如果此时你还在root用户下，则可以使用`exit`命令退出当前登陆的root用户，回到你一开始的自己用户会话中，然后使用`sudo 命令`来临时提权执行命令了。

# APT源配置

/etc/apt/sources.list

```
# 默认注释了源码镜像以提高 apt update 速度，如有需要可自行取消注释
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free
deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-updates main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye-backports main contrib non-free

deb https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
# deb-src https://mirrors.tuna.tsinghua.edu.cn/debian-security bullseye-security main contrib non-free
```

## 配置解析

`deb https://mirrors.tuna.tsinghua.edu.cn/debian/ bullseye main contrib non-free`

形如如下格式：

`deb http://site.example.com/debian/ distribution component1 component2 component3`

**Archive type**

每行的第一个单词**，deb** 或 **deb-src**，表示存档的类型。**Deb** 表示存档包含二进制包 （[deb](https://wiki.debian.org/deb)），即我们通常使用的预编译包。**Deb-src** 表示[源码包](https://wiki.debian.org/SourcePackage)，它们是原始程序[源](https://wiki.debian.org/source)码加上 Debian 控制文件 （.[dsc](https://wiki.debian.org/dsc)） 和差异.gz包含打包程序所需的更改。

**Repository URL**

该行的下一个条目是指向要从中下载包的[存储库](https://wiki.debian.org/DebianRepository)的 URL。Debian 存储库镜像的主要列表位于[此处](https://www.debian.org/mirror/list)。例如，咱们使用的是清华的镜像，进入该页面会看到如下的FTP界面：

**Distribution**

发行版本代号 or 发行版代号加`-updates`、`-backports`或`-security`后缀。例如当前（2022-10-25）Debian最新发行版本为11，代号bullseye。

在上述配置中，我们使用了：`bullseys`、`bullseye-updates`、`bullseye-backports`以及`bullseye-security`（注意，Distribution为`bullseye-security`时候，URL中的路径对应为`debian-security`，而不是`debian`）。

**Component**

- [main](http://www.debian.org/doc/debian-policy/ch-archive#s-main) 由[符合 DFSG](http://www.debian.org/social_contract#guidelines) 标准的软件包组成，这些软件包不依赖于该领域之外的软件来运行。这些软件包被认为是 Debian 发行版中唯一的一部分。
- [contrib](http://www.debian.org/doc/debian-policy/ch-archive#s-contrib) 软件包包含 DFSG 兼容的软件，但具有不在 main 中的依赖关系（可能打包为非自由的 Debian）。
- [非自由](http://www.debian.org/doc/debian-policy/ch-archive#s-non-free)软件包含不符合 DFSG 的软件。

![010-sources-list-path](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-27/010-sources-list-path.png)

当我们通过apt/apt-get进行软件安装的时候，如果出现某些软件找不到，一方面可能软件包名称不对；另一方面，可能是在配置sources.list的时候，main、contrib或者non-free遗漏了配置，导致不会进入这些目录下进行搜索。

# 基础库

## gcc/g++

build-essential包包含了在Linux下开发C/C++基础开发工具与库：

gcc/g++/make/libc6-dev/dpkg-dev等

```shell
sudo apt-get install build-essential
```

## git

```shell
sudo apt-get install git
```

## CMake

1）apt安装

```shell
sudo apt-get cmake
```

2）如果是源码方式安装，则还可以验证上述的安装gcc/g++等基础开发环境是否OK。

下载源码文件：[Download | CMake](https://cmake.org/download/)

进入cmake源码目录，执行配置、构建以及安装（[Installing | CMake](https://cmake.org/install/)）：

```
./bootstrap
make
make install
```

安装完成后，可以调用`cmake --version`查看版本，检查是否编译并安装成功。

## X11

```
sudo apt-get install libx11-dev
```

## OpenSSL

```
sudo apt-get install libssl-dev
```

## OpenGL（OpenGL的Library、Utilities以及ToolKit）

```
sudo apt-get install libgl1-mesa-dev
sudo apt-get install libglu1-mesa-dev
sudo apt-get install freeglut3-dev	
```

## dkms

DKMS全称是DynamicKernel ModuleSupport，它可以帮我们维护内核外的驱动程序，在内核版本变动之后可以自动重新生成新的模块。当需要开发Linux内核模块的时候，需要该包。同时，安装VBox增强工具时候，会编译内核文件，故也需要安装。

```
sudo apt-get install dkms
```

## Linux内核相关包

当前版本Linux内核的头文件。当需要开发Linux内核模块的时候，需要该包。同时，安装VBox增强工具时候，会编译内核文件，故也需要安装。

```
sudo apt-get install linux-headers-$(uname -r)
```

# 安装VBox增强工具

## 方式一：直接利用VBox的`Insert Guest Additions CD image`

![020-addition-insert-iso](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-27/020-addition-insert-iso.png)

使用该方式，VBox会自动查找对应的增强工具包CD镜像，并挂在到Debian系统。稍等片刻后，我们可以从桌面上看到一个名为：`VBOX_GAs_x.x.x`的CD镜像，双击打开后可以直接看到里面的内容：

![030-VBox-GAs-cd-image-content](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-27/030-VBox-GAs-cd-image-content.png)

我们首先把其中的`VBoxLinuxAdditions.sh`拷贝到桌面，或者任意路径。然后命令界面执行：

```shell
sudo sh VBoxLinuxAdditions.run
```

## 方式二：命令行处理

如果在纯命令行界面，依然是通过上述的`Insert Guest Additions CD image`操作。操作完成后，此时相当于我们插入一张包含有增强工具软件的CD光盘。此时，我们需要将“光驱”设备挂在到我们的一个目录上，譬如我们创建一个`/media/vbox-cd-content/`：

```shell
sudo mkdir /media/vbox-cd-content/
```

将`/dev/cdrom`挂载到上述目录：

```shell
sudo mount /dev/cdrom /media/vbox-cd-content
```

进入`/media/vbox-cd-content`，将里面的所有东西都复制到外部目录（譬如创建一个`~/VboxAdditions/`）：

```shell
mkdir ~/VboxAdditions/
sudo cp -R /media/vbox-cd-content/* ~/VBoxAdditions/
```

复制完成后，我们可以先卸载CD挂载目录：

```shell
sudo umount /media/vbox-cd-content/
# 注意点1：umount 不是 unmount！
# 注意点2：如果提示设备Busy，说明这个目录还占用着，比如你还在这个目录里面，可以先退到其他目录。
```

卸载目录后，我们在看里面是没有内容了。

进入我们拷贝好内容的`~/VBoxAdditions/`，执行命令，等待完成即可：

```shell
sudo sh VBoxLinuxAdditions.run
```

完成后，重启：

```shell
sudo reboot
```

重启完成后，我们就可以配置一个共享目录来检查是否安装成功。共享目录的配置，网上有很多，这里不再赘述。

![040-config-shared-dir](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-27/040-config-shared-dir.png)

![050-set-shared-dir](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-27/050-set-shared-dir.png)

这里没有选择Auto-mount自动挂载，那么需要手动挂载。这里点击OK以后，我们进入Linux，可以手动创建一个目录：`/media/shared-dir`目录，然后通过调用挂载的命令：

```shell
sudo mkdir /media/shared-dif/
sudo mount -t vboxsf SharedDir /media/shared-dir
```

解读下这个命令。首先，`-t vboxsf`必须要有，该参数是在完成VBox的增强功能安装以后才能使用；其次，`SharedDir`需要和上述配置共享文件夹配置里面的`Folder Name`保持一致，代表我要将这个外部对应的目录挂到Linux系统中；最后的`/media/shared-dir/`就是要挂到当前Linux系统中的那个目录。

完成挂载后，我们可以在宿主系统创建一个文件夹来验证：

![060-host-os-new-file](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-27/060-host-os-new-file.png)

![070-shared-file-in-linux](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2022-10-27/070-shared-file-in-linux.png)

可以看到已经将宿主系统里面的hello.txt在Linux读取出来了。
