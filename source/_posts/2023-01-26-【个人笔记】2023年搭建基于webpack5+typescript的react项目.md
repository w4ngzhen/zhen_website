---
title: 【个人笔记】2023年搭建基于webpack5+typescript的react项目
date: 2023-01-26
tags:
 - note
---

# 写在前面

由于我在另外的一些文章所讨论或分析的内容可能基于一个已经初始化好的项目，为了避免每一个文章都重复的描述如何搭建项目，我在本文会统一记录下来，今后相关的文章直接引用文本，方便读者阅读。此文主要为个人笔记，不会有太多的关于思路的描述；另外，本文仅仅描述如何搭建基础react项目，不涉及图片等资源的加载，关于图片等资源的处理，我会单独编写一期。

<!-- more -->

# 项目初始化

创建一个目录，例如：webpack5-react-demo，然后进入目录执行初始化指令

```shell
$ mkdir webpack5-react-demo
$ cd webpack5-react-demo 
$ yarn init
yarn init v1.22.19
question name (webpack5-react-demo): 
question version (1.0.0): 
question description: 
question entry point (index.js): 
question repository url: 
question author: 
question license (MIT): 
question private: 
success Saved package.json
✨  Done in 10.32s.
```

添加gitignore文件，路径为`项目根目录/.gitignore`：

```
.idea
.vscode
node_modules
dist
```

初始化git仓库：

```shell
$ git init
$ git add .
$ git commit -m 'init'
```

# （0）NPM依赖添加

```shell
echo '【1】基于webpack的项目核心相关内容'
echo '添加webpack基础四件套依赖'
yarn add -D webpack webpack-cli webpack-dev-server html-webpack-plugin

echo '【2】处理js(x)、ts(x)的相关模块'
echo '添加babel核心模块'
yarn add -D @babel/core
echo '添加babel相关preset欲集'
yarn add -D @babel/preset-env @babel/preset-react @babel/preset-typescript
echo '添加babel相关plugin插件'
yarn add -D @babel/plugin-proposal-class-properties @babel/plugin-proposal-object-rest-spread
echo '添加babel-loader'
yarn add -D babel-loader

echo '【3】处理style样式的相关模块'
echo '添加css-loader以及MiniCssExtractPlugin'
yarn add -D css-loader mini-css-extract-plugin
yarn add -D less less-loader

echo '【4】添加react/react-dom的类型定义以及运行依赖'
yarn add react react-dom
yarn add -D @types/react @types/react-dom
```

# （1）webpack.config.js

作用：webpack基本配置，定义入口、各种loader、plugin等。

路径：项目根目录/webpack.config.js

内容：

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
        filename: "index.js",
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
            inject: 'body'
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



# （2）.babelrc

作用：被@babel/core读取使用，其中定义了@babel/core要用到的preset、plugin等组件，对ts文件进行编译。想要深入理解，可以阅读另一篇文章：[【长文详解】TypeScript与Babel、webpack的关系以及IDE对TS的类型检查 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/561186916)。

路径：项目根目录/.babelrc

内容：

```json
{
  "presets": [
    "@babel/preset-env",
    "@babel/preset-typescript",
    ["@babel/preset-react", {"runtime": "automatic"}]
  ],
  "plugins": [
    "@babel/plugin-proposal-object-rest-spread",
    "@babel/plugin-proposal-class-properties"
  ]
}
```

# （3）tsconfig.json

作用：仅作为IDE进行TypeScript类型检查服务的文件，与ts代码编译没有直接关系。可以阅读另一篇文章来了解：[【长文详解】TypeScript与Babel、webpack的关系以及IDE对TS的类型检查 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/561186916)

路径：项目根目录/tsconfig.json

内容：

```json
{
  "compilerOptions": {
    "module": "commonjs",
    "rootDir": "./src",
    "outDir": "./dist",
    "jsx": "react-jsx"
  }
}
```

# （4）项目代码

src目录下，三个文件：index.tsx、index.module.less、react-app.d.ts。

## src/index.tsx

路径：项目根目录/src/index.tsx

内容：

```tsx
import {createRoot} from "react-dom/client";
import styles from './index.module.less';

const App = () => {
    return <div className={styles.app}>Hello, <span>App</span></div>
}

createRoot(document.querySelector('#app')).render(<App/>)
```

## src/index.module.less

路径：项目根目录/src/index.module.less

内容：

```less
html, body {
  height: 100%;
  width: 100%;
  margin: 0;
}

div {
  box-sizing: border-box;
}

.app {
  height: 100%;
  width: 100%;
  font-size: 20px;

  span {
    color: rgb(0, 111, 222);
    font-size: 24px;
  }
}
```

## src/react-app.d.ts（特别）

作用：*仅仅用于类型定义，目前定义的是模块化less文件的结构定义。*

路径：项目根目录/src/react-app.d.ts

内容：

```typescript
declare module '*.module.less' {
    const content: {
        [className: string]: any
    }
    export default content;
}
```

## public/index.html

作用：指定的html模板，根webpack.config.js里面HtmlWebpackPlugin所制定的模板路径保持一致。

路径：项目根目录/public/index.html

内容：

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<body>
<div id="app"></div>
</body>
</html>
```

# 运行

在package.json中添加运行脚本：

```diff
{
  "name": "webpack5-react-demo",
  "version": "1.0.0",
  "main": "index.js",
  "license": "MIT",
+ "scripts": {
+  "dev": "webpack-dev-server --config webpack.config.js"
+ },
  "devDependencies": {
		... ...
  },
  "dependencies": {
    ... ...
  }
}

```

效果：

![020-run](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-01-26/020-run.png)

# 附录

## 图解webpack配置与NPM包关系

![010-webpack-config-and-npm](https://src-1252109805.cos.ap-chengdu.myqcloud.com/images/post/2023-01-26/010-webpack-config-and-npm.png)

## github仓库

[w4ngzhen/webpack5-react-demo (github.com)](https://github.com/w4ngzhen/webpack5-react-demo)