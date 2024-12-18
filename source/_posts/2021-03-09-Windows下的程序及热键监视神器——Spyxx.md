---
title: Windows下的程序及热键监视神器——Spyxx
date: 2021-03-09
tags:
 - Windows
categories:
  - 技术
---

# Windows下的程序及热键监视神器——Spy++

## 背景

在使用Windows的时候，偶尔会发现某些应用程序的热键不生效了；又或是桌面弹出了弹框却并不知道这个弹框来自何处。例如，本人最近使用Vim的时候，发现创建分屏后，无法使用`ctrl+w`快捷键完成切屏操作，一开始以为是Vim配置出现了问题，后来发现就连Edge浏览器的`ctrl+w`关闭页面都无法完成，仔细一想才觉得是热键被占用了，这时候就要祭出Windows下一款简单的神器Spy++。

<!-- more -->

## 如何获取Spy++

### Visual Studio

如果你下载过VS2019（其他版本应该同理），那么你可以在`VS的安装目录/Common7/Tools/`中找到一个名叫`spyxx.exe`以及`spyxx_amd64.exe`，如果你的机器是64位版本最好使用后者（PS：spyxx中的xx即为倒着的加号++）。

[Spy++ 帮助 - Visual Studio | Microsoft Docs](https://docs.microsoft.com/zh-cn/visualstudio/debugger/spy-increment-help?view=vs-2019)

### GitHub

当然，如果你没有安装VS或是其他版本的VS没有spy++，已经有热心开发者把不同版本的spy++上传到了仓库。

[GitHub - westoncampbell/SpyPlusPlus: Microsoft Spy++](https://github.com/westoncampbell/SpyPlusPlus)

你只需要clone仓库下载即可。

### 本人提供度盘

如果你是一位普通的Windows使用者，对上面的方式都不怎么会，本人已经将VS2019中的spy++传到了度盘，下载即可使用。

链接：https://pan.baidu.com/s/1CwLPltBelCJVtbyFQObA0w 
提取码：cdn7 

## 如何使用Spy++

### 热键占用检测

Spy++能够对当前的热键占用进行检测并给出占用该热键的应用程序。具体做法如下：

1、打开Spy++，选择**日志消息（log message）**

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/log-message.jpg)

2、选择**All Windows in System**

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/set-all-windows-in-system.jpg)

这一步的目的是保证拦截到当前Windows系统中的所有窗体的有关消息

3、进入message页签，筛选热键消息

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/filter-message.jpg)

4、保存配置后，会看到界面有一个空白窗口，然后按下快捷键，会发现窗口中会出现对应的热键消息

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/message.gif)

本例中，本人使用了`alt+space`快捷键呼出了uTools（另一款本人特别喜欢的工具）。通过Spy++监听到了快捷键的按下。我们可以右键该条消息，查看Properties检查该热键是由哪个程序捕获的：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/prop1.jpg)

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/prop2.jpg)

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/prop3.jpg)

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/prop4.jpg)

### 窗体所属程序检测

除了上述Spy++能够监听Windows下的事件消息外，它还能获取Windows下任意窗口的句柄信息，进而获取到该窗口所属的应用程序。

1、Spy菜单中选择`Find Window`

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/find-window.jpg)

2、拖动FinderTool到你想要识别的窗口上，松开它。剩下的请看下面的gif。

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-03-09-spyxx/how-to-find-window.gif)