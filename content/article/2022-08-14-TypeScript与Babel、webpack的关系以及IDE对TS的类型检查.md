---
title: TypeScript与Babel、webpack的关系以及IDE对TS的类型检查
date: 2022-08-14
tags:
 - ts
 - babel
 - webpack
categories:
  - 技术
---

只要接触过ts的前端同学都能回答出ts是js超集，它具备静态类型分析，能够根据类型在静态代码的解析过程中对ts代码进行类型检查，从而在保证类型的一致性。那，现在让你对你的webpack项目（其实任意类型的项目都同理）加入ts，你知道怎么做吗？带着这个问题，我们由浅入深，逐步介绍**TypeScript**、**Babel**以及**我们日常使用IDE进行ts文件类型检查**的关系，让你今后面对基于ts的工程能够做到游刃有余。

<!-- more -->

# TypeScript基本认识

**原则1：主流的浏览器的主流版本只认识js代码**

**原则2：ts的代码一定会经过编译为js代码，才能运行在主流浏览器上**

要编译ts代码，至少具备以下几个要素：

1. ts源代码
2. ts编译器
3. ts编译器所需要的配置（默认配置也是配置）

# 编译TS的方式

目前主流的ts编译方案有2种，分别是官方tsc编译、babel+ts插件编译。

## 官方tsc编译器

对于ts官方模式来说，ts编译器就是tsc（安装typescript就可以获得），而编译器所需的配置就是tsconfig.json配置文件形式或其他形式。ts源代码经过tsc的编译（Compile），就可以生成js代码，在tsc编译的过程中，需要**编译配置**来确定一些编译过程中要处理的内容。

![010-ts-compile-flow](https://static-res.zhen.wang/images/post/2022-08-14/010-ts-compile-flow.png)

我们首先准备一个ts-demo，该demo中有如下的结构：

```
ts-demo
 |- packages.json
 |- tsconfig.json
 |- src
    |- index.ts
```

安装typescript

```
yarn add -D typescript
```

package.json

````json
{
  "name": "ts-demo",
  "version": "1.0.0",
  "description": "",
  "main": "index.js",
  "scripts": {
    "build-ts": "tsc"
  },
  "author": "",
  "license": "MIT",
  "devDependencies": {
    "typescript": "^4.7.4"
  }
}
````

tsconfig.js（对于这个简单的tsconfig，我不再赘述其配置的含义。）

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

index.ts

```typescript
interface User {
    name: string;
    age: number;
}
const userToString = (user: User) => `${user.name}@${user.age}`;
export {userToString, User};
```

此时，我们只需要运行`yarn build-ts`就可以将我们的index.ts编译为index.js：

commonjs模块化方式产物：

```js
"use strict";
exports.__esModule = true;
exports.userToString = void 0;
var userToString = function (user) { return "".concat(user.name, "@").concat(user.age); };
exports.userToString = userToString;
```

可以看到，原本index.ts编译为index.js的产物，使用了commonjs模块化方案（tsconfig里面配置模块化方案是"commonjs"，编译后的代码可以看到"exports"的身影）；倘若我们将模块化方案改为ESM（ES模块化）的es：`"module": "es6"`，编译后的产物依然是index.js，只不过内容采用了es6中的模块方案。

es6模块化方式产物：

```js
var userToString = function (user) {
  return "".concat(user.name, "@").concat(user.age);
};
export {userToString};
```

说了这么多，只是想要告诉各位同学，**ts无论有多么庞大的语法体系，多么强大的类型检查，最终的产物都是js**。

此外，ts中的模块化，不能和js中的模块化混为一谈。js中的模块化方案很多（es6、commonjs、umd等等），所以ts本身在编译过程中，需要指定一种js的模块化表达，才能编译为对应的代码。也就是说，在ts中的`import/export`，不能认为和es6的`import/export`是一样的，他们是完全不同的两个体系！只是语法上类似而已。

## babel+ts插件

如前文所述

>ts源代码经过tsc的编译（Compile），就可以生成js代码，在tsc编译的过程中，需要编译配置来确定一些编译过程中要处理的内容。

那么是不是说，编译器这块是不是有其他的代替呢？ts源码经过某种其他的编译器编译后，生成目标js代码。答案是肯定的：babel。

我们准备一个ts-babel-demo：

```
ts-babel-demo
 |- packages.json
 |- .babelrc
 |- src
    |- index.ts
```

依赖添加：

```shell
 yarn add -D @babel/core @babel/cli
 yarn add -D @babel/preset-env @babel/preset-typescript
 yarn add -D @babel/plugin-proposal-class-properties @babel/plugin-proposal-object-rest-spread
```

package.json：

```json
{
  "name": "ts-babel-demo",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "private": true,
  "scripts": {
    "build": "babel src -d dist -x '.ts, .tsx'"
  },
  "devDependencies": {
    "@babel/cli": "^7.18.10",
    "@babel/core": "^7.18.10",
    "@babel/plugin-proposal-class-properties": "^7.18.6",
    "@babel/plugin-proposal-object-rest-spread": "^7.18.9",
    "@babel/preset-env": "^7.18.10",
    "@babel/preset-typescript": "^7.18.6"
  }
}
```

.babelrc

```json
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-typescript"
  ],
  "plugins": [
    "@babel/plugin-proposal-object-rest-spread",
    "@babel/plugin-proposal-class-properties"
  ]
}
```

index.ts和ts-demo保持一致。

完成基础的项目搭建以后，我们执行`yarn build`：

```
~/Projects/web-projects/ts-babel-demo > yarn build
yarn run v1.22.17
$ babel src -d dist -x '.ts, .tsx'
Successfully compiled 1 file with Babel (599ms).
Done in 4.05s.
```

可以看到项目dist目录下出现了编译好的js代码：

```js
"use strict";

Object.defineProperty(exports, "__esModule", {
  value: true
});
exports.userToString = void 0;

var userToString = function userToString(user) {
  return "".concat(user.name, "@").concat(user.age);
};

exports.userToString = userToString;
```

可以看到和使用tsc编译为commonjs效果是一样。

回顾这个项目，其实按照我们之前的思路来梳理：

1. ts源文件（src/index.ts）
2. ts的编译器（babel）
3. 编译配置（.babelrc）

![020-babel-compile-flow](https://static-res.zhen.wang/images/post/2022-08-14/020-babel-compile-flow.png)

**了解babel机制**

如果对于babel不太熟悉，可能对上述的一堆依赖感到恐惧：

```
 yarn add -D @babel/core @babel/cli
 yarn add -D @babel/preset-env @babel/preset-typescript
 yarn add -D @babel/plugin-proposal-class-properties @babel/plugin-proposal-object-rest-spread
```

这里如果读者有时间，我推荐这篇深入了解babel的文章：[一口（很长的）气了解 babel - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/43249121)。当然，如果这口气憋不住（哈哈），我做一个简单摘抄：

>babel 总共分为三个阶段：解析，转换，生成。
>
>babel 本身不具有任何转化功能，它把转化的功能都分解到一个个 plugin 里面。因此当我们不配置任何插件时，经过 babel 的代码和输入是相同的。
>
>插件总共分为两种：
>
>- 当我们添加 **语法插件** 之后，在解析这一步就使得 babel 能够解析更多的语法。(顺带一提，babel 内部使用的解析类库叫做 babylon，并非 babel 自行开发)
>
>举个简单的例子，当我们定义或者调用方法时，最后一个参数之后是不允许增加逗号的，如 `callFoo(param1, param2,)` 就是非法的。如果源码是这种写法，经过 babel 之后就会提示语法错误。
>
>但最近的 JS 提案中已经允许了这种新的写法(让代码 diff 更加清晰)。为了避免 babel 报错，就需要增加语法插件 `babel-plugin-syntax-trailing-function-commas`
>
>- 当我们添加 **转译插件** 之后，在转换这一步把源码转换并输出。这也是我们使用 babel 最本质的需求。
>
>比起语法插件，转译插件其实更好理解，比如箭头函数 `(a) => a` 就会转化为 `function (a) {return a}`。完成这个工作的插件叫做 `babel-plugin-transform-es2015-arrow-functions`。
>
>同一类语法可能同时存在语法插件版本和转译插件版本。**如果我们使用了转译插件，就不用再使用语法插件了。**

简单来讲，使用babel就像如下流程：

```
源代码 =babel=> 目标代码
```

如果没有使用任何插件，源代码和目标代码就没有任何差异。当我们引入各种插件的时候，就像如下流程一样：

```
源代码
|
进入babel
|
babel插件1处理代码：移除某些符号
|
babel插件2处理代码：将形如() => {}的箭头函数，转换成function xxx() {}
|
目标代码
```

因为babel的插件处理的力度很细，我们代码的语法、语义内容规范有很多，如果我们要处理这些语法，可能需要配置一大堆的插件，所以babel提出，将一堆插件组合成一个preset（预置插件包），这样，我们只需要引入一个插件组合包，就能处理代码的各种语法、语义。

所以，回到我们上述的那些@babel开头的npm包，再回首可能不会那么迷茫：

```
@babel/core
@babel/preset-env
@babel/preset-typescript
@babel/preset-react
@babel/plugin-proposal-class-properties
@babel/plugin-proposal-object-rest-spread
```

- `@babel/core`毋庸置疑，babel的核心模块，实现了上述的流程运转以及代码语法、语义分析的功能；

- `@babel/cli`则是我们可以在命令行使用babel命令；

- plugin开头的就是插件，这里我们引入了两个：`@babel/plugin-proposal-class-properties`（[允许类具有属性](https://babel.docschina.org/docs/en/babel-plugin-proposal-class-properties/)）和`@babel/plugin-proposal-object-rest-spread`（[对象展开](https://babel.docschina.org/docs/en/7.0.0/babel-plugin-proposal-object-rest-spread/)）；

- preset开头的就是预置组件包合集，其中`@babel/preset-env`表示使用了可以根据实际的浏览器运行环境，会选择相关的转义插件包，通过配置得知目标环境的特点只做必要的转换。如果不写任何配置项，env 等价于 latest，也等价于 es2015 + es2016 + es2017 三个相加(不包含 stage-x 中的插件)；`@babel/preset-typescript`会处理所有ts的代码的语法和语义规则，并转换为js代码。

关于babel编译ts，并不是所有的语法都支持，这里有一篇文章专门介绍了其中注意点：《TypeScript 和 Babel：美丽的结合》。

# webpack项目级TS使用

前面的内容，我们已经介绍了将ts编译为js的两种方式（tsc、babel），但仅仅是简单将一个index.ts编译为index.js。实际上，对于项目级别的ts项目，还有很多需要了解的。接下来基于一个webpack项目来逐步介绍如何基于前文的两种方式来使用ts。

对于webpack来说，至少需要读者了解到webpack的基本机制：[概念 | webpack 中文文档 (docschina.org)](https://webpack.docschina.org/concepts/)。

简单来讲，webpack运行从指定的entry文件开始，从顶层开始分析依赖的内容，依赖的内容可以是任何的内容（只要是import的或require了的），而loader可以专门来处理各种类型的文件。

>webpack 只能理解 JavaScript 和 JSON 文件，这是 webpack 开箱可用的自带能力。**loader** 让 webpack 能够去处理其他类型的文件，并将它们转换为有效 [模块](https://webpack.docschina.org/concepts/modules)，以供应用程序使用，以及被添加到依赖图中

![030-webpack-base-flow](https://static-res.zhen.wang/images/post/2022-08-14/030-webpack-base-flow.png)

所以，当一个webpack项目是基于TS进行的时候，我们一定会有一个loader来处理ts（甚至是tsx）。当然，我们还是通过demo搭建来演示讲解。

## ts-loader

```shell
mkdir webpack-ts-loader-demo && cd webpack-ts-loader-demo
yarn init
yarn add -D webpack webpack-cli
yarn add -D ts-loader
```

**package.json**

```json
{
  "name": "webpack-ts-loader-demo",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "build": "webpack --config webpack.config.js"
  },
  "devDependencies": {
    "ts-loader": "^9.3.1",
    "webpack": "^5.74.0",
    "webpack-cli": "^4.10.0"
  }
}
```

**webpack.config.js**

```js
const {resolve} = require('path');
module.exports = {
  entry: './src/index.ts',
  output: {
    path: resolve(__dirname, './dist'),
    filename: "index.js"
  },
  module: {
    rules: [
      {
        test: /\.ts/,
        loader: "ts-loader"
      }
    ]
  }
};
```

**src/index.ts**

```ts
interface User {
    name: string;
    age: number;
}
const userToString = (user: User) => `${user.name}@${user.age}`;
export {userToString, User};
```

表面上，只需要上述三个文件，就可以编译ts文件，但是尝试运行`yarn build`会报错：

```
Module build failed (from ./node_modules/ts-loader/index.js):
Error: Could not load TypeScript. Try installing with `yarn add typescript` or `npm install typescript`. If TypeScript is installed globally, try using `yarn link typescript` or `npm link typescript`.
```

通过报错很容易理解，我们没有安装typescript。为什么？**因为ts-loader本身处理ts文件的时候，本质上还是调用的tsc，而tsc是typescript模块提供的**。因此，我们只需要`yarn add -D typescript`即可（其实只需要开发依赖即可），但是紧接着又会有另外一个报错：

```
ERROR in ./src/index.t
Module build failed (from ./node_modules/ts-loader/index.js):
Error: error while parsing tsconfig.json
```

报错提醒我们，解析tsconfig的出错，不难理解，我们还没有配置tsconfig.json，因为tsc需要！所以，在我们项目中，加上tsconfig.json即可：

**tsconfig.json**

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist"
  }
}
```

配置完成以后，我们再次编译，发现可以编译成功，并且在dist目录下会有对应的js代码。

然而，事情到这里就结束了吗？一个中大型的项目，必然有模块的引入，假如现在我们添加了个utils.ts

```ts
export const hello = () => {
  return 'hello';
}
```

修改index.ts的代码，引入该hello方法，并使用：

```typescript
import {hello} from "./utils"; // 引入utils
interface User {
    name: string;
    age: number;
}
const userToString = (user: User) => `${user.name}@${user.age}${hello()}`;
export {userToString, User};
```

再次运行`yarn build`，读者会发现还是会报错，但这一次的错误略有点出乎意料：

```
Module not found: Error: Can't resolve './utils' in '/Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/src'
resolve './utils' in '/Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/src'
```

核心报错在于，webpack似乎无法找到utils这个模块。为什么呢？因为webpack默认是处理js代码的，如果你的代码中编写了`import xxx from 'xxx'`，在没有明确指明这个模块的后缀的时候，webpack只会认为这个模块是以下几种：

1. 无后缀文件
2. js文件
3. json文件
4. wasm文件

所以，你会看到具体一点的报错：

```
resolve './utils' in '/Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/src'
  using description file: /Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/package.json (relative path: ./src)
    Field 'browser' doesn't contain a valid alias configuration
    using description file: /Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/package.json (relative path: ./src/utils)
      no extension
        Field 'browser' doesn't contain a valid alias configuration
        /Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/src/utils doesn't exist
      .js
        Field 'browser' doesn't contain a valid alias configuration
        /Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/src/utils.js doesn't exist
      .json
        Field 'browser' doesn't contain a valid alias configuration
        /Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/src/utils.json doesn't exist
      .wasm
        Field 'browser' doesn't contain a valid alias configuration
        /Users/w4ngzhen/Projects/web-projects/webpack-ts-loader-demo/src/utils.wasm doesn't exist
      as directory
```

要想让webpack知道我们引入的utils是ts代码，方式为在webpack配置中，指明webpack默认处理的文件后缀：

```js
const {resolve} = require('path');
module.exports = {
  // ... ...
  resolve: {
    // webpack 默认只处理js、jsx等js代码
    // 为了防止在import其他ts代码的时候，出现
    // " Can't resolve 'xxx' "的错误，需要特别配置
    extensions: ['.js', '.jsx', '.ts', '.tsx']
  },
  // ... ...
};

```

完成配置以后，我们就能够正确编译具备模块导入的ts代码了。

综合来看，在基于ts-loader的webpack项目的解析流程处理如下。

![040-webpack-ts-loader-flow](https://static-res.zhen.wang/images/post/2022-08-14/040-webpack-ts-loader-flow.png)

回顾一下webpack，它默认处理模块化js代码，比如index.js引用了utils.js（模块引用方式可以是commonjs，也可以是esModule形式），那么webpack从入口的index.js出发，来处理依赖，并打包为一个js（暂不考虑js拆分）。

对于wepack+ts-loader的ts项目体系主要是通过ts-loader内部调用typescript提供的tsc，将ts代码编译为js代码（编译后的js代码依然是js模块化的形式），所以这个过程是需要tsconfig参与；等到tsc将整个所有的ts代码均编译为js代码以后，再整体交给webpack进行依赖分析并打包（也就进入webpack的默认处理流程）。

细心的读者会发现这个过程有一个问题：由于先经过tsc编译后的js，又再被webpack默认的js处理机制进行分析并编译打包，这个过程一方面经过了两次编译（ts->标准模块化js->webpack模块体系js），那么如果ts项目特别大，模块特别多的时候，这个两次编译的过程会特别漫长！

## babel-loader

前面我们简单介绍了如何使用babel对一份ts进行编译，那么在webpack中，如何使用babel呢？有的同学可能会想到这样操作步骤：我先用babel对ts进行编译为js，然后再利用webpack对js进行打包，这样的做法是可以的，但细想不就和上面的ts-loader一样的情况了吗？

只要开发过基于webpack的现代化前端项目的同学，或多或少都看到过babel-loader的身影，他是个什么东西呢？先说结论吧，babel-loader是webpack和babel（由@babel/core和一堆预置集preset、插件plugins组合）的桥梁。

![050-webpack-babel-loader-flow](https://static-res.zhen.wang/images/post/2022-08-14/050-webpack-babel-loader-flow.png)

根据这个图，同学可能觉得这不是和ts-loader的架构很像吗？webpack启动，遇到入口ts，匹配到babel-loader，babel-loader交给babel处理，处理完毕，回到webpack打包。但是使用babel进行ts处理，比起ts-loader更加高效。而关于这块的说明，我更加推荐读者阅读这篇文章[ TypeScript 和 Babel：美丽的结合 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/59614089)，简单来讲：

>警告！有一个震惊的消息，你可能想坐下来好好听下。
>
>Babel 如何处理 TypeScript 代码？**它删除它**。
>
>是的，它删除了所有 TypeScript，将其转换为“常规的” JavaScript，并继续以它自己的方式愉快处理。
>
>这听起来很荒谬，但这种方法有两个很大的优势。
>
>第一个优势：️⚡️**闪电般快速**⚡️。
>
>大多数 Typescript 开发人员在开发/监视模式下经历过编译时间长的问题。你正在编写代码，保存一个文件，然后...它来了...再然后...**最后**，你看到了你的变更。哎呀，错了一个字，修复，保存，然后...啊。它**只是**慢得令人烦恼并打消你的势头。
>
>很难去指责 TypeScript 编译器，它在做很多工作。它在扫描那些包括 `node_modules` 在内的类型定义文件（`*.d.ts`），并确保你的代码正确使用。这就是为什么许多人将 Typescript 类型检查分到一个单独的进程。然而，Babel + TypeScript 组合仍然提供更快的编译，这要归功于 Babel 的高级缓存和单文件发射架构。

让我们来搭建一个项目来复习这一过程吧：

```
mkdir webpack-babel-loader-demo && cd webpack-babel-loader-demo
yarn init
yarn add -D webpack webpack-cli
yarn add -D babel-loader
yarn add -D @babel/core
yarn add -D @babel/preset-env @babel/preset-typescript
yarn add -D @babel/plugin-proposal-class-properties @babel/plugin-proposal-object-rest-spread
```

**package.json**

```json
{
  "name": "webpack-babel-loader-demo",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "build": "webpack --config webpack.config.js" 
  },
  "devDependencies": {
    "@babel/core": "^7.18.13",
    "@babel/plugin-proposal-class-properties": "^7.18.6",
    "@babel/plugin-proposal-object-rest-spread": "^7.18.9",
    "@babel/preset-env": "^7.18.10",
    "@babel/preset-typescript": "^7.18.6",
    "babel-loader": "^8.2.5",
    "webpack": "^5.74.0",
    "webpack-cli": "^4.10.0"
  }
}
```

**webpack.config.js**

```js
const {resolve} = require('path');
module.exports = {
  entry: './src/index.ts',
  output: {
    path: resolve(__dirname, './dist'),
    filename: "index.js"
  },
  resolve: {
    // webpack 默认只处理js、jsx等js代码
    // 为了防止在import其他ts代码的时候，出现
    // " Can't resolve 'xxx' "的错误，需要特别配置
    extensions: ['.js', '.jsx', '.ts', '.tsx']
  },
  module: {
    rules: [
      {
        test: /\.ts/,
        loader: "babel-loader"
      }
    ]
  }
};

```

**src/index.ts**

```ts
import {hello} from "./utils";
interface User {
    name: string;
    age: number;
}
const userToString = (user: User) => `${user.name}@${user.age}${hello()}`;
export {userToString, User};
```

**src/utils.ts**

```ts
export const hello = () => {
  return 'hello';
}
```

完成上述package.json、webpack.config.js、src源代码三个部分，我们可以开始运行`yarn build`，但实际上会报错：

```
ERROR in ./src/index.ts
Module build failed (from ./node_modules/babel-loader/lib/index.js):
SyntaxError: /Users/w4ngzhen/Projects/web-projects/webpack-babel-loader-demo/src/index.ts: Unexpected reserved word 'interface'. (1:0)

> 1 | interface User {
    | ^
  2 |     name: string;
  3 |     age: number;
  4 | }
    at instantiate (/Users/w4ngzhen/Projects/web-projects/webpack-babel-loader-demo/node_modules/@babel/parser/lib/index.js:72:32)
```

出现了语法的错误，报错的主要原因在于没有把整个babel处理ts的链路打通。目前的链路是：webpack找到入口ts文件，匹配上babel-loader，babel-loader交给@babel/core，@babel/core处理ts。由于我们没有给@babel/core配置plugin、preset，所以导致了babel还是以默认的js角度来处理ts代码，所以有语法报错。此时，我们需要添加.babelrc文件来指明让babel加载处理ts代码的插件：

**.babelrc**

```json
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-typescript"
  ],
  "plugins": [
    "@babel/plugin-proposal-object-rest-spread",
    "@babel/plugin-proposal-class-properties"
  ]
}
```

完成配置以后，我们再次运行`yarn build`，编译通过，但是在dist下的index.js却是空白的！

### 问题：babel-loader编译后，输出js内容空白

如果按照上述的配置以后，我们能够成功编译但是却发现，输出的js代码是空白的！原因在于：我们编写的js代码，是按照类库的模式进行编写（在indexjs中只有导出一些函数却没有实际的使用），且webpack打包的时候，没有指定js代码的编译为什么样子的库。

假如我们在index中编写一段具有副作用的代码：

```typescript
import {hello} from "./utils";
interface User {
    name: string;
    age: number;
}
const userToString = (user: User) => `${user.name}@${user.age}${hello()}`;

// 具备副作用：在id=app的元素上添加监听
document
    .querySelector('#app')
    .addEventListener('click', () => {})

export {userToString, User};
```

此时我们使用生产模式（mode: 'production'）来编译，会发现dist/index.js的内容如下：

```js
(() => {
  "use strict";
  document.querySelector("#app").addEventListener("click", (function () {
  }));
})();
```

会发现只有副作用代码，但是userToString相关的代码完全被剔除了！这时候，可能有读者会说，我导出的代码有可能别人会使用，你凭什么要帮我剔除？其实，因为webpack默认是生成项目使用的js，也就是做打包操作，他的目的是生成当前项目需要的js。在我们这个示例中，在没有写副作用之前，webpack认为打包是没有意义的，因为只有导出方法，却没有使用。那么，如果让webpack知道，我们需要做一个类库呢？在webpack中配置library字段即可：

```js
const {resolve} = require('path');
module.exports = {
  entry: './src/index.ts',
  mode: 'production',
  output: {
    // ... ...
    library: { 
      // 配置library字段的相关配置，这里我们配置为commonjs2
      // 至于这块配置的意义，读者需要自行学习～
      type: 'commonjs2',
    },
  },
  // ... ...
};

```

# tsc与babel编译的差异

现在我们先编写一个简单**错误代码**：

```typescript
interface User {
    name: string;
    age: number;
}
// user.myName并没有在User接口中提供
const userToString = (user: User) => `${user.myName}@${user.age}`;
export {userToString, User};
```

在这个示例中，我们试图访问在User类型中不存在的myName字段。

## ts-loader

前面我们提到了ts-loader内部调用的是tsc作为编译器，我们尝试运行基于ts-loader的webpack配置进行打包该模块，会发现报错：

```
... ...
      TS2551: Property 'myName' does not exist on type 'User'. Did you mean 'name'?
ts-loader-default_e3b0c44298fc1c14

webpack 5.74.0 compiled with 1 error in 2665 ms
error Command failed with exit code 1.
info Visit https://yarnpkg.com/en/docs/cli/run for documentation about this command.
```

可以看得出来，tsc帮助我们提示了类型错误的地方，user这个类型并没有对应的myName字段。

## babel-loader

我们切换一下到babel-loader对该ts文件进行编译，居然发现编译可以直接成功！并且，我们检查编译好的js代码，会发现这部分：

```js
// dist/index.js
(() => {
  "use strict";
  // ... ...
  var r = function (e) {
    // 注意这个地方：依然在使用myName
    return "".concat(e.myName, "@").concat(e.age);
  };
  module.exports = o;
})();

```

编译好的js代码就在直接使用myName字段。为什么类型检查失效了？还记得我们前面提到的babel怎么处理ts的？

>Babel 如何处理 TypeScript 代码？**它删除它**。
>
>是的，它删除了所有 TypeScript，将其转换为“常规的” JavaScript，并继续以它自己的方式愉快处理。

是的，babel并没有进行类型检查，而是将各种类型移除掉以达到快速完成编译的目的。那么问题来了，我们如何让babel进行类型判断呢？**实际上，我们没有办法让babel进行类型判断，必须要借助另外的工具进行。**那为什么我们的IDE却能够现实ts代码的错误呢？因为IDE帮助我们进行了类型判断。

# 主流IDE对TypeScript的类型检查

不知道有没有细心的读者在使用IDEA的时候，发现一个ts项目的IDEA右下角展示了typescript：

![060-idea-ts-service](https://static-res.zhen.wang/images/post/2022-08-14/060-idea-ts-service.png)

VSCode也能看到类似：

![070-vscode-ts-service](https://static-res.zhen.wang/images/post/2022-08-14/070-vscode-ts-service.png)

在同一台电脑上，甚至发现IDEA和VSCode的typescript版本都还不一样（4.7.4和4.7.3）。这是怎么一回事呢？实际上，IDE检测到你所在的项目是一个ts项目的时候（或包含ts文件），就会自动的启动一个ts的检测服务，专门用于所在项目的ts类型检测。这个ts类型检测服务，是通过每个IDE默认情况下自带的typescript中的tsc进行类型检测。

但是，我们可以全局安装（npm -g）或者是为每个项目单独安装typescript，然后就可以让IDE选择启动独立安装的typescript。比如，我们在本项目中，安装一个特定版本的ts（版本4.7.2）：

```
yarn add -D typescript@4.7.2
```

在IDEA中，设置 - Languages & Frameworks - TypeScript中，就可以选择IDEA启动的4.7.2版本的TypeScript为我们项目提供类型检查（注意看选项中有一个Bundled的TS，版本是4.7.4，就是默认的）：

![080-idea-select-ts](https://static-res.zhen.wang/images/post/2022-08-14/080-idea-select-ts.png)

IDE之所以能够在对应的代码位置展示代码的类型错误，流程如下：

![090-ide-ts-service-flow](https://static-res.zhen.wang/images/post/2022-08-14/090-ide-ts-service-flow.png)

但是，ts类型检查也要有一定的依据。譬如，有些类型定义的文件从哪里查找，是否允许较新的语法等，这些配置依然是由tsconfig.json来提供的，但若未提供，则IDE会使用一份默认的配置。如果要进行类型检测的自定义配置，则需要提供tsconfig.json。

还记得我们前面的ts-loader吗？在代码编译期，ts-loader调用tsc，tsc读取项目目录下的tsconfig.json配置。而咱们编写代码的时候，又让IDE的ts读取该tsconfig.json配置文件进行类型检查。

对于ts-loader项目体系来说，ts代码编译和ts的类型检测如下：

![100-ts-loader-and-ide](https://static-res.zhen.wang/images/post/2022-08-14/100-ts-loader-and-ide.png)

然而，对于babel-loader项目体系就不像ts-loader那样了：

![110-babel-loader-and-ide](https://static-res.zhen.wang/images/post/2022-08-14/110-babel-loader-and-ide.png)

在babel-loader体系中，代码的编译只取决于babel部分的处理，根类型没有根本的关系，而类型检查使用到的tsconfig和tsc则只作用在类型检查的部分，根ts代码编译没有任何关系。