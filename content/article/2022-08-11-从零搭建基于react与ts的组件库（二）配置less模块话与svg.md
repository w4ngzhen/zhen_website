---
title: 从零搭建react与ts组件库（二）less模块化与svg引入配置
date: 2022-08-11
tags:
 - react
 - ui
categories:
  - 技术
---

在上一篇《从零搭建react+ts组件库（一）项目搭建与封装antd组件》介绍了使用webpack来搭建一个基于antd的组件库的基本框架，但是作为一个组件库，实际上还有很多的都还未引入，本篇将会补充less模块化以及svg引入的基本方式。

<!-- more -->

本文所有修改的代码分支为chapter02位于[w4ngzhen/r-ui (github.com)](https://github.com/w4ngzhen/r-ui)仓库的`chapter02_less_and_svg`分支，该分支基于上一篇文章的`chapter01_init`分支而来（main分支总是显示最新的内容）。

为了讲解如何进行less模块化配置以及如何引入svg作为组件库的一部分，我们设想这样一个需求：一个搜索输入框，左侧是一个svg的icon搜索图标，右侧是输入框。

# 组件规划

首先考虑组件具备的属性，作为一个简单的搜索框，我们至少有3个属性：

1. 输入框初始默认值（defaultValue）
2. 占位提示信息（placeholder）
3. 输入改变事件（onChange）

对于UI结构来说，我们可以使用一个div作为整体包裹，然后左侧是图标的区域（使用一个div），右侧是输入框（input）。

综上，我们的初始代码如下：

```react
import * as React from "react";

interface SearchInputProps {
    defaultValue?: string;
    placeholder?: string;
    onChange?: (value: string, e: React.ChangeEvent<HTMLInputElement>) => void;
}

const SearchInput: React.FC<SearchInputProps> = (props) => {
    const {
        defaultValue,
        placeholder,
        onChange,
    } = props;

    const inputOnChange: React.ChangeEventHandler<HTMLInputElement>
        = (e: React.ChangeEvent<HTMLInputElement>) => {
        if (onChange) {
            onChange(e.target.value, e);
        }
    }

    return (
        // 包裹
        <div>
            {/*存放icon*/}
            <div></div>
            {/* 输入框*/}
            <input defaultValue={defaultValue}
                   placeholder={placeholder}
                   onChange={inputOnChange}></input>
        </div>
    );
}
```

# less样式模块化配置

首先我们编写less样式文件，当然，对于该文件我们不赘述实现。

```less
@input-size: 32px;

.centerAll() {
  display: flex;
  flex-direction: row;
  justify-content: center;
  align-items: center;
}

.searchInputWrap {

  width: 100%;
  height: @input-size;

  background-color: #f4f4f4;
  border-radius: 5px;

  .centerAll();

  .searchInputIconBox {
    width: @input-size;
    height: @input-size;

    background-color: transparent;
    .centerAll();
  }

  .searchInput {
    width: calc(100% - @input-size);
    height: 100%;
    border: none;
    outline: none;
    background-color: transparent;
  }

  .searchInput:focus {
    border: none;
    outline: none;
  }
}

```

修改组件代码，改动如下：

1. 以模块化的方式引入less文件

```diff
 import * as React from "react";
+import styles from './index.module.less';
```

2. 针对每个元素配置其less样式，采用`styles.xxx`方式使用

```diff
     return (
         // 包裹
-        <div>
+        <div className={styles.searchInputWrap}>
             {/*存放icon*/}
-            <div></div>
+            <div className={styles.searchInputIconBox}></div>
             {/* 输入框*/}
-            <input defaultValue={defaultValue}
+            <input className={styles.searchInput}
+                   defaultValue={defaultValue}
                    placeholder={placeholder}
                    onChange={inputOnChange}></input>
         </div>
     )
```

webpack对于css-loader需要进行简单的配置：

```diff
           {
             loader: MiniCssExtractPlugin.loader,
           },
-          'css-loader',
+          {
+            loader: "css-loader",
+            options: {
+              modules: true
+            }
+          },
           'less-loader'

```

对于该处的配置，详细可以查看关于css-loader的文档[webpack-contrib/css-loader: CSS Loader (github.com)](https://github.com/webpack-contrib/css-loader)

此时，如果有同学在使用IDEA会发现有编译报错。有同学会发现，我们的项目里面没有直接安装typescript，那么为什么IDEA能够检测到我们代码呢？实际上这是IDEA自带的ts在进行类型检测，**仅仅是类型检查**，实际上编译过程我们是调用的babel-loader+preset/typescript这条链路来完成的，所以并不影响编译后的内容。当然，为了能够进行正确的类型检查，我们在项目根目录下添加tsconfig.json：

```json
{
  "compilerOptions": {
    "noEmit": true,
    "esModuleInterop": true,
    "jsx": "react"
  },
  "include": [
    "src",
    "./src/external.d.ts"
  ]
}
```

其中，`"noEmit": true`表明由ts进行类型检查，但是不编译文件。include中的`./src/external.d.ts`中的内容如下：

```ts
// less模块声明
declare module '*.module.less' {
    const content: { [className: string]: string };
    export = content;
}
```

也就是说，希望IDEA的内置ts读取tsconfig.json，并添加关于import`*.module.less`时候得到的模块的类型定义。这样，IDEA的ts类型检查在这句话的时候就不会报错了：

```ts
import styles from './index.module.less';
```

总结一下，想要在ts+babel-loader项目中使用样式模块化。

在类型检查阶段，需要：

1. 单独配置tsconfig.json
2. 编写d.ts，并被tsconfig.json配置包含在类型定义查找的范围（inlcude）

在编译阶段，需要只需要配置css-loader的module为true即可。

这一块我会再写一篇文章来单独讲解webpack+ts+babel的方案。

## 效果演示

编写样例html：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>r-ui example</title>
    <script src="https://unpkg.com/react@17/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@17/umd/react-dom.development.js"></script>
    <script src="r-ui.umd.js"></script>
    <link href="r-ui.umd.css" rel="stylesheet"/>
</head>
<body>
<div id="example"></div>
<script>
  // window上存在rui，是因为我们将组件包导出为了umd包，取名为rui
  const button = React.createElement(rui.SearchInput, {
    placeholder: '占位符',
    defaultValue: 'hello, world',
    onChange: (value, e) => console.log(value, e)
  });
  ReactDOM.render(button, document.getElementById('example'));
</script>
</body>
</html>
```

编译r-ui后打开样例界面，可以看到如下效果：

![010-less-module](https://res.zhen.blog/images/post/2022-08-11/010-less-module.gif)

# svg引入配置

实际上，react中想要使用svg有这很多种方式，像是直接编写react组件，并返回svg代码：

```react
import * as React from "react";

const IconSearch = () => {
    return (
        <svg className="icon"
             viewBox="0 0 1024 1024" version="1.1" xmlns="http://www.w3.org/2000/svg"
             p-id="4136" width="25" height="25">
            <path
                d="M909.6 854.5L649.9 594.8C690.2 542.7 712 479 712 412c0-80.2-31.3-155.4-87.9-212.1-56.6-56.7-132-87.9-212.1-87.9s-155.5 31.3-212.1 87.9C143.2 256.5 112 331.8 112 412c0 80.1 31.3 155.5 87.9 212.1C256.5 680.8 331.8 712 412 712c67 0 130.6-21.8 182.7-62l259.7 259.6c3.2 3.2 8.4 3.2 11.6 0l43.6-43.5c3.2-3.2 3.2-8.4 0-11.6zM570.4 570.4C528 612.7 471.8 636 412 636s-116-23.3-158.4-65.6C211.3 528 188 471.8 188 412s23.3-116.1 65.6-158.4C296 211.3 352.2 188 412 188s116.1 23.2 158.4 65.6S636 352.2 636 412s-23.3 116.1-65.6 158.4z"
                p-id="4137"></path>
        </svg>
    );
}
```

这种方式弊端在于，每次svg有修改的时候，都需要重新复制svg代码对代码文件进行修改，且很多svg的数据较为复杂，容易出错。

## 将svg作为react组件来使用

我们知道，对于webpack来说，可以将一切的东西都是为模块，对于任何import进来的，webpack都可以通过匹配的规则（rules）调用对应的loader来进行处理。

那么，是否存在这样一种方式：

```react
import IconSearch from 'path/to/search.svg'
// IconSearch是一个React组件，可以在其他组件中使用
```

个人最常使用的方案是`svgr/webpack`（[Webpack - SVGR (react-svgr.com)](https://react-svgr.com/docs/webpack/)）

只需要三个步骤的配置：

1. 引入@svgr/webpack

```
yarn add -D @svgr/webpack
```

2. 配置webpack，处理svg

```js
      {
        test: /\.svg$/,
        use: ['@svgr/webpack'],
      }
```

3. external.d.ts配置（配置理由和上述less配置一样，为了达到类型检查）

```ts
// svg类型
declare module '*.svg' {
    const content: React.FunctionComponent<React.SVGAttributes<React.ReactSVGElement>>
    export default content
}
```

完成配置以后，我们就可以通过import XXX from 'path/to/xxx.svg'，来使用SVG组件了：

```diff
 import * as React from "react";
 import styles from './index.module.less';
+import IconSearch from "../../assets/svg/search.svg";
 
 interface SearchInputProps {
     defaultValue?: string;
     // ... ...
         // 包裹
         <div className={styles.searchInputWrap}>
             {/*存放icon*/}
-            <div className={styles.searchInputIconBox}></div>
+            <div className={styles.searchInputIconBox}>
+                <IconSearch width={18} height={18}/>
+            </div>
             {/* 输入框*/}

```

## 效果演示

![020-use-svg](https://res.zhen.blog/images/post/2022-08-11/020-use-svg.png)