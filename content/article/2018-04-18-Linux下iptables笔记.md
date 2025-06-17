---
layout: post
title: Linux下iptables学习笔记
date: 2018-04-18
tags: 
- Linux
- iftables
categories: 
- 技术
---

在Centos7版本之后，防火墙应用已经由从前的iptables转变为firewall这款应用了。但是，当今绝大多数的Linux版本（特别是企业中）还是使用的6.x以下的Centos版本，所以对iptables的了解还是很有必要的。此外，需要说明的是iptables自身并不具备防火墙的功能，它需要通过内核netfilter（网络过滤器）来实现，与firewalld一样，他们的作用都是用于维护规则，而真正使用规则干活的是内核的netfilter，只不过firewalld和iptables的结构以及使用方法不一样，他们都只是一个外壳应用罢了。打个比方，就好像有一本书，同样的内容，一种是纸质的，另一种是电子的，我们翻阅它的方式不同，给它做笔记的方式不同，但是内容（内核）一样。

<!-- more -->

### iptables

**iptables概念**

iptables中的规则（rule）。在我看来，规则是iptables中进行数据包检查的基本单元。每一条规则都定义了对于数据包的条件验证，譬如网络地址的验证、端口验证、协议验证等。

iptables中的链（chain）。链是数据包传播的路径，每一条链其实就是众多规则中的一个检查清单，每一条链中可以有一条或数条规则。就像这里，我们有一个INPUT链（INPUT概念之后再作解释），数据包来到这条链时，就会根据该链中的规则进行检查，譬如源、目的地址是否符合规则；源、目的端口是否符合规则等等。当一个数据包到达一个**链**时，iptables就会从INPUT链中规则1开始检查，看该数据包是否满足规则所定义的条件——如果满足，系统就会根据该条规则所定义的方法处理该数据包；否则iptables将继续检查下一条规则2，如果该数据包不符合链中任一条规则，iptables就会根据该链预先定义的默认策略来处理数据包。
![inputchain](https://res.zhen.wang/images/post/2018-04-18-iptables/inputchain.png)

iptables中的表（table）。表是一组链的集合，在iptables中默认定义了四张表：filter、nat、mangle和raw，分别用于实现包过滤（最常用去配置的表），网络地址转换、包重构(修改)和数据跟踪处理。如下图：
![alltables](https://res.zhen.wang/images/post/2018-04-18-iptables/alltables.png)
当然，上图只是一个大致示意，并不意味着每个表中的例如想PREROUTING这样的链是独自在各个表中的，实际上表与链之间是一种交叉的关系，为什么这么说呢？这需要我们理解在iptables中传输数据包的流程（请结合下图慢慢阅读）。
（1）当一个数据包进入网卡时，它首先进入PREROUTING链，根据该链中的规则判定数据包的处理方式（ACCEPT？DROP？REJECT？），一旦通过规则检测，Linux内核根据数据包的IP地址决定是将数据包留下传入进入内部，还是转发出去。
1）如果数据包就是进入本机的（IP地址表明），它就会到达INPUT链。数据包到了INPUT链后，便开始根据INPUT链中的规则来检查数据包是否满足一系列的条件，满足之后，完全进入主机，任何进程都会收到它。同时，主机中的任何程序都可以发送数据包，发送出来的数据包会走到OUTPUT链，再根据里面的规则判定处理，最后到POSTROUTING链，再判定处理。
2）如果数据包是准备转发的，他就会到达FORWARD链，再根据FORWARD链中的规则进行检查判定决定接下来的处理。如果通过了FORWARD链，说明内核允许该数据包转发，那么数据包就会到POSTROUTING链进行最后的判断。
![flow](https://res.zhen.wang/images/post/2018-04-18-iptables/flow.png)
同时我们可以看到，这里的角度并不是以table来看的，而是以chain来看的，这就是为什么我在上面提到的，尽管table是chain的集合，但并不意味着当我们定义防火墙的时候是按照table角度来定义，而是要根据chain角度来定义。我们要根据上图来决定我们要在何处怎样处理进入的数据包。这个flow在我看来必须要非常熟悉，对之后的命令行配置也有很大的帮助，因为iptables的命令行配置十分复杂。

#### iptables命令行配置

iptables的命令格式如下：
```shell
$ iptables [-t table] COMMAND chain rules [-j action]
# table 表名
# chain 链名
# rules 规则
# action 动作
```
**table** 

表选项用于指定命令要作用于哪一个表（raw、mangle、nat、filter）

**COMMAND**
```
1、对某一条的整条链进行管理的命令
-P或–policy <链名> 定义默认策略
  设置默认策略的（设定默认门是关着的还是开着的）
  默认策略一般只有两种（DROP 关闭 | ACCEPT 打开 ）
  比如：
  iptables -P INPUT DROP 
  这条命令就是我们将INPUT链默认的策略设置为关闭，即所有的想要进入我们主机的连接全部被拒绝。
-F或-flush <链名> 清空某个表中的某条链中的所有规则
  比如：
  iptables -t filter -F INPUT # 清空filter表中的INPUT链中的所有规则
  iptables -t filter -F # 清空filter表中的所有链的所有规则
-Z或–zero <链名> 将表中数据包计数器和流量计数器归零
...

2、对某一条链的具体规则进行相关的定义
-A或—append <链名> 追加，在当前链的最后新增一个规则
-I或–insert num <链名> 在指定的num位置插入1条规则
-D或–delete num <链名> 从规则列表中删除第num条规则
-R或–replace num <链名> 替换规则列表中的第num条规则

3、查看管理命令 “-L”
-L或–list <链名> 查看iptables规则列表
附加命令
  -n：以数字的方式显示ip，它会将ip直接显示出来，如果不加-n，则会将ip反向解析成主机名。
  -v：显示详细信息
  -vv
  -vvv :越多越详细
  -x：在计数器上显示精确值，不做单位换算
  --line-numbers : 显示规则的行号
  -t nat：显示所有的关卡的信息
```
**rules**
```
-i或–in-interface <网络接口名> 指定数据包从哪个网络接口进入，如ppp0、eth0和eth1等
-o或–out-interface <网络接口名> 指定数据包从哪块网络接口输出，如ppp0、eth0和eth1等
-p或—proto协议类型 <协议类型> 指定数据包匹配的协议，如TCP、UDP和ICMP等
-s或–source <源地址或子网> 指定数据包匹配的源地址
–sport <源端口号> 指定数据包匹配的源端口号，可以使用“起始端口号:结束端口号”的格式指定一个范围的端口
-d或–destination <目标地址或子网> 指定数据包匹配的目标地址
–dport目标端口号 指定数据包匹配的目标端口号，可以使用“起始端口号:结束端口号”的格式指定一个范围的端口
```
**action**
```
# 主要的action有如下的几个
ACCEPT 接受数据包
DROP 丢弃数据包（只丢弃，不作回应，与下面的REJECT不一样）
REJECT 明确的拒绝，还向发送者返回错误信息。
SNAT 源地址转换，即改变数据包的源地址
DNAT 目标地址转换，即改变数据包的目的地址
REDIRECT：重定向，主要用于实现端口重定向
MASQUERADE IP伪装，即是常说的NAT技术
LOG 日志功能，将符合规则的数据包的相关信息记录在日志中，以便管理员的分析和排错
```
这里大致整理一下iptables命令的格式：
![fullformat](https://res.zhen.wang/images/post/2018-04-18-iptables/fullformat.png)

接下来详细的讲解一下实际使用命令的要点：

**1、定义默认策略**

什么叫默认策略？可以试想这样的场景，首先结合上面的当一个数据包到达链的时候，会根据链中定义的规则进行处理，但是很显然，我们没法对任何一种数据包的处理方式都定义在规则中，当我们的数据包进入链的时候，如果不满足所有规则的进入条件，那么我们起码要有一种默认的策略方式来处理这个数据包。这个定义默认策略就是这个作用。
```
iptables [-t表名] <-P> <链名> <动作> 
参数说明如下。
[-t表名]：指默认策略将应用于哪个表，可以使用filter、nat和mangle，如果没有指定使用哪个表，iptables就默认使用filter表。
<-P>：定义默认策略。
<链名>：指默认策略将应用于哪个链，可以使用INPUT、OUTPUT、FORWARD、PREROUTING、OUTPUT和POSTROUTING。
<动作>：处理数据包的动作，可以使用ACCEPT（接受数据包）和DROP（丢弃数据包）。
```
![defaultPolicy](https://res.zhen.wang/images/post/2018-04-18-iptables/defaultPolicy.png)
**2、增加、插入、删除和替换规则**
```
相关规则定义的格式为：
iptables [-t表名] <-A | I | R | D> 链名 [规则编号] [-i | o 网卡名称] [-p 协议类型] [-s 源IP地址 | 源子网] [--sport 源端口号] [-d 目标IP地址 | 目标子网] [--dport目标端口号] <-j动作>
参数说明如下。
[-t表名]：定义默认策略将应用于哪个表，可以使用filter、nat和mangle，如果没有指定使用哪个表，iptables就默认使用filter表。
-A：新增加一条规则，该规则将会增加到规则列表的最后一行，该参数不能使用规则编号。
-I：插入一条规则，原本该位置上的规则将会往后顺序移动，如果没有指定规则编号，则在第一条规则前插入。
-R：替换某条规则，规则被替换并不会改变顺序，必须要指定替换的规则编号。
-D：从规则列表中删除一条规则，可以输入完整规则，或直接指定规则编号加以删除。
<链名>：指定查看指定表中哪个链的规则列表，可以使用INPUT、OUTPUT、FORWARD、PREROUTING、OUTPUT和POSTROUTING。
[规则编号]：规则编号用于插入、删除和替换规则时用，编号是按照规则列表的顺序排列，规则列表中第一条规则的编号为1。
[-i | o 网卡名称]：i是指定数据包从哪块网卡进入，o是指定数据包从哪块网卡输出。网卡名称可以使用ppp0、eth0和eth1等(注意CentOS6.x与7.x网卡命名区别)。
[-p 协议类型]：可以指定规则应用的协议，包含TCP、UDP和ICMP等。
[-s 源IP地址 | 源子网]：源主机的IP地址或子网地址。
[--sport 源端口号]：数据包的IP的源端口号。
[-d目标IP地址 | 目标子网]：目标主机的IP地址或子网地址。
[--dport目标端口号]：数据包的IP的目标端口号。
<-j动作>：处理数据包的动作，各个动作的详细说明可以参考前面的说明。
```
![modifyRule](https://res.zhen.wang/images/post/2018-04-18-iptables/modifyRule.png)

**3、查看iptables规则**
```
查看iptables规则的命令格式为：
iptables [-t表名] <-L> [链名]
参数说明如下。
[-t表名]：指查看哪个表的规则列表，表名用可以使用filter、nat和mangle，如果没有指定使用哪个表，iptables就默认查看filter表的规则列表。
<-L>：查看指定表和指定链的规则列表。
[链名]：指查看指定表中哪个链的规则列表，可以使用INPUT、OUTPUT、FORWARD、PREROUTING、OUTPUT和POSTROUTING，如果不指明哪个链，则将查看某个表中所有链的规则列表。
```
由于这个命令比较简单，就不贴图了。

**4、清除规则和计数器**

在新建规则时，往往需要清除原有的、旧的规则，以免它们影响新设定的规则。如果规则比较多，一条条删除就会十分麻烦，这时可以使用iptables提供的清除规则参数达到快速删除所有的规则的目的。
```
定义参数的格式为：
iptables [-t表名] <-F | Z>
参数说明如下。
[-t表名]：指定默认策略将应用于哪个表，可以使用filter、nat和mangle，如果没有指定使用哪个表，iptables就默认使用filter表。
-F：删除指定表中所有规则。
-Z：将指定表中的数据包计数器和流量计数器归零。
```
同上，由于这个命令比较简单，就不贴图了。

当然，我们只看命令格式是枯燥的，这里我们使用一些实例来结合命令，使我们更加直观的理解。

```
禁止客户机访问某个网站或者某个IP地址
【例1】添加iptables规则禁止用户访问域名为www.xxx.com的网站。
iptables <-A | I> FORWARD -d www.xxx.com -j DROP
【例2】添加iptables规则禁止用户访问IP地址为xxx.xxx.xxx.xxx的网站。
iptables <-A | I> FORWARD -d xxx.xxx.xxx.xxx -j DROP
```
解析：首先我们可以确定是需要添加或者是插入一条我们的规则，所以使用<-A | I>；由于这里的拓扑是客户机连接我们的Linux服务器，客户机访问某一个网站，数据包到我们这里并不进入我们主机内部，而是经过转发FORWARD，所以我们要给FORWARD链插入或添加规则；由于指定了目标IP或域名，所以使用 -d 参数，又因为是禁止访问，所以 -j 之后的操作我们使用DROP。

总结：这条命令告诉防火墙，我们现在添加了一条规则在默认的filter表中的FORWARD链，其规则为如果数据包到我们目前这个Linux服务器时，是要准备转发访问IP或域名为XXX的目标（-d），那么我们禁止它访问（DROP）。

```
禁止某些客户机上网
【例1】添加iptables规则禁止IP地址为192.168.1.X的客户机上网。
iptables <-A | I> FORWARD -s 192.168.1.X -j DROP
【例2】添加iptables规则禁止192.168.1.0子网里所有的客户机上网。
iptables <-A | I> FORWARD -s 192.168.1.0/24 -j DROP
```
解析：同上的拓扑一样，客户机连接我们的Linux服务器，我们的Linux服务器是要准备做转发服务器，替我们的客户机去访问资源。所以还是对FORWARD链添加或者插入规则；但是这一次我们是要准备禁止某个IP地址的客户机上网，或者是某一子网下所有的客户机上网，也就是说，只要数据包的源IP地址或者是源子网下的我们都要禁止，所以这里采用 -s 参数。后面 -j 接DROP代表我们要丢弃来自这些源地址的数据包

总结：这条命令告诉防火墙，我们现在添加了一条规则在默认的filter表中的FORWARD链，其规则为如果客户机的数据包到我们的服务器了，如果其源地址是xxx或者是某个子网下的地址，那么我们禁止转发他。

```
禁止客户机访问某些服务
【例1】禁止192.168.1.0子网里所有的客户机使用FTP协议下载。
iptables -I FORWARD -s 192.168.1.0/24 -p tcp –dport 21 -j DROP
【例2】禁止192.168.1.0子网里所有的客户机使用Telnet协议连接远程计算机。
iptables -I FORWARD -s 192.168.1.0/24 -p tcp –dport 23 -j DROP
```
解析：以例1为例，首先我们要知道FTP协议使用的是TCP下默认21号端口。禁止192.168.1.0子网里所有的客户机使用FTP协议下载，首先还是转发的地方进行限定所以要在FORWARD链添加规则，源地址为192.168.1.0/24这个子网下的所有客户机，协议使用的是TCP，目的端口为21号。后面 -j 接DROP代表我我们要丢弃满足上述规则的数据包。
```
强制访问指定的站点
【例】强制所有的客户机访问192.168.1.x这台Web服务器。
iptables -t nat -I PREROUTING -i eth0 -p tcp –dport 80 -j DNAT –to-destination 192.168.1.x:80

禁止使用ICMP协议
【例】禁止Internet上的计算机通过ICMP协议ping到NAT服务器的ppp0接口，但允许内网的客户机通过ICMP协议ping的计算机。
iptables -I INPUT -i ppp0 -p icmp -j DROP
```

### [补充学习]NAT类型

NAT（Network Address Translation，网络地址转换）是1994年提出的。当在专用网内部的一些主机本来已经分配到了本地IP地址（即仅在本专用网内使用的专用地址），但现在又想和因特网上的主机通信（并不需要加密）时，可使用NAT方法。

这种方法需要在专用网连接到因特网的路由器上安装NAT软件。装有NAT软件的路由器叫做NAT路由器，它至少有一个有效的外部全球IP地址。这样，所有使用本地地址的主机在和外界通信时，都要在NAT路由器上将其本地地址转换成全球IP地址，才能和因特网连接。

另外，这种通过使用少量的公有IP 地址代表较多的私有IP 地址的方式，将有助于减缓可用的IP地址空间的枯竭。

静态NAT(Static NAT)
静态NAT设置起来最为简单和最容易实现的一种，内部网络中的每个主机都被永久映射成外部网络中的某个合法的地址。

动态地址NAT(Pooled NAT)
动态地址NAT是在外部网络中定义了一系列的合法地址，采用动态分配的方法映射到内部网络。
动态地址NAT只是转换IP地址，它为每一个内部的IP地址分配一个临时的外部IP地址，主要应用于拨号，对于频繁的远程联接也可以采用动态NAT。

网络地址端口转换NAPT（Port－Level NAT）
NAPT是把内部地址映射到外部网络的一个IP地址的不同端口上。
最熟悉的一种转换方式。NAPT普遍应用于接入设备中，它可以将中小型的网络隐藏在一个合法的IP地址后面。NAPT与动态地址NAT不同，它将内部连接映射到外部网络中的一个单独的IP地址上，同时在该地址上加上一个由NAT设备选定的TCP端口号。

### [了解]在Centos7.x上卸载firewalld，安装iptables

由于CentOS7.x开始，防火墙应用已经不实用iptables，而是使用firewalld了。其应用的管理理念与iptables有着很大的区别。鉴于你确实可能不想用firewalld而是习惯使用iptables，这里提供相关的操作方式来卸载firewalld安装iptables。

#### 安装iptables、iptables-service
```shell
# 先检查是否安装了iptables
$ systemctl status iptables.service

# 安装iptables
$ yum install -y iptables
# 升级iptables（安装的最新版本则不需要）
$ yum update iptables 
# 安装iptables-services
$ yum install iptables-services # service!s!
```
#### 禁用/停止自带的firewalld服务
```shell
#停止firewalld服务
$ systemctl stop firewalld
#禁用firewalld服务
$ systemctl mask firewalld
```
#### 设置现有的规则
```
#查看iptables现有规则
$ iptables -L -n
```
![iptables-L-n](https://res.zhen.wang/images/post/2018-04-18-iptables/iptables-L-n.png)
可以看到显示的INPUT、FORWARD、OUTPUT上没有任何规则配置且默认策略均为ACCEPT
```shell
# 务必先配置INPUT链的默认规则为ACCEPT，这样一来，避免误配置导致我们无法进入
$ iptables -P INPUT ACCEPT
#清空所有默认规则
$ iptables -F
#清空所有自定义规则
$ iptables -X
#所有计数器归0
$ iptables -Z
#允许来自于lo接口的数据包(本地访问)
$ iptables -A INPUT -i lo -j ACCEPT
```
接下来定义特别的入站规则
```shell
# 开放22端口
$ iptables -A INPUT -p tcp --dport 22 -j ACCEPT
# 开放21端口(FTP)
$ iptables -A INPUT -p tcp --dport 21 -j ACCEPT
# 开放80端口(HTTP)
$ iptables -A INPUT -p tcp --dport 80 -j ACCEPT
# 开放443端口(HTTPS)
$ iptables -A INPUT -p tcp --dport 443 -j ACCEPT
# 允许ping
$ iptables -A INPUT -p icmp --icmp-type 8 -j ACCEPT
# 允许接受本机请求之后的返回数据 RELATED,是为FTP设置的
$ iptables -A INPUT -m state --state  RELATED,ESTABLISHED -j ACCEPT
```
在上面的入站规则配置好以后，接下来我们可以将INPUT默认策略转为DROP（拒绝）
```shell
# 其他入站一律丢弃
$ iptables -P INPUT DROP
# 所有出站一律允许
$ iptables -P OUTPUT ACCEPT
# 所有转发一律丢弃
$ iptables -P FORWARD DROP
```
最终我们可以看到我们目前定义的配置表：
![finalConfig](https://res.zhen.wang/images/post/2018-04-18-iptables/finalConfig.png)

#### 保存规则设定、开启iptables服务
```shell
# 保存上述规则
$ service iptables save

# 注册iptables服务
# 相当于以前的chkconfig iptables on
$ systemctl enable iptables.service
# 开启服务
$ systemctl start iptables.service
# 查看状态
$ systemctl status iptables.service
```
![enableAndStart](https://res.zhen.wang/images/post/2018-04-18-iptables/enableAndStart.png)
防火墙配置完成！

