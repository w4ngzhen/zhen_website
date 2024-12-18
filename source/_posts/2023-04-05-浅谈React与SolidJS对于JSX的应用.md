---
title: 浅谈React与SolidJS对于JSX的应用
date: 2023-04-05
tags:
 - jsx
 - babel
 - react
 - solidjs
categories:
  - 技术
---

React将JSX这一概念深入人心。但，并非只有React利用了JSX，VUE、SolidJS等JS库或者框架都使用了JSX这一概念。网上已经有大量关于JSX的概念与形式的讲述文章，不在本文的讨论范围。

<!-- more -->

# 前言

实际上，JSX并不是合法有效的JS代码或HTML代码。目前为止也没有任何一家浏览器的引擎实现了对JSX的读取和解析。此外，JSX本身没有完全统一的规范，除了一些基本的规则以外，各种**利用**了JSX的JS库可以根据自身需求来设计JSX额外的特性。譬如，React中的元素会有className属性，而SolidJS中的元素会有classList属性。

>在FaceBook官方博文中也明确提到了：
>
>JSX是一种类似XML的语法扩展。它不打算由引擎或浏览器实现。它也不会作为某种提案被合并到ECMAScript规范中。它旨在被各种预处理器（转译器）用于将这些标记转换为标准的ECMAScript。
>
>[JSX (facebook.github.io)](https://facebook.github.io/jsx/#sec-intro)

JSX的标签一定只有类似于HTML元素的标签吗？并不是这样的。比如，SolidJS中除了包含形如HTML的全部基础标签以外，还有一些控制标签，例如：`<For>`、`<Show>`等，它们是完全根据自身库的需要设计处理的标签。

回到更加现实的部分，浏览器总是基于HTML+JavaScript+CSS来完成前端的渲染的。前端领域中日新月异的库、框架绝大部分都逃离不了这三要素，JSX也包括在内。无论我们设计出来的JSX语法糖多么的“甜”，就现状来看，最终都或多或少的成为了HTML、JS或CSS中的某部分。

接下来，我们将进一步讨论各种前端框架是如何使用JSX的。

# React中的JSX

## 工程预编译JSX

React中使用JSX已经老生常谈了。简单来讲，通过编译器（一般都是babel）可以将结构化的JSX组件，转换为同样结构化的JS代码调用形式。在React中，转换JSX为原生JS代码分为两种形式：

1. React17**以前**的`React.createElment`形式；
2. React17**以后**的`'react/jsx-runtime'`形式。

先讲第一种：直接转换为`React.createElement`。假设源代码如下：

```jsx
import React from 'react';

function App() {
  return <h1>Hello World</h1>;
}
```

转换过程，会将上述JSX转换为如下的createElement代码：

```js
import React from 'react';

function App() {
  return React.createElement('h1', null, 'Hello world');
}
```

但官方提到了关于这种转换方式的两个问题：

- 如果使用 JSX，则需在 `React` 的环境下，因为 JSX 将被编译成 `React.createElement`。
- 有一些 `React.createElement` 无法做到的[性能优化和简化](https://github.com/reactjs/rfcs/blob/createlement-rfc/text/0000-create-element-changes.md#motivation)。

基于上述的问题，在React17以后，提供了另一种转换方式：引入jsx-runtime层。假设源码如下：

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

第二种模式的核心在于，编译出来的代码与React库本身进行了解耦，只将JSX转换为了与React无关的JS形式的调用描述，没有直接使用`React.createElement`。

![010-react-jsx-runtime](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/010-react-jsx-runtime.png)

上图描述了一个前端React工程里JSX代码转换为浏览器能够运行的JS代码的基本过程。当然，Babel在这个转换过程中承担了重要角色。

在Babel中，与上述两种转换相关的是部分是：`@babel/preset-react`（核心其实是该preset预置集内部的插件`@babel/plugin-transform-react-jsx`）。无论是`@babel/preset-react`还是`@babel/plugin-transform-react-jsx`，都允许我们配置上述的转换行为。

Babel的v7.9.0版本之前，只能转换为`React.createElement`形式。在v7.9.0版本以后，支持我们配置转换行为。默认选项为 `{"runtime": "classic"}`，也就是说默认还是`React.createElement`。

如需启用新的转换，你可以使用 `{"runtime": "automatic"}` 作为 `@babel/plugin-transform-react-jsx` 或 `@babel/preset-react` 的选项：

```json
// 如果你使用的是 @babel/preset-react
{
  "presets": [
    ["@babel/preset-react", {
      "runtime": "automatic"
    }]
  ]
}
```

```json
// 如果你使用的是 @babel/plugin-transform-react-jsx
{
  "plugins": [
    ["@babel/plugin-transform-react-jsx", {
      "runtime": "automatic"
    }]
  ]
}
```

## 浏览器使用JSX

首先，我们可以按照如下的方式，直接基于CDN模式的React进行开发：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<div id="app"></div>
<script src="https://unpkg.com/react@18.2.0/umd/react.production.min.js"></script>
<script src="https://unpkg.com/react-dom@18.2.0/umd/react-dom.production.min.js"></script>
<script>
    const appComp = React.createElement('div', {style: {color: 'blue'}}, 'hello, world');
    ReactDOM.createRoot(document.querySelector('#app')).render(appComp)
</script>
</body>
</html>
```

1. 调用`React.createElement`创建React节点实例；
2. 调用ReactDOM的API完成某个节点的渲染。

可以直接从页面上看到渲染效果：

![020-react-cdn-createElement](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/020-react-cdn-createElement.png)

这种方式则是最直接的，使用了最原生的写法。具备JS基础的同学应该都能理解。如果我们在script中编写了jsx代码：

```diff
-   const appComp = React.createElement('div', {style: {color: 'blue'}}, 'hello, world');
+   const appComp = (
+       <div style={{color: 'blue'}}>
+           hello, world
+       </div>
+   );
	  ReactDOM.createRoot(document.querySelector('#app')).render(appComp)
```

毫无疑问会报错：

![030-react-cdn-jsx-without-babel](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/030-react-cdn-jsx-without-babel.png)

严格意义上讲，浏览器没法解析JSX代码（前面已经提到了），但是我们可以通过Babel提供的`standalone`模块库（[@babel/standalone · Babel (babeljs.io)](https://babeljs.io/docs/babel-standalone.html)）来完成这一任务。该库不仅仅支持JSX，同时还支持ES6语法直接在浏览器上运行，而无需对代码进行预编译，其初衷是支持一些浏览器（说的就是你IE）能够编写ES6的代码。

关于@babel/standalone的具体使用方式为：

1. 引入`@babel/standalone`的CDN；
2. 将原有的含有JSX脚本的`<script>`标签添加属性`type="text/babel"`：`<script type="text/babel">`。

```diff
  <script src="https://unpkg.com/react@18.2.0/umd/react.production.min.js"></script>
  <script src="https://unpkg.com/react-dom@18.2.0/umd/react-dom.production.min.js"></script>
+ <script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>

- <script>
+ <script type="text/babel">
    // const appComp = React.createElement('div', {style: {color: 'blue'}}, 'hello, world');
    const appComp = (
        <div style={{color: 'blue'}}>
            hello, world
        </div>
    );
    ReactDOM.createRoot(document.querySelector('#app')).render(appComp)
</script>
```

完成上述的配置以后，我们就能在浏览器中看到源自JSX渲染而来的React组件了：

![040-react-cdn-jsx-with-babelstandalone](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/040-react-cdn-jsx-with-babelstandalone.png)

这个过程主要为`@babel/standalone`的js在加载的过程中，会读取HTML上的`type="text/babel"`的节点，然后对其内容进行编译转换。不难想到， 这个过程会十分消耗性能。所以Babel官方也强调了，[只能在某些场景下使用](https://babeljs.io/docs/babel-standalone#when-not-to-use-babelstandalone)。

# SolidJS中的JSX

SolidJS是新发展起来的又一响应式框架，同样的，SolidJS也使用JSX来完成视图层的编写。

> 不同于React的是，Solid 模型更简单，没有 Hook 规则。每个组件执行一次，随着依赖项的更新，钩子和绑定会多次执行。Solid 遵循与 React 相同的理念，具有单向数据流、读/写隔离和不可变接口。**但是放弃了使用虚拟 DOM**，使用了完全不同的实现。

## 工程于编译JSX

同样的，基于浏览器无法直接解析JSX事实，所以我们会比较好奇SolidJS编译出的内容，是什么样的。在SolidJS提供的Playground中（[Solid Playground (solidjs.com)](https://playground.solidjs.com/)），我们可以更加直观的看到SolidJS将JSX编译为了什么结果：

```jsx
import { createSignal } from "solid-js";

function Counter() {
  const [count, setCount] = createSignal(1);
  return (
    <button 
      type="button" 
      onClick={() => setCount(count() + 1)}
      >
        {count()}
    </button>
  );
}
```

![050-solidjs-jsx-compile-result](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/050-solidjs-jsx-compile-result.png)

在本文中，我们主要分析JSX的处理过程，暂不涉及响应式的实现方式。

首先可以看到我们编写的JSX，被**解析**为了一段非常纯粹的HTML代码字符串片段：

```html
`<button type="button"></button>`
```

然后，该字符串交给了来自`"solid-js/web"`中的`template`这个方法进行解析处理。

那么这个`template`方法是什么呢？通过查找类型定义，可以找到其来源于`solid-js/web`包中，`client.ts`导出的`template`的定义：

![060-where-is-template-func](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/060-where-is-template-func.png)

通过查看`client.ts`的源码，会发现`solid-js/web`关于`client.ts`的整个部分都来自`dom-expression/src/client`导出的内容：

[solid/client.ts at main · solidjs/solid (github.com)](https://github.com/solidjs/solid/blob/main/packages/solid/web/src/client.ts)

```tsx
// "solid-js/web"的client部分
export * from "dom-expressions/src/client";
```

所以，进一步翻阅dom-expression这个库，会找到client.js代码的实现，并且能够定位到template这个方法的实现（[dom-expressions/client.js at main · ryansolid/dom-expressions (github.com)](https://github.com/ryansolid/dom-expressions/blob/main/packages/dom-expressions/src/client.js)）：

```js
export function template(html, check, isSVG) {
  const t = document.createElement("template");
  t.innerHTML = html;
  if ("_DX_DEV_" && check && t.innerHTML.split("<").length - 1 !== check)
    throw `The browser resolved template HTML does not match JSX input:\n${t.innerHTML}\n\n${html}. Is your HTML properly formed?`;
  let node = t.content.firstChild;
  if (isSVG) node = node.firstChild;
  return node;
}
```

实际上，在不同的版本下，这些工具方法的实现有所不同，但是核心不变：

1. 创建template元素
2. 将html字符串插入到该元素
3. 进行一定的处理
4. 返回html对应的元素

比如我们编写一个demo：

![070-simple-demo-solidjs](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/070-simple-demo-solidjs.png)

经过编译后，查看编译代码，能够看到相关的实现：

![080-simple-demo-dist](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-04-05/080-simple-demo-dist.png)

与React一样，SolidJS同样用到了Babel对SolidJS的代码进行编译。核心的则是`babel-preset-solid`，与之前一些标准的preset（比如`@babel/preset-typescript`或是`@babel/preset-react`）命名不同，因为SolidJS还没有成为Babel官方预置集（还是比较小众的）。

关于SolidJS的代码处理过程，在Babel中，先经过`babel-preset-solid`进行编译，将JSX编译为模板字符串以及处理各种调用；然后，如果是TypeScript代码，则需要`@babel/preset-typescript`来进行TS代码处理。

## 浏览器使用JSX

遗憾的是，目前SolidJS还没有提供关于如何以UMD CDN方式直接在HTML中使用，就更不用说在浏览器中使用JSX进行代码编写了。不过，SolidJS还有一个名为`solid-element`的库，该库底层基于WebComponents，可以让我们预定义HTML元素，然后直接在HTML中使用这些元素。

