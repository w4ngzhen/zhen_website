---
title: macOS下由yarn与npm差异引发的Electron镜像地址读取问题
date: 2023-05-22
tags:
 - yarn
 - npm
categories:
  - 技术
---

记录macOS下由yarn与npm差异引发的Electron镜像地址读取问题

<!-- more -->

写在前面：该问题仅仅出现在Linux和macOS上，Windows上不存在该问题！

# 初始背景

最近笔者重新拾起了Electron，把最新版Electron的官方文档阅读了一遍。众所周知，Electron作为依赖在安装的时候，其二进制文件下载在国内一直以来都是问题（因为默认会从github上下载），好在现在Electron的官方文档已经写的非常详细了：[安装指导 | Electron (electronjs.org)](https://www.electronjs.org/zh/docs/latest/tutorial/installation#镜像)，只需要配置一个镜像地址到`.npmrc`中：

```
ELECTRON_MIRROR="https://npmmirror.com/mirrors/electron/"
```

> 记住这个大写的Key

笔者由于是新的机器，还没有配置改值，所以找到`.npmrc`文件的配置了上述的镜像后，便开开心心的准备进行项目搭建了。

# 问题出现

然而，当笔者准备使用**yarn**执行如下命令的时候，却出了问题：

```
yarn add -D electron
```

运行启动以后，在Electron安装的环境一直卡住了很久很久。

![010-electron-install-pending](https://static-res.zhen.wang/images/post/2023-05-22/010-electron-install-pending.gif)

咦，难道镜像配置写错了吗？仔细对比以后，没有问题。难道因为我的网络访问很慢吗？等到访问超时以后，发现一个IP地址超时了，心想国内镜像再怎么也不应该超时，盲猜镜像地址没有生效。于是乎，准备尝试对下载Electron二进制文件的过程进行debug。

# 问题排查

首先定位到`node_module/electron`包，能够看到有一段安装后脚本执行命令（`postinstall`）：

![020-node_module-electron-scripts](https://static-res.zhen.wang/images/post/2023-05-22/020-node_module-electron-scripts.png)

> 关于postinstall的详细说明：[scripts | npm Docs (npmjs.com)](https://docs.npmjs.com/cli/v6/using-npm/scripts#pre--post-scripts)

也就是说，`node_module/electron`本身npm包install完成以后，还会执行其包内的install.js。

定位进入了`node_module/electron`包下的`install.js`，该脚本内部主要逻辑是先检查Electron的二进制缓存，如果不存在缓存，则使用来自`@electron/get`包中提供的`downloadArtifact`方法从远端下载Electron二进制制品文件。

![030-installjs-flow](https://static-res.zhen.wang/images/post/2023-05-22/030-installjs-flow.png)

我们暂时先不看缓存读写的逻辑，着重了解远端下载的逻辑，所以我们进入`@electron/get`包中的`downloadArtifact`：

![040-@electron-get-npm](https://static-res.zhen.wang/images/post/2023-05-22/040-@electron-get-npm.png)

查看`@electron/get`包下的index.js内容：

![050-@electron-get-core-download-script](https://static-res.zhen.wang/images/post/2023-05-22/050-@electron-get-core-download-script.png)

前面我们提到，怀疑镜像地址没有生效导致下载超时，所以我们重点关注一下这里通过`getArtifactRemoteURL`方法得到的`url`值，

由于每一次这个包都会重新安装，我们不太好调试这个值，所以，我们做一个简单的trick：

1. 找到这个包的缓存（macOS上的路径为：`～/Library/Caches/Yarn/v6/npm-@electron-get-xxxx`）：

![060-@electron-get-cache-location](https://static-res.zhen.wang/images/post/2023-05-22/060-@electron-get-cache-location.png)

2. 找到上述indexjs代码，并添加一段日志打印：

![070-modify-cached-@electron-get-indexjs](https://static-res.zhen.wang/images/post/2023-05-22/070-modify-cached-@electron-get-indexjs.png)

3. 准备完毕以后，我们重新在demo项目下执行`yarn add -D electron`。执行以后，等到超时以后，发现控制台日志打印如下：

![080-remote-url-is-github](https://static-res.zhen.wang/images/post/2023-05-22/080-remote-url-is-github.png)

Why！？为什么这个下载的Electron二进制文件地址依然是github的？于是，我们有必要进一步查看这个URL是如何得到。

继续查看代码，这个`url`来源于`artifact-utils`中的`getArtifactRemoteURL`方法，而这个方法里面关于最终返回的`url`最重要的部分是下图所示的`base`的值：

![090-@electron-get-artifact-utils-getArtifactRemoteURL](https://static-res.zhen.wang/images/post/2023-05-22/090-@electron-get-artifact-utils-getArtifactRemoteURL.png)

而这个`base`值来源于`mirrorVar`这个方法：

![100-@electron-get-artifact-utils-mirrorVar](https://static-res.zhen.wang/images/post/2023-05-22/100-@electron-get-artifact-utils-mirrorVar.png)

根据上面代码的逻辑，name值为`"mirror"`，options未使用，defaultValue为：

`"https://github.com/electron/electron/releases/download/"`

也就是说，在后面的逻辑中，如果没有从`process.env`中找到对应的值，那么就会使用默认的github官方制品地址的值。按照代码逻辑，运行到这个方法的时候，会从`process.env`中尝试获取：

1. "NPM_CONFIG_ELECTRON_MIRROR"
2. "npm_config_electron_mirror"
3. "npm_package_config_electron_mirror"
4. "ELECTRON_MIRROR"

> 环境变量—— [配置 | npm 中文网 (nodejs.cn)](https://npm.nodejs.cn/cli/v9/using-npm/config#环境变量)
>
>任何以 `npm_config_` 开头的环境变量都将被解释为配置参数。 例如，将 `npm_config_foo=bar` 放入您的环境中会将 `foo` 配置参数设置为 `bar`。 任何未赋值的环境配置都将被赋值为 `true`。 配置值不区分大小写，因此 `NPM_CONFIG_FOO=bar` 的工作方式相同。 但是，请注意，在 [`scripts`](https://npm.nodejs.cn/cli/v9/using-npm/config#) 内部，npm 将设置自己的环境变量，并且 Node 会更喜欢那些小写版本，而不是您可能设置的任何大写版本。 详情见[此问题](https://npm.nodejs.cn/cli/v9/using-npm/config#)。
>
>请注意，您需要使用下划线而不是破折号，因此 `--allow-same-version` 将变为 `npm_config_allow_same_version=true`。
>
>此外，如果是配置在npmrc里面的配置，也会在npm/yarn启动的时候被作为环境变量放到process.env中被访问。

那我们在`.npmrc`中配置的`ELECTRON_MIRROR`，在`process.env`中变成了什么呢？通过添加日志打印，我们会看到：

![110-ProcessEnv-consolelog](https://static-res.zhen.wang/images/post/2023-05-22/110-ProcessEnv-consolelog.png)

![120-ProcessEnv-npm_config_ELECTRON_MIRROR](https://static-res.zhen.wang/images/post/2023-05-22/120-ProcessEnv-npm_config_ELECTRON_MIRROR.png)

可以看到，在`process.env`中，这个键为`"npm_config_ELECTRON_MIRROR"`（`npm_config`小写，`ELECTORN_MIRROR`大写）。我们知道，nodejs中object对象的属性值是大小写敏感的！所以，当上面的`mirrorVar`代码运行，尝试获取`process.env`中的值的时候，根本找不到了，因为没有`"NPM_CONFIG_ELECTRON_MIRROR"`、`"npm_config_electron_mirror"`、`"npm_package_config_electron_mirror"`、`"ELECTRON_MIRROR"`这些属性。

然而，如果我们使用**npm**进行安装的时候：

```
npm install -D electron
```

又能够很快安装。Why？！难道npm和yarn下的运行环境有差异吗？为了验证，我们编写一个简单的index.js代码：

```js
console.log("process.env['npm_config_electron_mirror']", process.env['npm_config_electron_mirror']);
console.log("process.env['NPM_CONFIG_ELECTRON_MIRROR']", process.env['NPM_CONFIG_ELECTRON_MIRROR']);
console.log("process.env['npm_config_ELECTRON_MIRROR']", process.env['npm_config_ELECTRON_MIRROR']);
```

然后，在package.json中添加脚本：

```diff
{
  "name": "simple-electron-main-app",
  "version": "1.0.0",
  "scripts": {
+   "start": "node index.js"
  },
  "devDependencies": {}
}
```

最后，我们分别使用yarn（`yarn start`）和npm（`npm run start`）来运行脚本：

![130-yarn-and-npm-diff](https://static-res.zhen.wang/images/post/2023-05-22/130-yarn-and-npm-diff.png)

在yarn运行上下文中，`.npmrc`中的`"ELECTRON_MIRROR"`直接拼接到了`"npm_config_"`后边，作为`process.env`的一个属性，所以你只能访问`process.env["npm_config_ELECTRON_MIRROR"]`得到值；

在npm运行山下文中，`.npmrc`中的`"ELECTRON_MIRROR"`首先被转为了小写，然后拼接到了`"npm_config_"`后边，作为了`process.env`的属性，所以你需要访问`process.env["npm_config_electron_mirror"]`来得到值。

# macOS解决方式

终于，我们能解释为什么当我们在`.npmrc`配置大写的`ELECTRON_MIRROR`的时候，使用`yarn add -D electron`安装electron的时候，二进制镜像地址没有生效了。那么，解决的办法也非常简单，两种：

1. `.npmrc`配置改为小写key：`electron_mirror="https://npmmirror.com/mirrors/electron/"`；
2. 使用`npm`上下文环境进行安装。

个人更加建议按照第一种方式配置，不然大小写敏感的坑太容易发生了。

# 关于Windows的特别说明

[process.env | Node.js API 文档 (nodejs.cn)](https://nodejs.cn/api/process/process_env.html)

在 Windows 操作系统上，环境变量不区分大小写。

```js
const { env } = require('node:process');

env.TEST = 1;
console.log(env.test);
// => 1
```

也就是说，在Windows机器上，即使`process.env`中的key为`"npm_config_ELECTRON_MIRROR"`，你也可以通过`"npm_config_electron_mirror"`或者是`"NPM_CONFIG_ELECTRON_MIRROR"`来访问这个值：

![140-windows-npmrc](https://static-res.zhen.wang/images/post/2023-05-22/140-windows-npmrc.png)

![150-process-env-Windows-output](https://static-res.zhen.wang/images/post/2023-05-22/150-process-env-Windows-output.png)

