---
title: 基于webpack与TypeScript的SolidJS项目搭建
date: 2023-03-22
tags:
 - solidjs
 - javascript
categories:
  - 技术
---

本文将讲述如何基于webpack与TypeScript搭建一个基础的支持less模块的solidjs项目。方便后续涉及到solidjs相关分析与讨论都可以基于本文的成果之上进行。

<!-- more --> 

# 前置

1. nodejs v14+
2. 全局yarn（npm亦可）
3. 稳定的网络环境

创建根目录`solidjs-webpack-ts-example`：

```shell
mkdir solidjs-webpack-ts-example
cd solidjs-webpack-ts-example
```

# yarn初始化

```shell
➜  solidjs-webpack-ts-example git:(main) yarn init                                                          
yarn init v1.22.19
question name (solidjs-webpack-ts-example): 
question version (1.0.0): 
question description: 
question entry point (index.js): 
question repository url: 
question author (w4ngzhen <w4ngzhen@hotmail.com>): 
question license (MIT): 
question private: 
success Saved package.json
✨  Done in 3.68s.
```

# git初始化

```shell
# 前置：添加.gitignore文件
git init
git add .
git commit -m "init"
```

# 库安装

## webpack 4件套

```shell
yarn add -D webpack webpack-cli webpack-dev-server html-webpack-plugin
```

## 样式处理 4件套

```
yarn add -D less less-loader css-loader mini-css-extract-plugin
```

1. less：less核心编译解析库；

2. less-loader：webpack与less的桥梁。当webpack处理less时，通过配置指定交给less-loader，less-loader调用安装的less，将less代码编译为css代码；
3. css-loader：wepback处理css样式代码的loader。处理css，或处理来自less编译成的css；
4. mini-css-extract-plugin：css样式处理最后一个环节，交给该插件的提供的loader、plugin完成独立样式文件打包生成。

## babel 7件套

```
yarn add -D @babel/core
yarn add -D @babel/preset-env @babel/preset-typescript babel-preset-solid
yarn add -D @babel/plugin-proposal-class-properties @babel/plugin-proposal-object-rest-spread
yarn add -D babel-loader
```

1. @babel/core：babel核心库；
2. @babel/preset-env、@babel/preset-typescript、babel-preset-solid（这个preset名字目前没有符合babel规范）：babel扩展preset，整合当前主流浏览器支持语法、typescript语法支持以及solidjs相关语法支持；
3. @babel/plugin-proposal-class-properties、@babel/plugin-proposal-object-rest-spread：babel扩展插件，支持类定义属性以及`...`剩余属性解构语法；
4. babel-loader：webpack与babel的桥梁。当webpack处理相关代码的时候，通过配置指定交给babel-loader，babel-loader内部调用上述第1个babel核心库，并结合相关的preset、plugin完成代码编译。

## TypeScript 1件套

实际山，主流IDE（WebStorm、VSCode）等都内置了TypeScript库，可以不用安装TS，只需要配置`tsconfig.json`就可以完成代码编写过程中的类型检查（babel编译的时候，不会进行类型检查）。关于这一块，推荐大家阅读另一篇文章：[【长文详解】TypeScript与Babel、webpack的关系以及IDE对TS的类型检查 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/561186916)。

为了方便的进行类型检查，本样会安装项目级：

```shell
yarn add -D typescript
```

至此，我们安装了目前基础项目所需要的开发依赖（devDependencies）。

```json
{
  "name": "solidjs-webpack-ts-example",
  "version": "1.0.0",
  "main": "index.js",
  "author": "w4ngzhen",
  "license": "MIT",
  "devDependencies": {
    "@babel/core": "^7.21.3",
    "@babel/plugin-proposal-class-properties": "^7.18.6",
    "@babel/plugin-proposal-object-rest-spread": "^7.20.7",
    "@babel/preset-env": "^7.20.2",
    "@babel/preset-typescript": "^7.21.0",
    "babel-loader": "^9.1.2",
    "babel-preset-solid": "^1.6.13",
    "css-loader": "^6.7.3",
    "html-webpack-plugin": "^5.5.0",
    "less": "^4.1.3",
    "less-loader": "^11.1.0",
    "mini-css-extract-plugin": "^2.7.5",
    "typescript": "^5.0.2",
    "webpack": "^5.76.2",
    "webpack-cli": "^5.0.1",
    "webpack-dev-server": "^4.13.1"
  }
}
```

## SolidJS 2件套

```
yarn add solid-js @solidjs/router
```

1. solid-js：SolidJs核心库；
2. @solidjs/router：solidjs官方SPA路由组件。

# 项目配置

## tsconfig.json

项目根目录

```json
{
  "compilerOptions": {
    "target": "es2016",
    "module": "commonjs",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "noEmit": true,
    "skipLibCheck": true,
    "jsx": "preserve",
    "jsxImportSource": "solid-js"
  },
  "include": [
    "src"
  ]
}
```

## webpack.config.js

项目根目录

```js
const {resolve} = require('path');
const HtmlWebpackPlugin = require("html-webpack-plugin");
const MiniCssExtractPlugin = require("mini-css-extract-plugin");
module.exports = {
    mode: "development",
    entry: {
        main: resolve(__dirname, 'src', 'index.tsx')
    },
    output: {
        filename: "app.js",
        path: resolve(__dirname, 'dist')
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
                test: /\.tsx?/,
                use: [
                    'babel-loader'
                ],
                exclude: /node_moudles/
            },
            {
                test: /\.css$/,
                use: [
                    MiniCssExtractPlugin.loader,
                    'css-loader'
                ]
            },
            {
                test: /\.less$/,
                use: [
                    MiniCssExtractPlugin.loader,
                    'css-loader',
                    'less-loader'
                ]
            }
        ]
    },
    plugins: [
        new HtmlWebpackPlugin({
            template: resolve(__dirname, 'public', 'index.html'),
            inject: 'body',
          	favicon: resolve(__dirname, 'public', 'favicon-32x32.png')
        }),
        new MiniCssExtractPlugin({
            filename: 'app.css'
        })
    ],
    devServer: {
        port: 8080
    }
}
```

## .babelrc

项目根目录

```json
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-typescript",
    "babel-preset-solid"
  ],
  "plugins": [
    "@babel/plugin-proposal-object-rest-spread",
    "@babel/plugin-proposal-class-properties"
  ]
}
```

## package.json

添加开发、构建以及类型检查脚本：

```diff
{
   ... ...
+  "scripts": {
+   "typecheck": "tsc",
+   "start": "webpack-dev-server --config webpack.config.js",
+   "build": "webpack --config webpack.config.js --mode=production"
+  },
   ... ...
}
```

# 初始文件创建

## app.d.ts

路径：项目根目录/src/app.d.ts。涉及到譬如less模块在ts中使用的类型定义。

```typescript
declare module '*.module.less' {
    const content: {
        [className: string]: any
    }
    export default content;
}
```

## index.html

路径：项目根目录/public/index.html（主要是与webpack中的HtmlWebpackPlugin.template选项对应）。

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <link rel="icon" type="image/png" sizes="32x32" href="/favicon-32x32.png">
    <title>SolidJS webpack ts example</title>
</head>
<body>
<div id="app"></div>
</body>
</html>
```

PS：public目录还创建了一个icon，就不详细说明了。

## index.tsx

路径：项目根目录/src/index.tsc

```tsx
import {render} from 'solid-js/web';
import styles from './index.module.less';

function HelloWorld() {
    return (
        <div class={styles.hello}>
            <p>Hello World!</p>
        </div>
    );
}

render(() => <HelloWorld/>, document.querySelector('#app')!)
```

## index.module.less

路径：项目根目录/src/index.module.less

```less
.hello {
  p {
    font-size: 16px;
    color: #2c4f7c;
  }
}
```

# 项目运行效果

![010-example-show](https://res.zhen.blog/images/post/2023-03-22/010-example-show.png)

# 附录

项目代码地址：

[w4ngzhen/solidjs-webpack-ts-example (github.com)](https://github.com/w4ngzhen/solidjs-webpack-ts-example)