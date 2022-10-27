---

layout: post
title: 在macOS上通过虚拟机搭建基础CentOS7系统环境
date: 2018-03-06
tags: 
- Linux

---
尽管从Mac的Terminal可以看出，macOS与UNIX、Linux或多或少都有血缘关系（shell、bash等），但是在mac进行Linux开发，或者把macOS直接当作Linux来使用依然是说不过去的，这其中包括一些命令行的使用，一些基本的文件夹体系等（如，在Linux上的/home目录与在macOS下的/Users）不一致。如果想要在macOS上进行Linux的学习，或者进行Linux开发，最完美的方案自然是安装虚拟机。

<!-- more -->

### 虚拟机的选择

mac平台上的主流虚拟机有VMWare Fusion、Parallels Desktop、VrtualBox。前两者均是付费产品，价格虽不是很贵（参考JetBrains、Adobe系列），但是对于只是在Linux上进行学习或者是进行一些简单的开发颇有些大材小用了。所以，本次教程选择免费的VirtualBox进行搭建。

地址：[VirtualBox](https://www.virtualbox.org/)

下载好以后，打开dmg双击pkg正常安装即可。

* **注意**

  VirtualBox在安装的时候需要安装内核扩展（Kernel extenstion）由于macOS 10.13 High Sierra的新安全特性会阻止外部内核扩展的安装，所以安装总是会被系统拦截，出现安装失败的情况，这时候打开“系统偏好设置 - 安全性与隐私”在界面下方会出现提示“来自Oracle America, Inc....”，点击“允许”再次安装即可。

* **点击“允许”没反应？**

  本人安装VBox还有其他软件曾经出现过点击“允许”没有反应的情况，可能是macOS存在的一个BUG。此时首先关闭“安全性与隐私”，进入“系统偏好设置 - 键盘 - 快捷键”，选择“所有控制”，然后在此进入“安全性与隐私”，按tab键移动光标焦点到“允许”按钮处按空格键即可点击成功。

### Linux的选择

本人选择CentOS 7作为本次的教程的Linux版本。

官方地址：[CentOS](https://www.centos.org/)

在download界面我们可以看到三种iso镜像：
* DVD.iso 可以用安装程序安装的所有安装包。
* Everything.iso 包含centos7的一套完整的软件包，可以用来安装系统或者填充本地镜像。
* Minimal.iso 最小安装盘，只有必要的软件，自带的软件最少。（**只有命令行界面**）

这里我们选择minimal，首先因为它是最精炼的一个，其次，一步一步定制我们的系统才能够更好的熟悉Linux系统。
下载好以后，打开VirtualBox安装CentOS系统。

**开始安装**

打开VirtualBox，点击“新建” - “专家模式”
按照如下的方式配置
![create-v.png](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/create-v.png)
内存大小会使用到你mac的运行时内存，请合理分配，这里我们分配1G内存
点击创建，进入下一步：
![create-v2.png](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/create-v2.png)
创建完成之后界面如下（CentOS7是我们创建的）：
![create-v3.png](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/create-v3.png)
由于我们只是为虚拟机分配了相关的空间以及配置，但是并没有将系统挂载，点击“启动”会提示我们选择镜像文件，点击右侧小图标来选择镜像文件：
![chooseimg-detail](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/chooseimg-detail.png)
选择完毕，准备启动：
![chooseimg](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/chooseimg.png)
接下来点击“start”，经过短暂的等待，我们进入到系统安装界面，我们点击该界面，直接进入虚拟机内部（mac上退出虚拟机操控请按 command 键），此时可以操作方向键来选择我们接下来的步骤，这里我们选择“Install CentOS 7”并回车：
![installUI](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/installUI.png)
接下来我们会进入到一个鼠标操控极差的界面来安装我们的系统，这里我们选择中文语言来进行系统安装：
![chooseCN](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/chooseEn.png)
选择过语言之后，会看到“安装位置”有红色警告提示是自动分区，这里CentOS已经为我们选择好了“自动分区”
![installCentos](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/installCentos.png)
此时只需要点击进去然后在左上角点击“完成”即可，只做一个确认即可：
![partition](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/partition.png)
回到主界面我们可以点击“开始安装”进行安装了。这里还要说一下，在安装过程中，就像下面的图示，系统是一直在安装的，但是此时系统中只有一个没有设置密码的root用户，就连管理员用户也没有，这里**需要**你设置root用户密码，可以不配置新的管理员用户：
![installing](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/installing.png)
密码设置短一点没有事儿，毕竟是练习机：
![setpasswd](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/setpasswd.png)
然后完成配置
![completeInstall](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/completeInstall.png)
之后等待最后的收尾工作结束，点击“重启”，进入命令行界面（选择的minimal，只有命令行界面），选择第一个系统，登陆root用户：
![chooseFirst](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/chooseFirst.png)
![login](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/login.png)
至此，虚拟机下Linux基本配置完成。

### 配置双网卡实现虚拟机 与主机（mac）与外网通信
在上面的配置中，我们已经搭建好了一个最基本的，不具备网络通信的CentOS7操作系统，但是不能与外网通信，我们不能通过在线安装我们需要的软件，这样的单机操作系统用处是不大的。接下来我将介绍如何通过双网卡的方式来实现实现虚拟机 与主机（mac）与外网通信。

#### 配置仅仅与主机通信的仅主机模式网卡

以下配置方式是不能与外网通信的，仅仅与主机通信，但好处在于我们可以使用静态IP地址，避免IP地址的变动。如果你没有这方面的需求，可以直接进入下一节。

无论是哪一台计算机，要进行网络通信，都必须要至少有一块网卡来进行通信，当然，我们可以通过软件来虚拟出网卡，让这个虚拟的网卡获取IP来通信，更多的细节需要你有计算机网络的相关知识来支撑，这里不多提。这里既然我们需要让虚拟机中的系统来与我们的主机（mac）来通信，自然而然，需要让我们mac有一块网卡，虚拟机有一块网卡，让它们处于同一网段，这样一来，我们自然就实现了虚拟机与主机（mac）之间的通信。主机上怎么创建一块虚拟网卡呢？这里VirtualBox就可以帮我们实现。

点击主界面上的“全局工具” - “主机网络管理器”，进入后点击“创建”（**注意**：这里可能会存在显示BUG，导致你点击创建之后没反应，**请不要**连续点击创建，点了一次之后切换一下画面），之后你就会在主机上（mac）创建一块用于和VirtualBox中的虚拟机进行通信的虚拟网卡：
![createHostNetworkUI](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/createHostNetworkUI.png)
这里VBox自动为我们在mac主机上创建了一块名称为vboxnet0的网卡
![createHostNetwork](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/createHostNetwork.png)
我们进入mac终端，显示当前的所有的网卡信息也能看到这块网卡在我们的mac主机上已经创建成功了：
```shell
$ ifconfig
...
vboxnet0: flags=8943<UP,BROADCAST,RUNNING,PROMISC,SIMPLEX,MULTICAST> mtu 1500
	ether 0a:00:27:00:00:00
	inet 192.168.56.1 netmask 0xffffff00 broadcast 192.168.56.255
# 我们看到这块网卡已经有一个IP地址为192.168.56.1
# 要让我们的虚拟机与我们的主机通信，只要虚拟机中有一块我们主机在同一网段的网卡就行了
...
```
**注意：请明确以上操作是为我们的主机mac，不是为我们的虚拟机创建网卡：）**

接下来，我们可以通过VirtualBox给CentOS配置一块网卡，用于我们的虚拟机中的操作系统与我们的主机（mac）之间的通信，所以首先我们要配置一块“仅主机（Host-Only）网络”：
![host-only](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/host-only.png)
我们可以看到，VirtualBox已经很智能的为我们选择了我们刚刚为主机mac创建的网卡。点击“OK”之后，我们虚拟机与主机之间的通信网卡配置完成（**注意**：仅仅是虚拟网卡配置好了，IP地址什么还没有配置）。接下来我们登录我们的虚拟机查看网卡的配置情况。
进入系统之后，我们使用命令ip addr来查看CentOS下的网卡配置情况（默认是没有ifconfig命令的，我们之后再装）
![ip_addr](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/ip_addr.png)
注意查看此处第二块网卡，我们可以看到它的硬件地址是08:00...:3d，和我们上面的配置Host-Only处显示的硬件MAC地址是一样的，同时请记住这块网卡的名称。

**配置网络**

在CentOS7中，我们进入/etc/sysconfig/network-script目录，显示当前目录下的文件：
```shell
$ cd /etc/sysconfig/network-script
$ ls
# 在众多的配置文件中，我们应该是能够找到和我们上面ip addr命令中显示的那块网卡名称一样的带有“ifcfg-”前缀的配置文件
...
ifcfg-enp0s3
...
```
**注意**：在CentOS6.x及一下，网卡的命名规则和7不一样，相关的差别请自行搜索。

接下来我们需要配置网卡的具体信息，由于我们选择的是minimal版本的CentOS，所以使用vi来打开编辑它：
```shell
$ vi ifcfg-enp0s3
# 同时为了配合上面的我们主机的IP网段，我们按照如下的方式来配置

# 类型=以太网
TYPE=Ethernet(默认)
# 设备名
DEVICE=enp0s3(默认)

#BOOTPROTO=dhcp(dhcp为自动分配ip地址,我们把他注释了，在下面另外加)
BOOTPROTO=static(新添加)
# IP地址
IPADDR=192.168.56.66(新添加)
# 子网掩码
NETMASK=255.255.255.0(新添加)

ONBOOT=yes(默认为no,修改为yes意为每次启动之后自动启动该网卡)
# 以上就是一个最精简的网卡配置文件
```
![ifcfg](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/ifcfg.png)
接下来reboot重启CentOS，重启之后通过ip addr再次查看我们网卡信息，也可以看到其网卡已经配置好了IP地址：
![ipaddrNew](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/ipaddrNew.png)
最后来进行测试，由于mac本身的安全防火墙是打开的，所以我们通过虚拟机（ip=192.168.56.66）去ping我们mac主机（ip=192.168.56.1）是ping不通的，我们只有在mac终端下去ping我们的虚拟机，发现已经ping通：
![ping](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/ping.png)

#### 配置既能与主机通信同时还能与外网通信的桥接模式

配置第二块网卡的原理十分简单，就是我们将我们主机的一块能够上网的网卡作为桥接网卡（可以理解为抽象成一台路由器上的网卡），这样，首先毫无疑问，我们的既然都是我们主机上的网卡了，自然主机与虚拟机之间是能够通信的，同时，我们的虚拟机还能够通过这块桥接网卡来与外界通信。

以上的简略了解以后，我们首先可以知道，我们可以不再需要Host-Only这样的主机模式了。所以我们不再选择主机模式。

选择桥接模式之前，我们首先查看我们当前能够使用的桥接的网络，在本次实验中，本人mac使用的是无线网卡en0连接的网络本地的局域网：
```shell
$ ifconfig
...
en0: flags=8863<UP,BROADCAST,SMART,RUNNING,SIMPLEX,MULTICAST> mtu 1500
	...
	inet 172.20.10.12 netmask 0xfffffff0 broadcast 172.20.10.15
	nd6 options=201<PERFORMNUD,DAD>
	media: autoselect
	status: active
...
```
在此基础之上，我们设置虚拟机的网络为桥接模式，取消仅主机模式网络，并且选择我们的无线网卡：
![nohostonly](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/nohostonly.png)
![en0](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/en0.png)
接下来进入系统，验证网卡是否配置成功，之后在测试ping外网：
![ipaddrBridgeAndPing](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/ipaddrBridgeAndPing.png)
再使用主机ping我们的虚拟机，发现没有问题
![pingVM](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/pingVM.png)
综上，我们的虚拟机 + 网络完全配置完成！

### 安装VirtaulBox增强工具包

VirtualBox的增强工具包通常不需要我们再去下载，它通常伴随VirtualBox下载好了。一般会存放在对应应用的根目录下（在Windows中就是安装根目录），在mac中我们知道应用的根目录就是当我们安装的应用的包内容中：找到VitualBox应用，右键，“显示包内容”，“Contents”，“MacOS”，可以看到一个名为VBoxGuestAddition.iso镜像文件：
![additioniso](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/additioniso.png)
接下来，就需要我们将这个iso镜像文件挂载到我们的Linux系统中。**注意，一定要明确这个增强工具是给我们虚拟机中的Linux系统安装的**。

#### 为CentOS安装必要的开发库工具库

进入Linux中，在前面安装配置好网络的基础上，我们使用yum安装如下待会增强功能包需要用到的工具库：
```shell
$ yum update
$ yum install gcc gcc-c++ make kernel-headers kernel-devel bzip2
# 务必注意先安装gcc、gcc-c++、make，这三个工具是后面编译的基础工具
```
安装完成以后，reboot重启。**(务必要重启，为了更新kernel内核系统信息)**

#### 挂载增强功能镜像文件

这里必须要提到Linux中的一个概念：**挂载（mount）**。根文件系统之外的其他文件要想能够被访问，都必须通过“关联”至根文件系统上的某个目录来实现，此关联操作即为“挂载”，此目录即为“挂载点”,解除此关联关系的过程称之为“卸载”。实际上，我们Windows上同样也有挂载，比如U盘的插入并读写，新添加的硬盘并读写等等。这一切由于在Windows上已经有对应的驱动程序帮助我们自动的进行了，所以我们不太熟悉这个过程。而在Linux上，特别是在命令行界面下，我们要去读取一些系统以外的文件（就比如上面的U盘，光驱等），就必须要手动的去做这样一个操作。

挂在的过程我们需要明确最主要的两点：设备以及挂载点。设备其实就是Linux操作系统中/dev/目录下显示那一些设备。挂载点其实就是Linux文件系统中的我们可以自定义的文件夹，在到时候使用挂在命令的时候，将外部文件系统与Linux内部文件系统关联起来的入口。可能还有些模糊，下面安装增强功能包会进一步根据实际来解释的。

在上面我们已经安装好了开发工具并重启之后，我们首先点击VirtualBox上的“devices”选项，找到“Install Guest Addition CD image”：
![installdevices](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/installdevices.png)
找到这一栏以后，我们点击它，现在是没有任何反应的，但是这里VirtualBox已经为我们将CD镜像“连接”到了Linux中了，就好像我们普通电脑装入光驱一个道理。

补充：VirtualBox上的devices其实就是我们的外设接口一样。我们可以看到devices显示的Optical Drivers、Audio、Network以及USB，实际上就是为我们虚拟化的接口。假设我们想要将U盘数据传到Linux中，就可以现在电脑上插入U盘，然后在USB中找到我们这个插入的U盘对应的驱动选项，VirtualBox就会为我们读入U盘，之后在Linux中我们把对应的设备挂载到某一文件夹就可以读取了。

继续上面的安装。接下来我们创建一个挂载点，挂载点通常就是创建一个临时的文件夹：
```shell
$ mkdir tempdir
```
创建完成以后，我们将cd设备挂载到这个文件夹上：
```shell
$ mount -t auto /dev/cdrom ./tempdir
# 完成挂载以后会打印设备的访问权限
```
接下来，我们就可以进入刚刚创建的tempdir，我们可以看到这个文件中，出现了一些新的文件，这些文件其实就是刚刚增强功能包中的文件：
![tempdir](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/tempdir.png)
看到这里，读者应该就能对挂载有基本更形象的认识了吧。其实就是将外部文件通过某一个我们内部创建的文件夹（挂载点）连接到我们Linux内部文件系统。
这个增强包中包含了Linux、Windows以及macOS系统的增强工具包。我们是Linux系统，所以运行VBoxLinuxAdditions.sh即可：
```shell
$ sh ./VBoxLinuxAdditions.sh
```
![building](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/building.png)
完成以后，我们会发现系统的分辨率发生了变化，并且通过输出也知道增强工具安装成功了。

最后请类似于像Windows上弹出U盘一样，卸载挂载点
```shell
$ umount tempdir
```

### 设置共享文件夹

VirtualBox为我们设计了“共享文件夹”，方便Linux与主机之间的文件传输。这一功能需要增强功能包安装完成了才可以使用。

这一步其实很简单，同样是在VirtualBox上的devices选项中，我们点击“Shared Folders” - “Shared Folders Settings”，点击新建共享文件夹按钮：
![newsf](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/newsf.png)
在弹出来的界面设置在主机上的共享文件夹，这里我在桌面上创建了一个名为“sharefolder”的文件夹，并选择了它，同时为了方便，文件名我设置为了“sf”。下方第一条“只读”按需勾选（以后可以修改），“Auto-mount”自动挂载选上，方便系统中自动挂载读取，“Make Permanet”永久化一般也选上，以后一直用这个文件夹与虚拟机中的Linux进行文件共享。

回到Linux中，我们为了方便文件的管理，创建一个名为myshare的文件夹作为挂载点。
```shell
$ mkdir myshare
```
接下来使用如下的命令将主机上的共享文件夹挂载进来：
```shell
$ mount -t vboxsf sf ./myshare
```
于是，当我们在主机上往共享文件夹中操作文件的时候，刷新Linux中的文件夹，就可以实时的看到（反之亦然）：
![sharefile](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/sharefile.png)
![deleteshare](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2018-03-06-Linux/deleteshare.png)

在此基础上，整个使用VirtualBox进行Linux虚拟机的搭建工作完成！
