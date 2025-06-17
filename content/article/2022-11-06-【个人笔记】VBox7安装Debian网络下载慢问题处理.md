---
title: 【个人笔记】VBox7安装Debian网络下载慢问题处理
date: 2022-11-06
tags:
 - note
categories:
  - 技术
  - 笔记
---

使用镜像安装Debian的过程中，会安装一些常用的软件包。但在安装软件包的阶段，默认情况下会通过网络进行下载。即使配置了国内的镜像，但是由于网络问题依然很慢。这个时候需要的在安装阶段选择从默认的DVD媒体安装。

<!-- more -->

# 准备工作

从官网下载完整镜像：[完整DVD镜像](https://cdimage.debian.org/debian-cd/current/amd64/iso-dvd/)（该步骤一定要下载完整镜像，一般名称为：`debian-11.5.0-amd64-DVD-1.iso`，且大小应该超过了3GB）。

# 安装过程

安装首选需要选择完整镜像：

![](https://res.zhen.wang/images/post/2022-11-06/010-use-complete-image.png)

安装过程中的步骤的`Configure the package manager`中，需要确保两件事：

1. 确保使用了DVD离线包

2. 手动关闭了网络连接。如此配置以后，安装的过程就一定无法从网络上下载了。

![](https://res.zhen.wang/images/post/2022-11-06/020-close-net-connection.png)

上述步骤操作完成以后，进入下一步，会提示是否使用网络镜像（`Use a network mirror？`），**此时一定要选择No不使用**。

![](https://res.zhen.wang/images/post/2022-11-06/030-dont-use-net-mirrors.png)

上述操作完成后，Debian接下来的安装就会从离线包中去安装软件。我们稍后进入系统以后再更新软件包即可。

后续按照一般的处理步骤进行即可。