---
title: Windows下node-gyp查找VS安装路径简单解析
date: 2021-03-27
tags:
- node-gyp
categories: 
- 技术
---

node-gyp的作用我已经不想赘述了，这里给一个我之前文章的链接：[cnblogs看这里](https://www.cnblogs.com/w4ngzhen/p/14086497.html)，[知乎看这里](https://zhuanlan.zhihu.com/p/330468774)。本文主要从源码入手，介绍node-gyp查找VisualStudio的过程

<!-- more -->

为了方便我们研究node-gyp的源码，我们随意创建一个node项目，然后我们npm install node-gyp，安装node-gyp这个包来开始我们源码探索之路吧。

```powershell
E:\Projects\node-gyp-demo> npm init
...
package name: (gyp-demo)
version: (1.0.0)
...
```

```bash
npm install node-gyp@latest // 安装最新的node-gyp
```

安装完成后，在项目/node_modules/node-gyp中，已经有了我们需要的node-gyp的js脚本代码：

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/gyp-lib-dir-position.jpg)

那么，我们应该怎么入手呢？这里需要再次提到node-gyp的处理过程，主要分为两个步骤：

1. configure

gyp首先根据C/C++源码目录下的binding.gyp文件+操作系统（Windows、macOS以及Linux）+编译构建工具（Windows下的VS，macOS以及Linux下的make）来决定生成什么样的项目结构（Windows下的sln以及vcxproj、macOS以及Linux下的make项目）这一步是*configure*配置过程，不会进行源码的编译，仅仅是生成能够作为对应平台下对应编译工具输入的项目结构。

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/node-gyp-configure-flow.jpg)

2. build

生成项目结构以后，执行build过程调用对应的编译工具完成编译任务。

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/node-gyp-build-flow.jpg)

所以，我们首先查看lib/configure.js文件，试着从源码中探索一下。进入configure.js，一下就可以看到我们期望的东西（图片顶部显示了js代码位置）：

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/gyp-configure-portal-code.jpg)

如果当前进程平台是`win32`（Windows操作系统标识），则会引入模块`find-visualstudio`。暂时停止阅读configure.js的代码，直接上我们的主角：`find-visualstudio.js`

## find-visualstudio.js

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/visualStudioFinder-class-def.jpg)

在该文件中定义了一个名为`VisualStudioFinder`的类，查找的过程就是执行创建该类的一个实例，并调用实例的一个名为`findVisualStudio`的方法。该方法被定义在该类的原型里：

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/method-the-findVisualStudio.png)

对于该函数来说，主要分为了三个步骤：

1. 对于参数msvs_version的处理
2. 对于环境变量VSINSTALLDIR的处理
3. 查找各个版本的VS

对于步骤1和2，我们暂时不进行解析，主要解析步骤3。因为绝大多数开发者就卡在这个步骤，导致安装需要原生编译的node模块失败。对于步骤3来说，我们不难看出处理的过程是优先查找本地的vs2017以及更高的版本，然后是vs2015，最后是vs2013，所以开发者Windows机器上没有安装VS或者是不在源码中支持的范围都一定会报错，提示VS找不到。我们首先解析`findVisualStudio2017OrNewer`这个函数，然后解析`findVisualStudio2015`和`findVisualStudio2013`，对于后两个，实际上最终都是相同的逻辑，后面会提到。

### findVisualStudio2017OrNewer

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/func-findVisualStudio2017OrNewer.png)

该函数的签名表示，这个函数是通过调用PowerShell脚本来获取关于VS2017或是更高版本VS的安装信息。

那么这段代码的运行情况到底如何呢？我们将该段代码单独拿出来，并将`Find-VisualStudio.cs`拷贝到运行目录下来Debug它。

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/mock-findVisualStudio2017OrNewer.png)

上图中，我模拟了node-gyp中查询VS2017以上版本的函数，通过Debug方式断点调试：

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/use-powershell.jpg)

`ps`变量值为：`C:\WINDOWS\System32\WindowsPowerShell\v1.0\powershell.exe`，即为Windows下对应的最初版本的PowerShell。cs文件不再赘述，我们也不对CSharp代码解读了。代码的最后就是执行弄得的chile_process模块中的`execFile`函数，通过传入可执行程序的完整路径已经执行参数，完成外部程序调用。

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/execFile-stdout.jpg)

而在这一步当中，如果执行出现了异常就会导致node-gyp的执行过程出现异常，进而导致需要原生编译的模块无法完成安装等。为了方便开发人员进行在Windows上查找VS2017以及以上版本，我把这段代码和CSharp代码提取出来，放在了[github仓库（w4ngzhen/node-gyp-find-vs-check）](https://github.com/w4ngzhen/node-gyp-find-vs-check)，读者如果出现了问题，可以直接下载脚本和CSharp代码进行环境的确认。

当然，有些读者的机器还是VS2015或者VS2013等版本，我们继续分析。

### findVisualStudio2015/2013

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/method-findVs2015Or2013.jpg)

通过源码可以知道，最终都调用了方法：`findOldVS`，并且还知道，nodejs的主版本大于等于9时，根本不会查找VS了。接下来我们查看方法`findOldVs`：

![](https://static-res.zhen.wang/images/post/2021-03-27-node-gyp/method-findOldVS.jpg)

对于该段代码，其实一点也不难理解，就是根据注册表上对应的键去查找的VS的安装路径（PS：好像又学习到了VS的安装路径可以从注册表里面查看呢！）对于该段代码，本人不提供demo代码帮助查询了。有兴趣的读者可以自己提取代码，模拟调用。