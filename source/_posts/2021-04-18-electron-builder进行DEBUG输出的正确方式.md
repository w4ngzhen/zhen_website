---
title: electron-builder进行DEBUG输出的正确方式
date: 2021-04-18
tags:
 - electron
 - electron-builder
categories: 
- 技术
---

# 前言

使用Electron进行打包通常会用到electron-builder或者electron-packager两种工具。在使用electron-builder的时候，由于对机制的不熟悉，我们在打包过程中常常遇到很多环境错误，但最终只是一些简单的错误信息，难以排查问题。本文将介绍electron-builder进行DEBUG输出的正确方式来帮助排查打包过程中的各种问题。

<!-- more -->

# 本地node与electron内部的node

在对Electron进行打包的时候，需要对当前Electron项目中使用到的node原生C/C++模块进行额外的平台编译，这个过程被称为`rebuild`。有这样的一个步骤，是因为electron在运行主进程脚本的时候，是跑在了electron内部的一个nodejs环境的，electron内部的nodejs与开发机器上的nodejs并不一定是相同的。为了验证这一论点，我们进行如下的一个测试，来分别打印本地机器安装的node的版本和electon内部的node版本：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-04-18-electron-builder/test-show-version1.png)

接下来是electron主进程脚本的node版本显示（main.js）：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-04-18-electron-builder/test-show-version2.png)

接下来是分别运行`npm run show-local-node-version`和`npm run start`：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-04-18-electron-builder/show-local-and-inner-version.png)

可以看到输出确实和我们的理解是一致的，版本为11.2.0的electron内部的node版本呢是12，而我们本地机器上的node是14。这也侧面说明了为什么一般的electron应用程序会很大，因为一份electron应用程序，就有一个node的运行时。

# electron-builder调试输出正确方式

electron-builder进行打包的时候，会建议你在此之前使用`electron-builder install-app-deps`的命令。该命令的作用就是针对即将打包的electron程序对应的node版本进行原生模块的编译工作，以达到模块运行时匹配。实际上，`install-app-deps`内部依然使用了`node-gyp`相关知识，关于这一块的知识，读者可以翻阅我之前的文章《node-pre-gyp以及node-gyp的源码简单解析（以安装sqlite3为例）》来了解，这里不再赘述。本文着重介绍electron-builder如何进行debug打印，好知道打包的过程中发生了什么。

1. 首先我们准备在项目中，安装一个需要根据平台原生编译的npm包：`images`。
2. 然后在package.json中的scripts中添加一段脚本：`"installappdeps": "electron-builder install-app-deps"`
3. 最后调用命令`npm rum installappdeps`执行该脚本

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-04-18-electron-builder/images-install-and-run-install-app-deps.png)

## DEBUG环境变量

在[官方文档](https://www.electron.build/#debug)中指出了，你可以设置环境变量的方式来方便输出：

Set the `DEBUG` environment variable to debug what electron-builder is doing:

设置`DEBUG`环境变量值为字符串`electron-builder`：

```bash
DEBUG=electron-builder
```

`FPM_DEBUG` 环境变量，将会显示更多关于构建Linux平台程序的细节（除了snap和appimage）。

- **cmd（Windows CMD）**

On [Windows](https://github.com/visionmedia/debug#windows-command-prompt-notes) the environment variable is set using the set command：

在Windows CMD设置环境变量可以使用如下命令：

```cmd
set DEBUG=electron-builder
```

- **PowerShell**

PowerShell uses different syntax to set environment variables：

PowerShell使用不同的语法来设置环境变量：

```powershell
$env:DEBUG=electron-builder
```

在我们的机器上，我们同样设置该环境变量，然后执行：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-04-18-electron-builder/install-app-deps-with-DEBUG.png)

发现输出了大量的关于electron-builder的DEBUG打印，为我们了解`electron-builder install-app-deps`提供了更多的信息。

## verbose参数

然而，有的时候光是设置上述`DEBUG`环境变量还不够，因为electron-builder内部在进行rebuild操作的时候，还会以子进程方式调用`node-gyp`等工具，这些工具可不会查看上面的环境变量来输出调试信息的。我们需要在electron-builder调用node-gyp的时候，还能够打印这些工具的调试信息。熟悉node的读者可能会说，那我使用`electron-builder`的时候，传入`--verbose`怎么样？就像如下的方式：

```json
"installappdeps": "electron-builder install-app-deps --verbose"
```

不幸的是，虽然`--verbose`能被node-gyp识别，无法被electron-builder识别，。当你直接这么调用的时候，会出错：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-04-18-electron-builder/electron-cannot-recognise-verbose.png)

那么要如何解决这个问题呢？正确的做法是编写两个`scripts`：

```
  "scripts": {
	......
    "installappdeps": "electron-builder install-app-deps",
    "installappdeps-with-verbose": "npm run installappdeps --verbose"
  },
```

然后在想要进行verbose打印的时候，执行`npm rum installappdeps-with-verbose`：

![](https://cdn.jsdelivr.net/gh/w4ngzhen/CDN/images/post/2021-04-18-electron-builder/after-use-two-scripts-output.png)

可以看到，在install-app-deps的DEBUG打印前，我们还看到node给出的一些额外信息。