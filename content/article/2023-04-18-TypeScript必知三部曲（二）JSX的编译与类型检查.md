---
title: TypeScript必知三部曲（二）JSX的编译与类型检查
date: 2023-04-18
tags:
 - typescript
 - jsx
categories:
  - 技术
  - TypeScript必知三部曲 
---

在本三部曲系列的第一部中，我们介绍了TypeScript编译的两种方案(tsc编译、babel编译）以及二者的重要差异，同时分析了IDE是如何对TypeScript代码进行类型检查的。该部分基本涵盖了TypeScript代码编译的细节，但主要是关于TS代码本身的编译与类型检查。而本文，我们将着重讨论含有JSX的TypeScript代码（又称TSX）如何进行类型检查与代码编译的。

<!-- more -->

# 前言：JSX编译

在介绍如何对JSX代码进行类型检查前，让我们花一点时间认识一下JSX，以及如何对其进行编译。

>注意：这块内容很多，如果读者已经熟悉这块的内容，可以直接从**JSX（TSX）的类型检查**开始阅读。

实际上，JSX并不是合法有效的JS代码或HTML代码。目前为止也没有任何一家浏览器的引擎实现了对JSX的读取和解析。此外，JSX本身没有完全统一的规范，除了一些基本的规则以外，各种利用了JSX的JS库可以根据自身需求来设计JSX额外的特性。譬如，React中的元素会有className属性，而SolidJS中的元素会有classList属性。在FaceBook官方博文中也明确提到了：

> JSX是一种类似XML的语法扩展。它不打算由引擎或浏览器实现。它也不会作为某种提案被合并到ECMAScript规范中。它旨在被各种预处理器（转译器）用于将这些标记转换为标准的ECMAScript。—— [JSX (facebook.github.io)](https://link.zhihu.com/?target=https%3A//facebook.github.io/jsx/%23sec-intro)

当然，只要提到JSX我们就不得不提React，尽管React与JSX是相互独立的东西，但是React将JSX发扬光大，让更多的开发者接触到了JSX。所以我们先从React入手，分析JSX是如何编译为JS代码的。对于JSX的编译方案，已知的有两种：

1. babel编译方案
2. tsc编译方案

就像TypeScript编译一样，只要涉及到了编译环节，我们总是离不开编译三要素模型：源代码、编译器以及编译配置：

![010-code-compile-flow](https://static-res.zhen.wang/images/post/2023-04-18/010-code-compile-flow.png)

接下来将分别详细介绍这两种编译体系的编译过程。

## babel编译体系

通过babel可以将结构化的JSX组件，转换为同样结构化的JS代码调用形式。在React中，转换JSX为原生JS代码分为两种形式：

1. React17**以前**的`React.createElment`形式；
2. React17**以后**的`'react/jsx-runtime'`形式。

先讲第一种：直接转换为`React.createElement`。假设源代码如下：

```tsx
import React from 'react';

function App() {
  return <h1>Hello World</h1>;
}
```

转换过程，会将上述JSX转换为如下的`createElement`代码：

```js
import React from 'react';

function App() {
  return React.createElement('h1', null, 'Hello world');
}
```

但官方提到了关于这种转换方式的两个问题：

- 如果使用 JSX，则需在 `React` 的环境下，因为 JSX 将被编译成 `React.createElement`，也就是说强绑定`React`。
- 有一些 `React.createElement` 无法做到的[性能优化和简化](https://link.zhihu.com/?target=https%3A//github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md%23motivation)。

基于上述的问题，在React17以后，提供了另一种转换方式：**引入jsx-runtime层**。假设源码如下：

```jsx
function App() {
  return <h1>Hello World</h1>;
}
```

下方是新 JSX 被转换编译后的结果：

```js
// 由编译器引入（禁止自己引入！）
import {jsx as _jsx} from 'react/jsx-runtime';

function App() {
  return _jsx('h1', { children: 'Hello world' });
}
```

第二种模式的核心在于：JSX编译出来的代码与React库本身进行了解耦，只将JSX转换为了与React无关的JS形式的调用描述，没有直接使用`React.createElement`。**引入了jsx-runtime这一层，屏蔽具体的调用细节，只专注JSX到JS代码最基础的映射。**至于这个`_jsx`的具体实现，就是内部调用的是`React.createElement`还是另一种`createElement`，则可以由库内部来进行实现。

![020-react-jsx-runtime](https://static-res.zhen.wang/images/post/2023-04-18/020-react-jsx-runtime.png)

> PS：可能有小伙伴会说，_jsx不还是从`react/jsx-runtime`这个React相关库导出的吗？实际上，这个包仅仅是由react团队在维护的原因。

上图描述了一个前端React工程里JSX代码基本的转换思路。当然，Babel在这个转换过程中承担了重要角色。在Babel中，与上述两种转换相关的核心部分是：`@babel/preset-react`里面引用的插件`@babel/plugin-transform-react-jsx`。

Babel`v7.9.0`版本之前的该插件，只能将JSX代码转换为`React.createElement`调用形式。而在v7.9.0版本以后，支持我们配置转换行为。默认选项为 `{"runtime": "classic"}`，也就是说默认还是`React.createElement`。

如需启用新的转换，你可以使用 `{"runtime": "automatic"}` 作为 `@babel/plugin-transform-react-jsx` 或 `@babel/preset-react` 的选项：

```json
// 如果你使用的是 @babel/preset-react（内部引用了@babel/plugin-transform-react-jsx）
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "automatic"
    }]
  ]
}
// 如果你使用的是 @babel/plugin-transform-react-jsx
{
  "plugins": [
    ["@babel/plugin-transform-react-jsx", {
      "runtime": "automatic"
    }]
  ]
}
```
让我们创建一个样例`jsx-babel-example`，来实践上述过程。对应编译模型三要素，我们定义好如下的内容：

（1）源代码：src/index.jsx

```jsx
const MyButton = (props) => <button>{props.children}</button>

function App() {
    return <h1><MyButton>Hello World</MyButton></h1>;
}
```

（2）babel编译器以及相关插件：

```bash
yarn add -D @babel/core @babel/cli
yarn add -D @babel/plugin-transform-react-jsx
```

（3）编译配置.babelrc：

```json
{
  "plugins": [
    [
      "@babel/plugin-transform-react-jsx"
    ]
  ]
}
```

补充一个脚本：

```diff
{
  ...
+ "scripts": {
+ 	 "build": "babel src -x .jsx --config-file ./.babelrc -d dist"
+ },
}
```

搭建完成以后，整体如下：

![030-jsx-babel-compile-project](https://static-res.zhen.wang/images/post/2023-04-18/030-jsx-babel-compile-project.png)

项目结构并不复杂，编译的过程我们也不再赘述。当我们运行`yarn build`的时候可以看到在dist目录下能够生成对应js代码：

![040-jsx-babel-compile-result](https://static-res.zhen.wang/images/post/2023-04-18/040-jsx-babel-compile-result.png)

从上图可以看到，我们的代码直接转换为了`React.createElement`。但是注意的是，编译结果中，babel是没有替我们插入`import React from 'react'`这一句代码的！如果你的代码本身没有添加`import React from 'react'`，那么最终编译到了js代码（无论是commonjs还是esmodule），也不会引入React，然而代码却调用的是：`React.createElement`。正是因为如此，所以才会有我们日常小伙伴会发现，项目能够编译通过，但是运行起来的时候，会提示：

***ReferenceError: React is not defined***

对于上面问题的解决办法，有两种方式解决：

方式一：在你的代码中手动写上：`import React from 'react'`。编译后，自然而然就有了`import React from 'react'`。

方式二：使用编译参数：`"runtime": "automatic"`：

```diff
{
  "plugins": [
    [
      "@babel/plugin-transform-react-jsx",
+     { "runtime": "automatic" } 
    ]
  ]
}
```

新增上述配置以后，重新编译代码，能够看到生产的js代码：

![050-jsx-babel-compile-with-automatic](https://static-res.zhen.wang/images/post/2023-04-18/050-jsx-babel-compile-with-automatic.png)

对于重新编译好的代码，此时可以看到`React.createElement`调用变为了来源于`"react/jsx-runtime"`中的`jsx`方法。同时，由于这一段引入是由**编译器自动加入**的，因此代码进行后续的babel编译的时候，由于有`react/jsx-runtime`的引入，所以就不再会有所谓的`React is not defined`的问题。

## tsc编译体系

介绍完babel编译jsx体系以后，我们再讲一下关于tsc编译jsx代码的方式。当然，基于编译要素模型，我们依然准备一个项目来解释这个过程。

（1）源代码同上src/index.jsx。

（2）typescript包：

```bash
yarn add -D typescript
```

（3）编译配置tsconfig.json：

```json
{
  "compilerOptions": {
    "jsx": "react",
    "outDir": "dist",
    "rootDir": "src",
    "allowJs": true
  }
}
```

补充一个脚本：

```diff
{
  ...
  "scripts": {
+ 	"build": "tsc -p tsconfig.json"
  },
  ...
}

```

搭建完成后，整体如下：

![060-jsx-tsc-compile-project](https://static-res.zhen.wang/images/post/2023-04-18/060-jsx-tsc-compile-project.png)

当我们运行`yarn build`的时候可以看到在dist目录下能够生成对应js代码：

![070-jsx-tsc-compile-result-by-jsxreact](https://static-res.zhen.wang/images/post/2023-04-18/070-jsx-tsc-compile-result-by-jsxreact.png)

读者应该能够看到由tsc编译出来的js代码，JSX相关的代码编译为了`React.createElement`形式的调用。对于这个编译结果，tsconfig.json里面的配置起到了决定性作用。现在，我们着重讲一下两个配置项：`jsx`、`allowJs`。

`"allowJs"`

由于本example中我们没有编写tsx代码，还是用的jsx代码，如果不配置`"allowJs": true`，那么tsc编译器默认将不会处理js以及jsx文件，又因为example中src目录下只有jsx文件，于是会出现报错：

```
error TS18003: No inputs were found in config file '/Users/w4ngzhen/projects/web-projects/jsx-tsc-example/tsconfig.json'. Specified 'include' paths were '["**/*"]' and 'exclude' paths were '["dist"]'.
```

后续如果是TSX的文件，将不会出现这个问题，也不用显式配置该选项。

`"jsx"`

对于`"jsx"`这个配置，主要有以下几个值：

- `react`: 将 JSX 改为等价的对 `React.createElement` 的调用并生成 `.js` 文件。
- `react-jsx`: 改为 `__jsx` 调用并生成 `.js` 文件。
- `preserve`: 不对 JSX 进行改变并生成 `.jsx` 文件。
- `react-jsxdev`: 改为 `__jsx` 调用并生成 `.js` 文件。
- `react-native`: 不对 JSX 进行改变并生成 `.js` 文件。

下图展示了当`"jsx"`的配置分别为：`"react"`、`"react-jsx"`的结果：

![080-tsconfig-jsx-result](https://static-res.zhen.wang/images/post/2023-04-18/080-tsconfig-jsx-result.png)

不难发现，`"react"`与`"react-jsx"`配置的编译结果，与前面babel编译中插件`@babel/plugin-transform-react-jsx`的`"runtime"`的配置分别为`"classic"`（或默认不配置）、`"automatic"`的编译结果能够相对应。

>tsconfig默认使用commonjs作为模块化方案，所以，`"jsx": "react-jsx"`配置的编译结果中引用`react/jsx-runtime`时，使用commonjs规范的`require`。如果给tsconfig.json添加配置`"module": "ES6"`，则会看到`import {jsx as __jsx} from 'react/jsx-runtime'`的引用方式。

# 正文：JSX（TSX）的类型检查

在《2023-04-08-TypeScript必知三部曲（一）TypeScript编译方案以及IDE对TS的类型检查》中，我们已经了解了，babel不会参与TS代码的类型检查，TS代码本身的类型检查、IDE上的类型检查提示，都是经过tsc配合tsconfig配置完成。所以，接下来我们所谈的关于JSX（TSX）的类型检查，将会围绕tsc+tsconfig来进行讨论。

## 准备工作

在进行讨论之前，我们依然准备一个样例，这个样例与前面关于tsc编译体系的样例差别不大，**重点在于index.jsx改为了index.tsx**：

（1）源代码src/index.tsx：

```tsx
const MyButton = (props) => <button>{props.children}</button>

export function App() {
    return <h1><MyButton>Hello World</MyButton></h1>;
}
```

（2）tsconfig.json：

```json
{
  "compilerOptions": {
    "module": "ES6",
    "jsx": "react",
    "outDir": "dist",
    "rootDir": "src",
    "allowJs": true
  }
}
```

**注意`"jsx"`的配置我们使用`"react"`。**

（3）安装typescript并添加编译脚本：

```json
{
  "name": "jsx-tsc-example",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
  "scripts": {
    "build": "tsc -p tsconfig.json"
  },
  "devDependencies": {
    "typescript": "^5.0.4"
  }
}
```

准备后整体如下：

![090-tsx-tsc-example-project](https://static-res.zhen.wang/images/post/2023-04-18/090-tsx-tsc-example-project.png)

## 类型问题

在上述的项目搭建完成后，我们会发现一个问题：在index.tsx代码中，我们用到了两个jsx基础标签：`<button>`、`<h1>`以及我们自己编写的React组件`<MyButton>`，但IDE替我们显示了红色，鼠标悬浮以后，会看到报错提示：

![100-tsx-jsxreact-error-result](https://static-res.zhen.wang/images/post/2023-04-18/100-tsx-jsxreact-error-result.png)

- **Cannot find name 'React'.**

为什么会这样呢？针对该问题，需要两个知识点来解释。

第一，tsconfig.json的`"jsx": "react"`配置的编译结果是将 JSX 改为等价的对 `React.createElement` 的调用并生成 `.js` 文件；

第二，IDE进行TS的类型检查流程如下：

![110-IDE-ts-check-flow](https://static-res.zhen.wang/images/post/2023-04-18/110-IDE-ts-check-flow.png)

基于上述两点，我们可以解释这个出错的过程为：IDE识别到了tsconfig.json中的`"jsx": "react"`配置，调用了形如`tsc --noEmit`的指令，又因为我们的项目没有添加对`react`的依赖（类型文件也没有），因此出现了这个地方的IDE报错提示。不难想到，我们实际运行脚本进行编译的时候，会出现同样的错误：

![120-tsx-jsxreact-error-result-in-cli](https://static-res.zhen.wang/images/post/2023-04-18/120-tsx-jsxreact-error-result-in-cli.png)

> 细心的小伙伴会看到dist目录下依然生成了index.js代码，因为类型检查结果实际上不妨碍实际js代码的生成。

上面的配置，我们使用了`"jsx": "react"`，当我们修改为`"jsx": "react-jsx"`又会有什么效果呢？

修改配置以后，报错如下：

![130-tsx-jsxreactjsx-error-result](https://static-res.zhen.wang/images/post/2023-04-18/130-tsx-jsxreactjsx-error-result.png)

有两方面：

- Cannot find module 'react/jsx-runtime'。

- Did you mean to set the 'moduleResolution' option...。

当我们将`"moduleResolution": "node"`添加以后可以解决问题2，剩下的问题就是：

![140-tsx-jsxreactjsx-error-result-with-moduleResolution-Node](https://static-res.zhen.wang/images/post/2023-04-18/140-tsx-jsxreactjsx-error-result-with-moduleResolution-Node.png)

- **Cannot find module 'react/jsx-runtime' or its corresponding type declarations.**

> 无法找到模块`react/jsx-rutnime`或它对应的类型声明。

对照前面的`"jsx": "react"`，当我们的配置改为了`"jsx": "react-jsx"`以后，JSX标签都将编译为`_jsx("div", ..., ...)`的调用形式，而这个`_jsx`来源于：`import {jsx as _jsx} from 'react/jsx--runtime'`，而我们的项目并没有安装这个包。所以，IDE根据`react-jsx"`配置的结果，识别到了问题，并帮助我们提示了对应的问题。

无论是`Cannot find name 'React'.`还是`Cannot find module 'react/jsx-runtime' or its corresponding type declarations.`问题，要处理起来也很简单，安装react以及其类型定义：

```shell
yarn add react
yarn add -D @types/react
```

`@types/react`中包含了`React`和`react/jsx-runtime`类型定义。

![150-tsx-jsxreactjsx-error-result-import-react](https://static-res.zhen.wang/images/post/2023-04-18/150-tsx-jsxreactjsx-error-result-import-react.png)

终于，我们不再有类型问题了。此时，我们看看这些h1、button标签到底是什么类型以及ts是如何对这些JSX标签进行类型定义的。

在安装了`@types/react`后，IDEA里面，通过CTRL+鼠标左键点击相关的标签就能进入到对应的定义里面，比如我们查看`<a>`标签的具体定义：

![160-jsx-tag-dts](https://static-res.zhen.wang/images/post/2023-04-18/160-jsx-tag-dts.png)

通过查看类型定义dts文件，可以很容易的看到该类型为：`JSX.IntrinsicElements.a`。这个`JSX.IntrinsicElements`接口是什么呢？让我们探索。

## 类型检查的源头：JSX.IntrinsicElements

> Intrinsic：固有的，本质的，根本的。

内在元素（IntrinsicElements）在特殊接口（既`JSX.IntrinsicElements`接口）上查找。 默认情况下，如果未指定此接口，则在TypeScript进行类型检查的时候，会直接忽略这些类型JSX标签具体的类型定义，任何JSX都不会对内部元素进行类型检查。 但是，如果存在此接口定义，则内部元素的名称将作为接口上的属性进行查找。

举一个简单的例子，我们可以尝试修改上图中react的dts代码，添加一个新的接口字段abc，该字段还有一个必填的`name`属性：

```diff
        interface IntrinsicElements {
+           abc: { name: string; children?: Element };
        }
```

![170-jsx-tag-dts-add-abc-example](https://static-res.zhen.wang/images/post/2023-04-18/170-jsx-tag-dts-add-abc-example.png)

于是，在代码中，我们就能使用这个`<abc>`标签，同时，如果不填写`name`字段的值，TS还会有类型检查异常，只有正确填写`name`属性才能通过类型检查：

![180-jsx-use-abc-tag](https://static-res.zhen.wang/images/post/2023-04-18/180-jsx-use-abc-tag.png)

同时，当我们检查编译后的代码，会发现无论是`"jsx": "react"`还是`"jsx": "react-jsx"`，关于我们使用的`<abc>`标签的部分，都变成了字符串`"abc"`的处理（这里只用tsc编译演示，babel是同样的结果，不再赘述）：

![190-jsx-use-abc-tag-compile-result](https://static-res.zhen.wang/images/post/2023-04-18/190-jsx-use-abc-tag-compile-result.png)

当然，我们还能编写自己的自定义组件，譬如上面示例里面的`<MyButton>`。`MyButton`是一个函数组件，满足React DTS文件里面的类型定义关于使用函数组件类型进行createElement的类型定义：

![200-func-comp-dts](https://static-res.zhen.wang/images/post/2023-04-18/200-func-comp-dts.png)

总结来讲，JSX（TSX）中关于**内置标签**的类型检查流程如下：

![210-intrinsic-elements-type-check-flow](https://static-res.zhen.wang/images/post/2023-04-18/210-intrinsic-elements-type-check-flow.png)



在前面，我们在react的官方dts中的`JSX.IntrinsicElements`添加了`abc`字段，所以我们才能编写`<abc>`标签并通过类型检查。但这种方式目前来讲，有个问题：非常不优雅，居然去修改react类型定义代码。那么，还有什么方式扩展JSX的内置标签元素呢？编写声明文件扩充即可：

![220-extend-intrinsic-elements](https://static-res.zhen.wang/images/post/2023-04-18/220-extend-intrinsic-elements.png)

上图中，我们主动声明了`JSX.IntrinsicElements`接口，并且向里面添加了`a-custom-tag`，于是，后面的tsx代码中我们就能使用`<a-custom-tag>`这个标签了。

但要注意的是，我们声明的种种类型，**只针对类型检查**。它仅仅保证了tsc在进行类型检查的正确性。而实际编译后的代码，因为会生成诸如：`React.createElement("a-custom-tag", ...)`或`_jsx('a-cutoms-tag', ...)`等调用的js代码。不难想到在实际运行过程中，React内部是无法处理这个所谓的`a-custom-tag`的“内置标签”的，它就不明白这个`"a-custom-tag"`是什么，所以在运行时一定会有错误。

在前言中，我们已经解释了如何将JSX编译为react、react/runtime的相关调用。那么，我们可以自定义处理JSX代码吗？当然可以，如果使用的是babel编译体系，则需要自己编写babel插件；如果是tsc编译体系，则需要自定义`jsxFactory`，像是solidjs，就有自己的babel插件（[babel-preset-solid - npm (npmjs.com)](https://www.npmjs.com/package/babel-preset-solid)）。本文不再赘述关于自定义JSX的编译过程了，网上有很多优秀的文章可以阅读。

# 写在最后

本文的内容其实还是有点杂乱，后续可能会重构这篇文章。
