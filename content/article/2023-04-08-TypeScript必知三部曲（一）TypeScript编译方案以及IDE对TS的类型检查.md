---
title: TypeScript必知三部曲（一）TypeScript编译方案以及IDE对TS的类型检查
date: 2023-04-08
tags:
  - ts
  - babel
  - webpack
categories:
  - 技术
  - TypeScript必知三部曲 
---

TypeScript代码的编译过程一直以来会给很多小伙伴造成困扰，typescript官方提供tsc对ts代码进行编译，babel也表示能够编译ts代码，它们二者的区别是什么？我们应该选择哪种方案？为什么IDE打开ts项目的时候，就能有这些ts代码的类型定义？为什么明明IDE对代码标红报错，但代码有能够编译出来？

带着这些问题，我们由浅入深介绍**TypeScript**代码编译的两种方案以及**我们日常使用IDE进行ts文件类型检查**
的关系，让你今后面对基于ts的工程能够做到游刃有余。

<!-- more -->

# 写在前面

其实这篇文章并非是全新的文章，早在22年的8月份，我就写了一篇名为《TypeScript与Babel、webpack的关系以及IDE对TS的类型检查》的文章，里面的内容就包含了本文的内容，但迫于当时编写的匆忙，整个文章的结构安排的不好，脉络不清晰，东一块西一块想到哪里写到哪里，同时还想把webpack相关的也介绍了，所以最终内容比较多比较乱。有强迫症的我一直以来对当时的文章都不是很满意。

恰好刚好最近又在写有关TSX（基于TypeScript代码的JSX代码）的类型检查相关的介绍，故重新将当时的文章翻了出来，重新编排整理了内容，增加了更多的示意图，移除了有关webpack的部分，着重介绍现阶段TypeScript代码的编译方案，让文章内容更加聚焦。而在三部曲的第二部分，则会着重介绍本文移除了的对于webpack工程如何编译TypeScript项目的内容（考虑到该部分内容需要有本文的基础，故放在了第二部分）。在最后一部分将会介绍TSX的类型检查。

# TypeScript基本原则

**原则1：主流的浏览器的主流版本只认识js代码**

**原则2：ts的代码一定会经过编译为js代码，才能运行在主流浏览器上**

# TypeScript编译方式

首先，想要编译ts代码，至少具备以下3个要素：

1. ts源代码
2. ts编译器
3. ts编译配置

![010-ts-code-compile-flow](https://static-res.zhen.wang/images/post/2023-04-08/010-ts-code-compile-flow.png)

上述过程为：**ts编译器**读取**ts源代码**，并通过指定的**编译配置**，将ts源代码编译为指定形式的js代码。

目前主流的ts编译方案有2种，分别是：

1. tsc编译
2. babel编译

接下来将详细介绍上述两种方案以及它们之间的差异。

## tsc编译

官方编译方案，按照TypeScript官方的指南，你需要使用tsc（TypeScript Compiler）完成，该tsc来源于你本地或是项目安装的typescript包中。

按照上面的ts代码编译3要素，我们可以完成一一对应：

1. ts源代码
2. ts编译器：**tsc**
3. ts编译配置：**tsconfig.json**

![020-ts-code-compile-by-tsc](https://static-res.zhen.wang/images/post/2023-04-08/020-ts-code-compile-by-tsc.png)

让我们通过一个simple-tsc-demo，实践这一过程。

首先，创建一个名为simple-tsc-demo的空文件夹，并进行`yarn init`（`npm init`亦可）。然后，我们按照上述的三要素模型，准备：

**（1）ts源代码**：编写`项目根目录/src/index.ts`

```ts
interface User {
    id: string;
    name: string;
}

export const userToString = (u: User) => `${u.id}/${u.name}`
```

**（2）编译器tsc**：安装typescript获得

```bash
yarn add typescript
```

**（3）编译配置tsconfig.json：**`项目根目录/tsconfig.json`

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

简单介绍上述tsconfig.json配置：

1. module：指定ts代码编译生成何种模块方案的js代码，这里暂时写的commonjs，后续会介绍其它值的差异；
2. rootDir：指定ts代码存放的根目录，这里就是当前目录（项目根目录）下的src文件夹，能够匹配到我们编写的
   `项目根目录/src/index.ts`；
3. outDir：指定ts代码经过编译后，生成的js代码的存放目录。

当然，为了方便执行命令，我们在package.json中添加名为`build`的脚本：

```diff
{
  ... 
+ "scripts": {
+  "build": "tsc"
+ },
  ...
}
```

完成搭建以后，项目整体如下：

![030-simple-tsc-example-full](https://static-res.zhen.wang/images/post/2023-04-08/030-simple-tsc-example-full.png)

运行build脚本，能够看到在项目根目录产生`dist/index.js`：

![040-simple-tsc-compile-result-commonjs](https://static-res.zhen.wang/images/post/2023-04-08/040-simple-tsc-compile-result-commonjs.png)

对于index.js的内容，熟悉js模块化规范的小伙伴应该很容易看出这是commonjs的规范：给exports对象上添加属性字段，exports对象会作为模块导出，被其他模块使用。

之所以产生的js代码是符合commonjs模块规范的代码，源于我们在tsconfig.json中配置的module值为`commonjs`。倘若我们将module字段改为
`es6`：

```diff
{
  "compilerOptions": {
- 	"module": "commonjs",
+   "module": "es6",
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

再一次编译以后，会看到编译后的js代码则是符合es6模块规范的代码：

![050-simple-tsc-compile-result-es6](https://static-res.zhen.wang/images/post/2023-04-08/050-simple-tsc-compile-result-es6.png)

对于tsc编译方案，按照TypeScript编译三要素模型简单总结一下：我们准备了ts源码、tsc编译器以及tsconfig.json配置。通过**tsc编译器
**读取**tsconfig.json编译配置**，将ts源码编译为了js代码。此外，在tsconfig.json中，我们配置了生成的js代码的两种模块规范：
`"module": "commonjs"`与`"module": "es6"`，并验证了其结果符合对应的模块规范。

对于编译器这部分来说，除了上面我们尝试过的tsc编译器，是否还存在其他的编译器呢？答案是肯定的：babel。

## babel编译

本文并不是一篇专门讲babel的文章，但是为了让相关知识能够比较好的衔接，还是需要介绍这块内容的。当然如果读者有时间，我推荐这篇深入了解babel的文章：[一口（很长的）气了解 babel - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/43249121)。

> babel 总共分为三个阶段：解析，转换，生成。
>
>babel 本身不具有任何转化功能，它把转化的功能都分解到一个个 plugin 里面。因此当我们不配置任何插件时，经过 babel 的代码和输入是相同的。
>
>插件总共分为两种：
>
>- 当我们添加 **语法插件** 之后，在解析这一步就使得 babel 能够解析更多的语法。(顺带一提，babel 内部使用的解析类库叫做
   babylon，并非 babel 自行开发)
>
>举个简单的例子，当我们定义或者调用方法时，最后一个参数之后是不允许增加逗号的，如 `callFoo(param1, param2,)`
> 就是非法的。如果源码是这种写法，经过 babel 之后就会提示语法错误。但最近的 JS 提案中已经允许了这种新的写法(让代码 diff
> 更加清晰)。为了避免 babel 报错，就需要增加语法插件 `babel-plugin-syntax-trailing-function-commas`
>
>- 当我们添加 **转译插件** 之后，在转换这一步把源码转换并输出。这也是我们使用 babel 最本质的需求。
>
>比起语法插件，转译插件其实更好理解，比如箭头函数 `(a) => a` 就会转化为 `function (a) {return a}`。完成这个工作的插件叫做
`babel-plugin-transform-es2015-arrow-functions`。
>
>同一类语法可能同时存在语法插件版本和转译插件版本。**如果我们使用了转译插件，就不用再使用语法插件了。**

总结来说，babel转换代码就像如下流程：

```
源代码 -(babel)-> 目标代码
```

如果没有使用任何插件，源代码和目标代码就没有任何差异。当我们引入各种插件的时候，就像如下流程一样：

```
源代码
|
进入babel
|
babel插件1处理代码，例如移除某些符号
|
babel插件2处理代码，例如将形如() => {}的箭头函数，转换成function xxx() {}
|
目标代码
```

> babel提倡一个插件专注做一个事情，比如某个插件只进行箭头函数转换工作，某个插件只处理将const转var代码，这样设计的好处是可以灵活的组合各种插件完成代码转换。
>
>
但又因为babel的插件处理的力度很细，JS代码的语法规范有很多，为了处理这些语法，可能需要配置一大堆的插件。为了解决这个问题，babel设计preset（预置集）概念，preset组合了一堆插件。于是，我们只需要引入一个插件组合包preset，就能处理代码的各种语法。
>
> PS：官方收编的插件包通常以 “@babel/plugin-” 开头的，而预置集包通常以 “@babel/preset-” 开头。

回到TypeScript编译，对于babel编译TS的体系，我们同样按照TypeScript编译三要素模型，来一一对应：

1. ts源码
2. ts编译器：**babel+相关preset、plugin**
3. ts编译配置：**.babelrc**

![060-ts-code-compile-by-babel](https://static-res.zhen.wang/images/post/2023-04-08/060-ts-code-compile-by-babel.png)

同样的，让我们通过一个simple-babel-demo，实践这一过程。

首先，创建一个名为simple-babel-demo的空文件夹，并进行`yarn init`（`npm init`亦可）。然后，我们按照上述的三要素模型，准备：

**（1）源代码**：编写`项目根目录/src/index.ts`

```ts
interface User {
    id: string;
    name: string;
}

export const userToString = (u: User) => `${u.id}/${u.name}`
```

**（2）ts编译器babel+相关preset、plugin**：项目安装如下依赖包

```bash
yarn add -D @babel/cli @babel/core
yarn add -D @babel/preset-env @babel/preset-typescript
yarn add -D @babel/plugin-proposal-object-rest-spread
```

读者看到需要安装这么多的依赖包不要感到恐惧，让我们一个一个分析：

- `@babel/core`：babel的核心模块，控制了整体代码编译的运转以及代码语法、语义分析的功能；

- `@babel/cli`：支持我们可以在控制台使用babel命令；

- `@babel/preset-`开头的就是预置组件包合集，其中`@babel/preset-env`
  表示使用了可以根据实际的浏览器运行环境，会选择相关的转义插件包，通过配置得知目标环境的特点只做必要的转换。如果不写任何配置项，env
  等价于 latest，也等价于 es2015 + es2016 + es2017 三个相加(不包含 stage-x 中的插件)；`@babel/preset-typescript`
  会处理所有ts的代码的语法和语义规则，并转换为js代码。

- plugin开头的就是插件，这里我们引入：
  `@babel/plugin-proposal-object-rest-spread`（[对象展开](https://babel.docschina.org/docs/en/7.0.0/babel-plugin-proposal-object-rest-spread/)
  ），它会处理我们在代码中使用的`...`运算符转换为普通的js调用。

介绍完以后，是不是有了一些清晰的认识了呢。让我们继续三要素的最后一个：编译配置。

**（3）编译配置.babelrc**：`项目根目录/.babelrc文件`

```json
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-typescript"
  ],
  "plugins": [
    "@babel/plugin-proposal-object-rest-spread"
  ]
}
```

上面的配置并不复杂，对应了我们安装依赖包中关于preset与plugin的部分。这部分配置，也是在告诉babel，处理代码的时候，需要加载哪些preset、plugin好让它们处理代码。

最后，我们在package.json添加编译脚本：

```diff
{
	...
+ "scripts": {
+ 	"build": "babel src --config-file ./.babelrc -x .ts -d dist"
+ },
	...
}
```

编译指令指定了babel要读取的源代码所在目录（`src`）、babel配置文件地址（`--config-file ./.babelrc`）、babel需要处理的文件扩展（
`-x .ts`）、编译代码生成目录（`-d dist`）。

完成项目搭建以后，整体如下：

![070-simple-babel-example-full](https://static-res.zhen.wang/images/post/2023-04-08/070-simple-babel-example-full.png)

运行build脚本，能够看到在项目根目录产生`dist/index.js`：

![080-simple-babel-compile-result-commonjs](https://static-res.zhen.wang/images/post/2023-04-08/080-simple-babel-compile-result-commonjs.png)

这段代码，与上面tsc基于commonjs编译的js代码差别不大。也就是说，babel基于`@babel/preset-env`+`@babel/preset-typescript`
就能将TS代码编译为commonjs代码。那么我们如何使用babel将ts代码编译器es6的代码呢？从babel配置下手，实际上，我们只需要将babelrc的
`@babel/preset-env`移除即可：

```diff
{
  "presets": [
-  	"@babel/preset-env",
    "@babel/preset-typescript"
  ],
  "plugins": [
    "@babel/plugin-proposal-object-rest-spread"
  ]
}
```

再次编译后，可以看到生成的index.js符合es6规范：

![090-simple-babel-compile-result-es6](https://static-res.zhen.wang/images/post/2023-04-08/090-simple-babel-compile-result-es6.png)

对于babel编译，同样简单总结一下，对应TypeScript编译三要素模型，我们准备了ts源码、babel与相关preset和plugin作为编译器，以及babelrc作为编译配置。babel处理代码的流程启动以后，根据编译配置知道需要加载哪些plugin、preset，将代码以及相关信息交给plugin、preset处理，最终编译为js代码。此外，在babelrc中，我们通过是否配置
`@babel/preset-env`控制生成满足commonjs或es6模块规范的js代码。

## 编译总结

不难看出，**ts无论有多么庞大的语法体系，多么强大的类型检查，最终的产物都是js**。

此外还要注意的一点是，ts中的模块化不能和js中的模块化混为一谈。js中的模块化方案很多（es6、commonjs、umd等等），所以ts本身在编译过程中，需要指定一种js的模块化表达，才能编译为对应的代码。在ts中的
`import/export`，不能认为和es6的`import/export`是一样的，他们是完全不同的两个体系！只是语法上相似而已。

# tsc编译与babel编译的差异

前面，我们介绍了tsc编译与babel编译TS代码，那他们二者有什么差异呢？让我们先来看这样一个场景：下面是一段ts源代码：

```ts
interface User {
    id: string;
    name: string;
}

export const userToString = (u: User) => `${u.id}/${u.name}`
```

我们故意将`u.name`错写为`u.myName`：

```diff
- export const userToString = (u: User) => `${u.id}/${u.name}`
+ export const userToString = (u: User) => `${u.id}/${u.myName}`
```

预期上讲，类型检查肯定不通过，因为`User`接口根本没有`name`字段。让我们分别在tsc编译和babel编译中看一下编译的结果是否满足我们的预期。

## tsc编译错误代码

![100-tsc-compile-error-code](https://static-res.zhen.wang/images/post/2023-04-08/100-tsc-compile-error-code.png)

可以从结果很清楚的看到，使用tsc编译错误代码的时候，tsc类型检查帮助我们找到了代码的错误点，符合我们的预期。

## babel编译错误代码

![110-babel-compile-error-code](https://static-res.zhen.wang/images/post/2023-04-08/110-babel-compile-error-code.png)

从结果来看，babel编译居然可以直接成功！查看生成的index.js代码：

```js
export const userToString = u => `${u.id}/${u.myName}`;
```

从js代码角度来看，这段代码没有任何的问题，此时的`u`参数变量在js层面，并没有明确的类型定义，js作为动态语言，运行的时候，
`myName`也可能就存在，这谁也无法确定。

为什么babel编译会这样处理代码？这不得不提到babel中的`@babel/preset-typescript`是如何编译TS代码的：

> 警告！有一个震惊的消息，你可能想坐下来好好听下。
>
> Babel 如何处理 TypeScript 代码？**它删除它**。
>
> 是的，它删除了所有 TypeScript，将其转换为“常规的” JavaScript，并继续以它自己的方式愉快处理。
>
> 这听起来很荒谬，但这种方法有两个很大的优势。
>
> 第一个优势：️⚡️**闪电般快速**⚡️。
>
> 大多数 Typescript 开发人员在开发/监视模式下经历过编译时间长的问题。你正在编写代码，保存一个文件，然后...它来了...再然后...
**最后**，你看到了你的变更。哎呀，错了一个字，修复，保存，然后...啊。它**只是**慢得令人烦恼并打消你的势头。
>
> 很难去指责 TypeScript 编译器，它在做很多工作。它在扫描那些包括 `node_modules` 在内的类型定义文件（`*.d.ts`
> ），并确保你的代码正确使用。这就是为什么许多人将 Typescript 类型检查分到一个单独的进程。然而，Babel + TypeScript
> 组合仍然提供更快的编译，这要归功于 Babel 的高级缓存和单文件发射架构。

具体的内容小伙伴可以查看： [TypeScript 和 Babel：美丽的结合 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/59614089)。

也就是说，**babel处理TypeScript代码的时候，并不进行任何的类型检查！**那小伙伴可能会说，那如果我使用babel编译方案，怎么进行类型检查以确保ts代码的正确性呢？答案则是：
**引入tsc，但仅仅进行类型检查**。

回到我们之前的simple-babel-example。在之前的基础上，我们依旧安装typescript从而获得tsc：

```diff
{
	...
	"devDependencies": {
    "@babel/cli": "^7.21.0",
    "@babel/core": "^7.21.4",
    "@babel/plugin-proposal-object-rest-spread": "^7.20.7",
    "@babel/preset-env": "^7.21.4",
    "@babel/preset-typescript": "^7.21.4",
+   "typescript": "^5.0.4"
  }
}
```

然后，在项目中添加tsconfig.json文件，配置如下

```json
{
  "compilerOptions": {
    "noEmit": true,
    "rootDir": "src"
  }
}
```

比起tsc编译方案里面的配置有所不同，在babel编译方案中的类型检查的tsconfig.json需要我们配置`noEmit`为`true`，表明*
*tsc读取到了ts源代码以后，不会生成任何的文件，仅仅会进行类型检查**。

于是，在babel编译方案中，整个体系如下：

![120-babel-compile-flow-with-type-check](https://static-res.zhen.wang/images/post/2023-04-08/120-babel-compile-flow-with-type-check.png)

# 主流IDE对TS项目如何进行类型检查

不知道有没有细心的读者在使用IDEA的时候，会发现如果是IDE当前打开的TS文件，IDEA右下角会展示一个typescript：

![130-idea-typescript-service](https://static-res.zhen.wang/images/post/2023-04-08/130-idea-typescript-service.png)

VSCode同样也会有：

![140-vscode-typescript-service](https://static-res.zhen.wang/images/post/2023-04-08/140-vscode-typescript-service.png)

在同一台电脑上，甚至发现IDEA和VSCode的typescript版本都还不一样（5.0.3和4.9.5）。这是怎么一回事呢？实际上，IDE检测到你所在的项目是一个ts项目的时候（或当前正在编辑ts文件），就会自动的启动一个ts的检测服务，专门用于当前ts代码的类型检测。这个ts类型检测服务，同样使用tsc来完成，但这个tsc来源于两个途径：

1. 每个IDE默认情况下自带的typescript中的tsc
2. 当前项目安装的typescript的tsc

例如，上图本人机器上的IDEA，因为检测到了项目安装了`"typescript": "^5.0.3"`
，所以自动切换为了项目安装的TypeScript；而VSCode似乎没有检测到，所以使用VSCode自带的。

当然，你也可以在IDE中手动切换：

![150-IDEA-and-VSCode-switch-typescript](https://static-res.zhen.wang/images/post/2023-04-08/150-IDEA-and-VSCode-switch-typescript.png)

最后，我们简单梳理下IDE是如何在对应的代码位置展示代码的类型错误，流程如下：

![160-IDE-ts-check-flow](https://static-res.zhen.wang/images/post/2023-04-08/160-IDE-ts-check-flow.png)

但是，同样是IDE中的ts类型检查也要有一定的依据。譬如，外部库的类型定义的文件从哪里查找，是否允许较新的语法等，这些配置依然是由tsconfig.json来提供的，但若未提供，则IDE会使用一份默认的配置。如果要进行类型检测的自定义配置，则需要提供tsconfig.json。

# 编译方案与IDE类型检查整合

综合前面的tsc编译与babel编译的过程，再整理上述的IDE对TS项目的类型检查，我们可以分别总结出tsc编译与babel编译两种场景的代码编译流程和IDE类型检查流程。

首先是tsc编译方案：

![170-tsc-compile-and-type-check](https://static-res.zhen.wang/images/post/2023-04-08/170-tsc-compile-and-type-check.png)

在这套方案中，ts项目的代码本身的编译，会走项目安装的typescript，并加载项目本身的tsconfig.json配置。同时，IDE也会利用项目本身的typescript以及读取相同配置的tsconfig.json来完成项目代码的类型检查。

于是，无论是代码编译还是IDE呈现的类型检查，都是走的一套逻辑，当IDE提示了某些ts代码的编译问题，那么ts代码编译一定会出现相同的问题。
**不会存在**这样的情况：代码有编译问题，但是IDE不会红色显示类型检查问题。

再来看babel编译方案：

![180-babel-compile-and-type-check](https://static-res.zhen.wang/images/post/2023-04-08/180-babel-compile-and-type-check.png)

很显然，babel编译方案，代码编译与IDE的类型检查是两条路线。也就是说，有可能你的IDE提示了错误，但是babel编译是没有问题。这也是很多小伙伴拿到基于babel编译的TS项目容易出现IDE有代码异常问题的UI显示，但是编译代码有没有问题的原因所在。

# 写在最后

本文着重介绍了TypeScript代码的两种编译方案，以及IDE是如何进行TypeScript的类型检查的。作为三部曲的第一部，内容比较多，比较细，感谢读者的耐心阅读。接下来的剩余两部分，将分别介绍webpack如何编译打包基于TypeScript的项目以及TSX是如何进行类型检查。