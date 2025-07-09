---
title: 详解prettier使用以及与主流IDE的配合
date: 2023-07-17
tags:
 - prettier
 - IDE
categories:
  - 技术
---

很多前端小伙伴在日常使用prettier的时候都或多或少有一点疑惑，prettier在每一个IDE中究竟是怎样工作起来的，为什么配置有时候生效，有时又毫无效果。为了让我们的前端小伙伴更加熟悉这块，本文将对prettier在主流IDE中的使用过程一探究竟。

<!-- more -->

# prettier是什么

在介绍prettier如何集成到IDE中之前，让我们了解下prettier是什么。prettier是一款代码格式化工具：

- 一个“有态度”的代码格式化工具
- 支持大量编程语言
- 已集成到大多数编辑器中

它的基本使用过程如下：

![010-prettier-arch-simple](https://static-res.zhen.wang/images/post/2023-07-17/010-prettier-arch-simple.png)

正如上面的流程图所示，prettier不过就是一个安装在机器上的软件，它接收源代码，配合prettier的配置，完成对源代码的格式化。那么如何安装prettier呢？

# 安装prettier

prettier实际上是通过js编写的node模块。它可以有两种方式进行安装：

1. 作为全局工具进行全局级别安装：`npm install -g prettier`。这种方式下，只要你配置过了全局包所在路径到环境变量，那么你就可以在命令行中使用`prettier`命令行调用它。
2. 作为某个前端项目进行项目级别安装：在某个项目下`npm install -D pretter`或是`yarn add -D prettier`。这种方式下，prettier作为项目级别安装，只会在你的node_modules下安装prettier的包。此时，**在项目根目录下**，你就可以通过调用`npx prettier`来调用它。

无论是哪种方式，我们的最终目的都是要安装好这个工具，并能调用它。例如，本人以全局安装方式安装了prettier以后，在命令行就能访问到它：

```bash
$ which prettier
/usr/local/bin/prettier
$ prettier --version
3.0.0
```

# CLI使用prettier

为了简单使用prettier，我们创建一个名为`prettier-demo`的目录，在里面编写一个demo.js：

![020-prettie-usage-demo](https://static-res.zhen.wang/images/post/2023-07-17/020-prettie-usage-demo.png)

demo.js代码有两个格式问题：

1. `return 'tom'`之前的有太多的空格；
2. `getName`换行不对劲。

进入prettier-demo目录以后，我们在项目根目录下使用如下的命令行进行操作：

![030-prettie-check-and-format](https://static-res.zhen.wang/images/post/2023-07-17/030-prettie-check-and-format.png)

1. （可选）使用`prettier -c ./demo.js`来检查一下当前的文档是否存在有代码样式问题。这里prettier告诉我们代码存在样式问题；
2. 使用`prettier ./demo.js`对代码进行格式化。这里prettier帮助我们进行格式化处理，并输出到了控制台。由于prettier默认是将格式化后的代码输入到控制台的，为了能够直接将格式化代码写回到源文件，需要添加`--write`命令行：`prettiter --write ./demo.js`。结果如下，会看到格式化后的代码：

![040-prettier-code-formatted](https://static-res.zhen.wang/images/post/2023-07-17/040-prettier-code-formatted.png)

当然，prettier格式化代码的时候，需要的配置项还有很多，例如：是否行尾添加分号；tab等于多少空格；字符串使用单引号还是双引号等等规则配置。尽管这些配置是可以通过命令行参数形式传递，但是一旦规则数量太多，命令行很明显不是一个好的方式，而更加优雅的方式则是使用配置文件：[Configuration File · Prettier](https://prettier.io/docs/en/configuration.html)

配置文件最简单的方式，就是在项目根目录下添加一个名为`.prettierrc.json`的文件，然后在其中编写配置。例如，在本例中，我们在prettier-demo项目根目录下创建名为`.prettierrc.json`文件，并编写如下的内容：

```json
{
  "tabWidth": 4,
  "semi": true,
  "singleQuote": false
}
```

`"tabWidth": 4`表明一个tab等同于4个空格；`"semi": true`表明使用分号结尾；`"singleQuote": false`表明字符串等使用双引号。

> 注意！这里只是一个演示demo，并不是格式化规范的最佳实践。

至此，我们的demo目录结构如下：

```
~/projects/web-projects/prettier-demo/
  - .prettierrc.json
  - demo.js
```

有了这套配置，让我们再次格式化代码（`prettier --write ./demo.js`），会发现prettier按照我们的配置规则进行了代码格式化：

![050-prettier-code-by-config-file](https://static-res.zhen.wang/images/post/2023-07-17/050-prettier-code-by-config-file.png)

# 主流IDE中使用prettier

上面介绍了如何以原生的方式使用prettier。然而一般来说，我们都会使用IDE来进行应用开发，我们很少会为了使用prettier的格式化功能专门使用命令行。还好，无论是JetBrains系的IDEA或WebStorm还是VSCode，都有对应的插件来调用prettier。它们的整体思路大体都是一样的：

![060-IDE-plugin-prettier](https://static-res.zhen.wang/images/post/2023-07-17/060-IDE-plugin-prettier.png)

1. IDE安装prettier插件；
2. prettier插件调用prettier工具

接下来我们详细介绍这两种主流IDE的pretter环境配置过程。

## IDEA和WebStorm

在IDEA和WebStorm中（后续统一使用IDEA进行讲解，他们是一样的机制），我们首先安装prettier插件：

![070-IDEA-prettier-plugin-install](https://static-res.zhen.wang/images/post/2023-07-17/070-IDEA-prettier-plugin-install.png)

安装完成以后，我们需要给IDEA配置一下prettier插件：

在配置 | Languages & Frameworks | JavaScript | Prettier 中，我们可以配置IDEA关于prettier插件本身的配置：

![080-IDEA-prettier-plugin-config-disable](https://static-res.zhen.wang/images/post/2023-07-17/080-IDEA-prettier-plugin-config-disable.png)

在这里，主要有三种方式：

**选项1：Disable Prettier 禁用prettier**。也就是说不会调用prettier进行格式化；

**选项2：Automatic Prettier configuration 自动配置**

![090-IDEA-prettier-plugin-config-automatic](https://static-res.zhen.wang/images/post/2023-07-17/090-IDEA-prettier-plugin-config-automatic.png)

对于这种方式，IDEA会首先搜索项目node_modules下安装的prettier以及在项目目录中能定位到的`.prettierrc.*`配置文件，这块主要是IDEA自动检测。同时，默认会对js、ts、jsx、tsx等等前端文件提供格式化支持。

**方式3：Manual Prettier configuration 手动配置**

![100-IDEA-prettier-plugin-config-manual](https://static-res.zhen.wang/images/post/2023-07-17/100-IDEA-prettier-plugin-config-manual.png)

当我们选择手动配置的时候，需要我们手动的选择应该使用哪个prettier工具。由于本demo中，我们是通过`npm -g`全局安装的prettier，所以我们手动选择全局安装的prettier工具。同时，依然默认对js、ts、jsx、tsx等等前端文件提供格式化支持。

当然，无论是自动配置、手动配置，我们还会发现下面有这样一项配置：`Run on save`，配置了它以后，我们在在保存文件的时候，就会自动进行格式化处理。如果不配置该选贤，则我们需要在打开源代码文件以后，右键选择使用prettier进行格式化：

![110-IDEA-prettier-format-by-options](https://static-res.zhen.wang/images/post/2023-07-17/110-IDEA-prettier-format-by-options.png)

## VSCode

在VSCode中，使用prettier的思路是一样的。首先，我们安装prettier插件：

![120-vscode-prettier-plugin-install](https://static-res.zhen.wang/images/post/2023-07-17/120-vscode-prettier-plugin-install.png)

安装完成以后，我们会看到右下角插件的安装结果：

![130-vscode-plugin-install-tip](https://static-res.zhen.wang/images/post/2023-07-17/130-vscode-plugin-install-tip.png)

右键代码 - Format Document With...，可以选择使用prettier进行格式化：

![140-vscode-plugin-format](https://static-res.zhen.wang/images/post/2023-07-17/140-vscode-plugin-format.png)

格式化以后，我们会看到插件的输出，能够更加仔细的查看处理过程：

![150-vscode-plugin-output](https://static-res.zhen.wang/images/post/2023-07-17/150-vscode-plugin-output.png)

# 总结

本文主要介绍了prettier的使用以及在主流IDE中的使用，希望读者阅读本文以后，能够了解prettier与IDE如何进行配合。

