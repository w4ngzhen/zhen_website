---
title: Linux下Electron loadURL报错 ERR_FAILED Not allowed to load local resource
date: 2021-02-24
tags:
  - electron
categories:
  - 技术
---

# Linux下Electron loadURL报错 ERR_FAILED(-2) Not allowed to load local resource

Linux Electron打包后页面无法加载，报错：Not allowed to load local resource

<!-- more -->

## 背景

使用electron-vue的时候，窗体创建后加载页面使用的是`loadURL`函数，并且传入的参数形如：

```js
`file://${__dirname}/index.html`
```

PS：在**electron-vue**中，编译打包后的`__dirname`是`asar所在绝对地址/dist/electron/`。

编译打包后的Electron程序启动时候，发现在调用loadURL的时候会出现：ERR_FAILED(-2) 。

遂临时启动DevTool窗口查看控制台输出，发现类似如下错误：

```
Not allowed to load local resource: file://XXXXXX/app.asar/dir1/dir2/index.html
```

## 原因及方案

### 1、文件并不存在于你的asar包中

处理方式：

1. 全局安装`npm install -g asar`
2. 通过asar解压app.asar包，检查上述提到的file路径中是否存在你的index.html。

### 2、尝试禁用窗体参数中的`webSercurity`

处理方式：

```js
{
  webPreferences: {
    webSecurity: false
  }
}
```

### 3、检查webpack配置中的__dirname，防止webpack处理'mock'

这一点主要是当上述`Not alloed to load local resource`提到的路径明显不正确时候，可以检查。

```
  // webpack配置文件中的node节点，当打包的时候需要防止webpack处理
  node: {
    __dirname: false
  }
```

webpack配置文件中的node节点，当打包的时候需要防止webpack处理`__dirname`，如果你使用了electron-vue脚手架应该不需要关心，因为你会看到如下的内容，已经帮你处理了：

```
  node: {
    __dirname: process.env.NODE_ENV !== 'production',
    __filename: process.env.NODE_ENV !== 'production'
  },
```

### 4、使用loadFile而不是loadURL

使用loadFile接口来加载本地的路径，loadFile('path/to/index.html')，这个路径是以app.asar根路径为base的。例如，loadFile('
dir_path1/dir_path2/index.html')，那么你的index.html在app.asar是如下的结构：

```
app.asar
|-dir_path1
  |-dir_path2
    |-index.html
```~~
