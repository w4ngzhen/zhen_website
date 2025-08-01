---
title: 从零搭建基于react与ts的组件库（一）项目搭建与封装antd组件
date: 2022-05-27
tags:
 - react
 - ts
categories:
  - 技术
---

为什么会有这样一篇文章？因为网上的教程/示例只说了怎么做，没有系统详细的介绍引入这些依赖、为什么要这样配置，甚至有些文章还是错的！迫于技术洁癖，我希望更多的开发小伙伴能够真正的理解一个项目搭建各个方面的细节，做到面对对于工程出现的错误能够做到有把握。

<!-- more -->

最近使用阿里低开引擎的时候，想要封装一套组件库作为物料给低开引擎引入。根据低开引擎的物料封层模式，我的诉求是做一套组件库，并且将该组件库以umd方式生成。当然，从零开始开发组件库也是一个比较耗时耗力的事情，所以我想到将antd组件封装，于是催生出了本篇文章。

**在封装组件并生成umd代码过程中，踩了很多的坑，也更加系统的了解了babel。**

## 整体需求

1. **react**组件库，取名r-ui，能够导出**r-ui.umd.js**和**r-ui.umd.css**。
2. 代码使用**typescript**进行开发。
3. 样式使用**less**进行开发。
4. 引入antd组件库作为底层原子组件库，并且r-ui.umd.js和r-ui.umd.css包含antd组件代码和样式代码。
5. 依赖的react、react-dom模块以**外部引用方式**。

## 开发与打包工具选型

### 使用webpack作为打包工具

老牌而又经典的打包工具，广泛的使用，丰富的插件生态以及各种易得的样例。

### 使用babel来处理typescript代码

> 由于 TypeScript 和 Babel 团队官方合作了一年的项目：[TypeScript plugin for Babel](https://link.zhihu.com/?target=https%3A//babeljs.io/docs/en/babel-preset-typescript.html)（`@babel/preset-typescript`），[TypeScript](https://link.zhihu.com/?target=https%3A//www.typescriptlang.org/) 的使用变得比以往任何时候都容易。 —— 摘自《[TypeScript With Babel: A Beautiful Marriage （TypeScript 和 Babel：美丽的结合）](https://iamturns.com/typescript-babel/)》

建议各位读者可以先阅读一下上面的文章（有中文翻译文章）。

### 使用less-loader、css-loader等处理样式代码

### 使用MiniCssExtractPlugin分离CSS

## 项目搭建思路

### 整体结构

```
- r-ui
  |- src
     |- components
        |- button
           |- index.tsx
  |- index.tsx
```

### 方案思路

编写webpack.config.js配置文件，添加核心loader：

1. babel-loader。接收ts文件，交给babel-core以及相关babel插件进行处理，得到js代码。
2. less-loader。接收less样式文件，处理得到css样式代码。
3. css-loader+MiniCssExtractPlugin.loader。接收css样式代码进行处理，并分离导出组件库样式文件。

## 项目搭建实施

### 初始化

**初始化r-ui项目**

```shell
mkdir r-ui && cd r-ui && npm init
# 配置项目基本信息（name、version......）
```

**初始化git仓库，添加gitignore文件（后续所有命令非特殊情况，均相对于项目根目录）**

```shell
git init
# .gitignore文件内容请直接查看项目内文件
# 完成后，初始提交：
# git add . && git commit -m "init"
```

**安装webpack（包管理器使用yarn）**

```shell
yarn add -D webpack webpack-cli webpack-dev-server
# 安装webpack-dev-server是为后续构建样例页面做准备，前期可以不安装。
```

```diff
diff --git a/package.json b/package.json
index e01c1b1..53dd9a3 100644
--- a/package.json
+++ b/package.json
@@ -8,5 +8,9 @@
   },
   "author": "",
   "license": "MIT",
-  "devDependencies": {}
+  "devDependencies": {
+    "webpack": "^5.72.1",
+    "webpack-cli": "^4.9.2",
+    "webpack-dev-server": "^4.9.0"
+  }
 }
```

**项目根目录添加webpack.config.js并进行初始配置**

```js
// webpack.config.js
const {resolve} = require("path");
module.exports = {
  // 组件库的起点入口
  entry: './src/index.tsx',
  output: {
    filename: "r-ui.umd.js", // 打包后的文件名
    path: resolve(__dirname, 'dist'), // 打包后的文件目录：根目录/dist/
    library: 'rui', // 导出的UMD js会在window挂rui，即可以访问window.rui
    libraryTarget: 'umd' // 导出库为UMD形式
  },
  resolve: {
    // webpack 默认只处理js、jsx等js代码
    extensions: ['.js', '.jsx', '.ts', '.tsx']
  },
  externals: {},
  // 模块
  module: {
    // 规则
    rules: []
  }
};
```

### Babel引入

#### 引入babel-loader以及相关babel库

```shell
yarn add -D babel-loader @babel/core @babel/preset-env @babel/preset-typescript @babel/preset-react @babel/plugin-proposal-class-properties @babel/plugin-proposal-object-rest-spread
```

```diff
diff --git a/package.json b/package.json
index 53dd9a3..33c32b6 100644
--- a/package.json
+++ b/package.json
@@ -9,6 +9,13 @@
   "author": "",
   "license": "MIT",
   "devDependencies": {
+    "@babel/core": "^7.18.2",
+    "@babel/plugin-proposal-class-properties": "^7.17.12",
+    "@babel/plugin-proposal-object-rest-spread": "^7.18.0",
+    "@babel/preset-env": "^7.18.2",
+    "@babel/preset-react": "^7.17.12",
+    "@babel/preset-typescript": "^7.17.12",
+    "babel-loader": "^8.2.5",
     "webpack": "^5.72.1",
     "webpack-cli": "^4.9.2",
     "webpack-dev-server": "^4.9.0"
(END)
```

#### 了解Babel

如果对于babel不太熟悉，可能对这一堆的依赖感到恐惧，这里如果读者有时间，我推荐这篇深入了解babel的文章：[一口（很长的）气了解 babel - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/43249121)。当然，如果这口气憋不住（哈哈），我做一个简单摘抄：

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

`@babel/core`毋庸置疑，babel的核心模块，实现了上述的流程运转以及代码语法、语义分析的功能。

以plugin开头的就是插件，这里我们引入了两个：`@babel/plugin-proposal-class-properties`（[允许类具有属性](https://babel.docschina.org/docs/en/babel-plugin-proposal-class-properties/)）和`@babel/plugin-proposal-object-rest-spread`（[对象展开](https://babel.docschina.org/docs/en/7.0.0/babel-plugin-proposal-object-rest-spread/)）。

以preset开头的就是预置组件包合集，其中`@babel/preset-env`表示使用了可以根据实际的浏览器运行环境，会选择相关的转义插件包：

>env 的核心目的是通过配置得知目标环境的特点，然后只做必要的转换。
>
>如果不写任何配置项，env 等价于 latest，也等价于 es2015 + es2016 + es2017 三个相加(不包含 stage-x 中的插件)。
>
>```
>{
>  "presets": [
>    ["env", {
>      "targets": {
>        "browsers": ["last 2 versions", "safari >= 7"]
>      }
>    }]
>  ]
>}
>```
>
>如上配置将考虑所有浏览器的最新2个版本(safari大于等于7.0的版本)的特性，将必要的代码进行转换。而这些版本已有的功能就不进行转化了。
>
>—— 摘自《[一口（很长的）气了解 babel - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/43249121)》

`@babel/preset-typescript`会处理所有ts的代码的语法和语义规则，并转换为js代码；`@babel/preset-react`

故名思义，**可以帮助处理使用React相关特性**，例如JSX标签语法等。

#### webpack的基于babel-loader的处理流程

讲了这么多，我们的打包工具webpack如何使用babel相关组件处理代码的呢？还记得我们安装过**babel-loader**吗？

实际上，我们通过配置webpack.config.js，使用**babel-loader**建立起webpack处理代码与babel处理代码的连接：

```diff
diff --git a/webpack.config.js b/webpack.config.js
index 8bfbb63..6767fd8 100644
--- a/webpack.config.js
+++ b/webpack.config.js
@@ -17,6 +17,11 @@ module.exports = {
   // 模块
   module: {
     // 规则
-    rules: []
+    rules: [
+      {
+        test: /\.tsx?$/,
+        use: 'babel-loader',
+        exclude: /node_modules/
+      }
+    ]
   }
 };
(END)
```

这一步的配置，就是让webpack遇到ts或tsx的时候，将这些代码交给babel-loader，babel-loader作为桥接把代码交给内部引用的@babel/core相关API进行处理，当然为了防止babel-loader去解析依赖库node_modules的内容，需要配置exclude。

那么，@babel/core如何知道要使用我们安装的各种plugin插件和preset预置插件包的呢？通过`.babelrc文件`（注：实际上还有其他配置方式，但个人倾向于.babelrc）。这里，我们在项目根目录创建.babelrc文件，并添加一下内容：

```json
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-typescript",
    "@babel/preset-react"
  ],
  "plugins": [
    "@babel/plugin-proposal-class-properties",
    "@babel/plugin-proposal-object-rest-spread"
  ]
}
```

这里的配置不难理解，plugins字段存放要使用的插件，presets字段存放预置插件包名称，具体的配置可以查阅官方文档。

**总结一下，配置babel可以按照如下思路进行：**

1. xxx.ts(x)代码交给webpack打包；
2. webpack遇到ts(x)结尾的代码文件，根据webpack.config.js配置，交给babel-loader；
3. babel-loader交给@babel/core；
4. @babel/core根据.babelrc配置交给相关的插件处理代码，转为js代码；
5. webpack进行后续的打包操作。

### 引入React相关库（externals方式）

还记得我们的需求吗？

>依赖的react、react-dom模块以**外部引用方式**。

什么是外部引用方式？简单来讲，我希望react、react-dom等组件库的包，不会被打入到组件库中，而是在html中引入（[Add React to a Website – React (reactjs.org)](https://17.reactjs.org/docs/add-react-to-a-website.html#step-2-add-the-script-tags)）：

```html
  <!-- ... other HTML ... -->
  <!-- Load React. -->
  <!-- Note: when deploying, replace "development.js" with "production.min.js". -->
  <script src="https://unpkg.com/react@17/umd/react.development.js" crossorigin></script>
  <script src="https://unpkg.com/react-dom@17/umd/react-dom.development.js" crossorigin></script>
  <!-- 组件库JS -->
  <script src="r-ui.js"></script>
</body>
```

要实现这样的效果，第一步是配置webapck.config.js：

```diff
diff --git a/webpack.config.js b/webpack.config.js
index 6767fd8..54fc0e5 100644
--- a/webpack.config.js
+++ b/webpack.config.js
@@ -13,7 +13,13 @@ module.exports = {
     // webpack 默认只处理js、jsx等js代码
     extensions: ['.js', '.jsx', '.ts', '.tsx']
   },
-  externals: {},
+  externals: {
+    // 打包过程遇到以下依赖导入，不会打包对应库代码，而是调用window上的React和ReactDOM
+    // import React from 'react'
+    // import ReactDOM from 'react-dom'
+    'react': 'React',
+    'react-dom': 'ReactDOM'
+  },
   // 模块
   module: {
     // 规则
(END)
```

第二部，在引入react相关库的时候，可以不用引入到dependencies运行时依赖，而只需要引入对应的类型定义到devDependencies开发依赖中：

```shell
yarn add -D @types/react@17.0.39 @types/react-dom@17.0.17
```

```diff
diff --git a/package.json b/package.json
index 33c32b6..bd17763 100644
--- a/package.json
+++ b/package.json
@@ -15,6 +15,8 @@
     "@babel/preset-env": "^7.18.2",
     "@babel/preset-react": "^7.17.12",
     "@babel/preset-typescript": "^7.17.12",
+    "@types/react": "17.0.39",
+    "@types/react-dom": "17.0.17",
     "babel-loader": "^8.2.5",
     "webpack": "^5.72.1",
     "webpack-cli": "^4.9.2",
```

至此，我们已经完成了处理**基于TypeScript**的**React项目**的webpack配置，此时我们的项目结构如下：

```
- r-ui
  |- .babelrc
  |- package.json
  |- webpack.config.js
```

### 阶段演示1：基于TypeScript的React组件项目的webpack配置可行性

#### 编写组件代码

新增src目录，在src目录下添加index.tsx（用于将所有的组件导出）

src目录下添加components/button目录，并创建index.tsx文件。具体结构与目录如下：

```
- r-ui
  |- src/components/button/index.tsx
  |- src/index.tsx
  |- ... ...
```

**src/components/button/index.tsx**

```tsx
import * as React from 'react';

interface ButtonProps {
}

const Button: React.FC<ButtonProps> = (props) => {
    const {children, ...rest} = props;
    return <button {...rest} >{children}</button>
}

export default Button;
```

**src/index.tsx**

```tsx
export {default as Button} from './components/button';
```

#### 修改package.json

添加webpack处理脚本

```diff
diff --git a/package.json b/package.json
index bd17763..01565ad 100644
--- a/package.json
+++ b/package.json
@@ -4,6 +4,7 @@
   "description": "",
   "main": "index.js",
   "scripts": {
+    "build": "webpack --config webpack.config.js",
     "test": "echo \"Error: no test specified\" && exit 1"
   },
   "author": "",
(END)
```

#### 编译打包组件库

```shell
yarn run build
```

打包完成后，在**项目根目录/dist目录**下，会生成一个r-ui.umd.js文件。

#### 效果演示

想要查看效果，可以在dist目录下添加如下的html：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>r-ui example</title>
    <script src="https://unpkg.com/react@17/umd/react.development.js"></script>
    <script src="https://unpkg.com/react-dom@17/umd/react-dom.development.js"></script>
  	<!-- 注意r-ui.umd.js的路径 -->
    <script src="r-ui.umd.js"></script>
</head>
<body>
<div id="example"></div>
<script>
  const onClick = () => {
    alert('hello');
  };
  // window上存在rui，是因为我们将组件包导出为了umd包，取名为rui
  // 使用React原生方法创建Button的react组件实例
  // 等价于：
  // <Button onClick={onClick}>hello, world</Button>
  const button = React.createElement(rui.Button, {onClick}, 'hello, world');
  // 调用ReactDOM方法，将button组件实例挂载到example DOM节点上
  ReactDOM.render(button, document.getElementById('example'));
</script>
</body>
</html>
```

```
- r-ui
  |- dist
     |- index.html
     |- r-ui.umd.js
  |- ... ...
```

此时，可以直接使用浏览器打开index.html查看效果：

![010](https://static-res.zhen.wang/images/post/2022-05-27-webpack-ts-react/010.gif)

### 处理样式（less编译与css导出）

#### 依赖引入

根据上述内容，我们已经搭建了基础的项目结构，但是目前来说我们还需要处理我们的less样式，并且能够支持导出r-ui.umd.css样式文件。基于此考虑，我们需要引入：

1. less-loader。处理less样式代码，转为css；
2. less。由于less-loader内部是调用了less模块进行less代码编译，故还需要引入less（模式和babel-loader内部使用@babel/core一样）；
3. css-loader。处理css样式代码，进行适当加工；
4. mini-css-extract-plugin。MiniCssExtractPlugin的loader用于进一步处理css，并且该插件用于导出独立样式文件。

```shell
yarn add -D less-loader less css-loader mini-css-extract-plugin
```

```diff
diff --git a/package.json b/package.json
index 01565ad..3070d07 100644
--- a/package.json
+++ b/package.json
@@ -19,6 +19,10 @@
     "@types/react": "17.0.39",
     "@types/react-dom": "17.0.17",
     "babel-loader": "^8.2.5",
+    "css-loader": "^6.7.1",
+    "less": "^4.1.2",
+    "less-loader": "^11.0.0",
+    "mini-css-extract-plugin": "^2.6.0",
     "webpack": "^5.72.1",
     "webpack-cli": "^4.9.2",
     "webpack-dev-server": "^4.9.0"
```

#### 配置webpack

根据上述依赖，我们可以知道需要less-loader、css-loader以及MiniCssExtractPlugin的内置loader来处理我们的样式代码。但是配置到webpack需要注意： webpack中的顺序是**【从后向前】**链式调用的，所以注意下面配置的代码中use数组的顺序：

```diff
diff --git a/webpack.config.js b/webpack.config.js
index 54fc0e5..9db43b8 100644
--- a/webpack.config.js
+++ b/webpack.config.js
@@ -1,5 +1,6 @@
 // webpack.config.js
 const {resolve} = require("path");
+const MiniCssExtractPlugin = require('mini-css-extract-plugin');
 module.exports = {
   // 组件库的起点入口
   entry: './src/index.tsx',
@@ -27,7 +28,28 @@ module.exports = {
       {
         test: /\.tsx?$/,
         use: 'babel-loader',
         exclude: /node_modules/
+      },
+      {
+        test: /\.less$/,
+        use: [
+          // webpack中的顺序是【从后向前】链式调用的
+          // 所以对于less先交给less-loader处理，转为css
+          // 再交给css-loader
+          // 最后导出css（MiniCssExtractPlugin.loader）
+          // 所以注意loader的配置顺序
+          {
+            loader: MiniCssExtractPlugin.loader,
+          },
+          'css-loader',
+          'less-loader'
+        ]
       }
     ]
-  }
+  },
+  plugins: [
+    // 插件用于最终的导出独立的css的工作
+    new MiniCssExtractPlugin({
+      filename: 'r-ui.umd.css'
+    }),
+  ]
 };
```

### 阶段演示2：less样式处理配置可行性

#### 编写样式代码

新增**src/components/button/index.less**

```less
@color: #006fde;

.my-button {
  color: @color;
}
```

修改**src/components/button/index.tsx**

```diff
 import * as React from 'react';
+// 引入less样式
+import './index.less';
 
 interface ButtonProps {
 }
 
 const Button: React.FC<ButtonProps> = (props) => {
     const {children, ...rest} = props;
-    return <button {...rest} >{children}</button>
+    // 使用my-button样式
+    return <button {...rest} className='my-button'>{children}</button>
 }
 
 export default Button;
```

#### 编译组件库

再次打包组件库以后，dist目录下会额外生成文件：r-ui.umd.css。所以，我们需要在index.html中添加样式文件的引入：

```diff
 <head>
     <meta charset="UTF-8">
     <title>r-ui example</title>
     <script src="https://unpkg.com/react@17/umd/react.development.js"></script>
     <script src="https://unpkg.com/react-dom@17/umd/react-dom.development.js"></script>
     <script src="r-ui.umd.js"></script>
+    <link href="r-ui.umd.css" rel="stylesheet"/>
 </head>
```

#### 效果演示

刷新页面后，可以看到按钮的文字颜色已经生效

![020](https://static-res.zhen.wang/images/post/2022-05-27-webpack-ts-react/020.jpg)

### 引入AntDesign

根据我们的需求，我们希望将antd组件代码引用到我们组件内部进行封装，所以需要以dependencies方式引入：

```shell
yarn add antd
```

```diff
diff --git a/package.json b/package.json
index 3070d07..09ca792 100644
--- a/package.json
+++ b/package.json
@@ -26,5 +26,8 @@
     "webpack": "^5.72.1",
     "webpack-cli": "^4.9.2",
     "webpack-dev-server": "^4.9.0"
+  },
+  "dependencies": {
+    "antd": "^4.20.6"
   }
 }
```

#### 引用antd的button样式

**src/components/button/index.less**

```diff
-@color: #006fde;
-
-.my-button {
-  color: @color;
-}
+@import "~antd/lib/button/style/index.css";
```

#### 引用antd的button组件

```diff
 import * as React from 'react';
+// 使用antd的Button和ButtonProps
+// 为了不和我们的Button冲突，需要改导出名
+import {Button as AntdButton, ButtonProps as AntdButtonProps} from 'antd';
 // 引入less样式
 import './index.less';
 
-interface ButtonProps {
+interface ButtonProps extends AntdButtonProps {
 }
 
 const Button: React.FC<ButtonProps> = (props) => {
     const {children, ...rest} = props;
-    // 使用my-button样式
-    return <button {...rest} className='my-button'>{children}</button>
+    // 使用AntdButton
+    return <AntdButton {...rest}>{children}</AntdButton>
 }
 
 export default Button;
```

### 阶段演示3：antd组件引入可行性

通过上述的代码修改以后，我们直接进行编译，然后检查效果即可：

![030](https://static-res.zhen.wang/images/post/2022-05-27-webpack-ts-react/030.gif)

## 写在最后

实际上，代码开发过程中，还有很多可以辅助开发的模块、流程，例如eslint检查，热更新等。但是那些内容不在本文的讨论范围。后续会出相关的文章再进一步进行介绍。

本文所搭建的整个项目，我都按照文章一步一步进行了git提交，开发小伙伴可以边阅读文章边对照git提交一步一步来看。

github地址：[w4ngzhen/r-ui (github.com)](https://github.com/w4ngzhen/r-ui)

