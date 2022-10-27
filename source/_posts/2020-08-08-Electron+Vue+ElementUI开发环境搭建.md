---

title: Electron+Vue+ElementUI开发环境搭建
date: 2020-08-08

---

本文将从零开始，进行Electron+Vue+ElementUI的开发环境搭建

<!-- more -->

## Node环境搭建

本文假定你完成了nodejs的环境基础搭建：
镜像配置（暂时只配置node包镜像源，部分包的二进制镜像源后续讨论）、全局以及缓存路径配置，全局路径加入到了环境变量

```shell
 $ node -v
 v12.18.1
 $ npm -v
 6.14.5
 $ yarn -v
 1.22.4
 
 $ npm list -g --depth 0
 +-- hexo@4.2.1 // 忽略Hexo，本人写博用的
`-- yarn@1.22.4
```

## vue脚手架安装

为了更加便捷的创建一个vue项目，我们安装脚手架@vue/cli和@vue/init（vue-cli已经deprecated了）。之所以要安装@vue/init，是因为@vue/cli是3的版本，创建项目使用命令**vue create app-name**，且无法暂时无法使用模板，但是下文要用electron-vue模板进行创建，还是需要vue2的init命令来通过指定模板创建项目，为了兼容vue2的init特性，官方提供@vue/init作为桥接方式。

```shell
$ npm install -g @vue/cli
$ npm install -g @vue/cli-init

$ npm list -g --depth 0
+-- @vue/cli@4.4.6
+-- @vue/cli-init@4.4.6
+-- hexo@4.2.1
`-- yarn@1.22.4

关于init说明：
vue init [options] <template> <app-name>       generate a project from a remote template (legacy API, requires @vue/cli-init)
```

至此，你可以使用如下的形式，利用vue脚手架结合模板（template）来初始化你的vue项目（app-name）结构

```
vue init [options] <template> <app-name>
```

## 初始化Electron项目结构

在指定的目录下，我们使用如下的命令进行electron-vue的项目初始化：

```shell
$  vue init simulatedgreg/electron-vue electron-vue-demo
```

然而，这个过程很慢，甚至卡住不动。原因是指定模板进行创建时，会拉取github上的仓库进行模板初始化。幸运的是vue提供模板离线初始化的功能。

下载模板源码

https://github.com/SimulatedGREG/electron-vue

下载后解压存放在 **用户目录/.vue-templates/** 下（没有就创建，注意复数s），形成如下的结构：

```
{home目录}/
  .vue-templates/
     electron-vue-master/（目录名随便，但是在待会儿init指定的时候需要一致）
       .github/
       template/
       ....
```

之后就可以使用离线（offline）模式创建：

```shell
 vue init --offline electron-vue-master electron-vue-demo # 名称和上述文件夹名称一致即可
```

之后就是按照向导进行创建工作：

```bash
$ vue init --offline electron-vue-master electron-vue-demo
> Use cached template at ~\.vue-templates\electron-vue-master

? Application Name electron-vue-demo（项目名）
? Application Id com.compilemind（Id，这里本人使用了自己的域名）
? Application Version 0.0.1（版本）
? Project description electron vue demo（描述）
? Use Sass / Scss? No（是否使用Sass/Scss编译器）
? Select which Vue plugins to install (Press <space> to select, <a> to toggle all, <i> to invert selection)
axios, vue-el, ectron, vue-router, vuex, vuex-electron（插件包）
? Use linting with ESLint? Yes（启用ESlint）
? Which ESLint config would you like to use? Standard（ESLint配置）
? Set up unit testing with Karma + Mocha? No（测试模块）
? Set up end-to-end testing with Spectron + Mocha? No（测试模块）
? What build tool would you like to use? builder # 这里我们使用electron-builder构建可执行程序
? author w4ngzhen <w4ngzhen@hotmail.com>

   vue-cli · Generated "electron-vue-demo".
```

## 运行Electron-Vue示例

```
  $ cd electron-vue-demo
  $ yarn (or `npm install`)
  $ yarn run dev (or `npm run dev`)
```

在install的过程中，我们会看到如下一段console输出：

```shell
> electron@2.0.18 postinstall D:\Projects\electron-vue-demo\node_modules\electron
> node install.js

Downloading electron-v2.0.18-win32-x64.zip
[==========================================>  ] 97.0% of 50.7 MB (2.77 MB/s)
```

可以注意到electron的node包安装完成后，配置了postinstall（安装完成后调用的脚本），该脚本执行其内部install.js脚本后，开始下载electron的二进制的文件。

首先为什么会有这个额外下载的过程呢？在本人看来，electron是基于Chromium内核的跨平台客户端解决方案（本人另一篇文章正好进行了CefSharp的封装工作），既然涉及到跨平台，而不同平台的底层实现必然有所差异，那么electron项目通过自己去实现跨平台，封装底层逻辑，让我们不关心底层的实现，而是专心于前端的开发，封装成果就是上述的electron-v2.0.18-win32-x64.zip内容。这里因为我们调试和构建的时候，就需要运行时，所以electron根据我们的当前的平台，去下载了对应已经完成针对平台编译封装的二进制内容。

为什么要下载的问题搞明白了，接下来我们要看看如何去下载。有些朋友可能会发现，自己在进行electron二进制包下载的时候，速度慢的离谱。为什么这么慢？我会过一两天对下载的脚本一探究竟（时间有限，过两天写）

现阶段我们需要在.npmrc文件中增加一行配置，让electron下载二进制文件的时候从指定的地方下载：

```shell
ELECTRON_MIRROR=http://npm.taobao.org/mirrors/electron/
```

完成后，我们在install会发现有明显的提升。完成node包的install后，我们运行命令

```shell
$ npm run dev
```

启动后会发现客户端能够运行起来（即主进程能够运行），但是渲染进程报错：

**Webpack ReferenceError:process is not defined**，官方ISSUE已经存在该条：[ISSUE](https://github.com/SimulatedGREG/electron-vue/issues/871)

解决方案为：移除src\index.ejs中的该段代码，详细原因可以看ISSUE。

```js
// src\index.ejs
    <% if (!process.browser) { %>
      <script>
        if (process.env.NODE_ENV !== 'development') window.__static = require('path').join(__dirname, '/static').replace(/\\/g, '\\\\')
      </script>
    <% } %>
```

移除后，再次运行可以看到渲染成功的界面。


## 引入ElementUI

引入ElementUI相关包

```shell
npm install element-ui -S
```
在渲染进程模块的main.js中加入ElementUI组件

```js
import ElementUI from 'element-ui' 
import 'element-ui/lib/theme-chalk/index.css' 
...
Vue.use(ElementUI)
```

完成配置以后，我们就可以按照以往的方式进行前端的开发了。
